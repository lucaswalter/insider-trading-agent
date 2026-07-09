# CLAUDE.md — Insider Trading Agent

Instructions for the Claude Code routine that runs this pipeline. **This file is the
workflow contract** — the routine's saved prompt is *"Please execute the workflow in
the claude.md file,"* so everything below is what you (the routine) do on each fire.

## What triggers you

A **Firecrawl page monitor** watches <https://www.secform4.com/insider-sales> once an
hour and, after each check completes, POSTs to this routine's fire endpoint. **Firecrawl
fires you on every hourly check — changed or not.** It does **not** pass any input, so
you must decide for yourself whether a genuinely new insider-sale filing appeared. Most
fires (roughly 23 of every 24) will find nothing new: **exit fast in that case.** Do real
work only when there is a new row.

Dedup is free: Firecrawl only labels the page `changed` when its content actually differs
from the previous snapshot, so consecutive `same` checks never re-flag a row you already
processed. You do **not** need to keep your own last-seen state.

## Constants

- `MONITOR_ID` = `019f48e3-702a-733c-9b8d-8f42fd77da9e`
- Firecrawl API base = `https://api.firecrawl.dev`
- Detection log = `detections/insider-sales.jsonl` (in this repo)

## Prerequisite

`FIRECRAWL_API_KEY` must be set in this routine's environment (add it as a secret in the
routine's settings on claude.ai). If it is missing, **exit gracefully** — do not error out.

## Workflow

Run these steps in order. `jq` is used for JSON parsing.

### Step 0 — Preflight

```bash
if [ -z "$FIRECRAWL_API_KEY" ]; then
  echo "FIRECRAWL_API_KEY not set in routine environment — cannot query the monitor. Exiting."
  exit 0
fi
AUTH="Authorization: Bearer $FIRECRAWL_API_KEY"
MONITOR_ID=019f48e3-702a-733c-9b8d-8f42fd77da9e
BASE=https://api.firecrawl.dev
```

### Step 1 — Get the check that just completed

Fetch recent completed checks and pick the newest by `finishedAt`:

```bash
CHECKS=$(curl -s "$BASE/v2/monitor/$MONITOR_ID/checks?status=completed&limit=5" -H "$AUTH")
CHECK_ID=$(echo "$CHECKS" | jq -r '.data | sort_by(.finishedAt) | last | .id')
CHANGED=$(echo "$CHECKS" | jq -r '.data | sort_by(.finishedAt) | last | .summary.changed')
echo "Latest completed check: $CHECK_ID (changed pages: $CHANGED)"
```

### Step 2 — Gate on "did anything change at all?"

```bash
if [ "$CHANGED" = "0" ] || [ -z "$CHANGED" ] || [ "$CHANGED" = "null" ]; then
  echo "No changed pages in the latest check — no new filings. Exiting."
  exit 0
fi
```

### Step 3 — Confirm it's a *meaningful new row*, not page churn

Pull the changed page(s) and inspect the AI judgment. The monitor's goal instructs the
judge to flag **only** when a new insider-sale row is added, so require
`judgment.meaningful == true`. Treat it as a real detection when the judgment is meaningful
**and** the diff adds one or more table rows.

```bash
DETAIL=$(curl -s "$BASE/v2/monitor/$MONITOR_ID/checks/$CHECK_ID?status=changed" -H "$AUTH")

MEANINGFUL=$(echo "$DETAIL" | jq -r '[.data.pages[]? | select(.judgment.meaningful == true)] | length')
# Added markdown table rows in the diff = new filings. Table rows contain pipe delimiters.
ADDED_ROWS=$(echo "$DETAIL" | jq -r '
  .data.pages[]?.diff.text // empty' | grep -E '^\+.*\|.*\|' | grep -vE '^\+\s*\|\s*-+' || true)

if [ "$MEANINGFUL" = "0" ] || [ -z "$ADDED_ROWS" ]; then
  echo "Change was not a meaningful new insider-sale row (page churn). Exiting."
  exit 0
fi

echo "NEW FILING(S) DETECTED:"
echo "$ADDED_ROWS"
```

### Step 4 — Record the detection

Append each new filing to the detection log (audit trail) and commit it. Capture the
judge's reasons for context.

```bash
mkdir -p detections
TS=$(date -u +%Y-%m-%dT%H:%M:%SZ)
echo "$DETAIL" | jq -c --arg ts "$TS" --arg check "$CHECK_ID" '
  .data.pages[]? | select(.judgment.meaningful == true) |
  {detected_at: $ts, check_id: $check, url: .url,
   reason: .judgment.reason,
   changes: [.judgment.meaningfulChanges[]? | select(.type == "added")],
   diff: .diff.text}' >> detections/insider-sales.jsonl

git add detections/insider-sales.jsonl && \
  git commit -m "Detect new insider-sale filing(s) — check $CHECK_ID" || \
  echo "(nothing to commit or git unavailable — continuing)"
```

### Step 5 — Act on the new filing  ⟵ EXTENSION POINT (define the business action)

This is where the pipeline's actual behavior goes once decided. Options on the table
(none wired yet — do **not** invent one):

- Post a summary of the new filing(s) to Slack (the Slack MCP connector is attached to
  this routine — supply the target channel to enable).
- Parse each added row into structured fields (ticker, insider, role, shares, $ amount,
  ownership) and run analysis.
- Open the linked Form 4 filing (`View` link in the row) for detail.

Until this is defined, stop after Step 4. The detection is safely logged.

## Prototype caveats

- **Data is ~6 months delayed.** We monitor the public (unauthenticated) page, which
  `secform4.com` delays ~6 months for non-subscribers. New rows are real Form 4 sales but
  old ones, and the delayed feed likely advances ~once per trading day. Real-time detection
  needs an authenticated Insider Pro session — deferred.
- **You fire hourly regardless of change.** The Step 2/3 gates are what keep the ~23/24
  no-op fires cheap. Never skip them.
- A single-page monitor's first-ever check is `status: "new"` (baseline). Real new filings
  arrive as `status: "changed"` — which is why the gate keys on `summary.changed`, not `new`.
