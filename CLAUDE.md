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

### Step 4 — Extract role + transaction amount for each new filing

Each added row is a markdown table row with these columns (pipe-delimited):

| # | Column | awk field (`-F'\|'`) |
|---|--------|----------------------|
| c1 | Transaction Date + type | `$2` |
| c2 | Reported DateTime | `$3` |
| c3 | Company | `$4` |
| c4 | Symbol | `$5` |
| c5 | **Insider Relationship** (name + role) | `$6` |
| c6 | Shares Traded | `$7` |
| c7 | Average Price | `$8` |
| c8 | **Total Amount** ($ value of the sale) | `$9` |
| c9 | Shares Owned | `$10` |
| c10 | Filing (`View` link) | `$11` |

The two fields the gates need — the **role** (c5) and the **Total Amount** (c8) — are in the
row for essentially every filing, so no extra fetch is normally required. **Firecrawl is the
fallback** when a row is missing or ambiguous (empty/garbled role, or a `$0`/blank Total
Amount): scrape the filing's `View` page (c10) or the insider's page and read the reporting
person's title and the transaction total from there —

```bash
# Fallback context fetch (only when the row itself is insufficient):
curl -s -X POST https://api.firecrawl.dev/v2/scrape -H "$AUTH" -H "Content-Type: application/json" \
  -d "{\"url\":\"$VIEW_LINK\",\"formats\":[\"markdown\"],\"onlyMainContent\":true}" | jq -r '.data.markdown'
```

### Step 5 — Qualifying gates: C-suite **AND** Total Amount > $1,000,000

Keep a filing **only if both** hold. **Interpretation: this is an AND** — "only interested in
sales > $1M" is a hard floor, so a $1M+ sale by a non-exec is dropped, and a C-suite sale
under $1M is dropped. (If you ever want OR instead, change the `&&` in the gate below to `||`.)

**C-suite** = the reporting person holds a chief-level executive title: any `Chief … Officer`
(CEO, CFO, CTO, COO, CIO, CMO, CLO, CAO, CHRO, CISO, CRO, CCO, …) or **President**. Excluded:
Director, 10% Owner, plain General Counsel, Secretary/Treasurer, and VP/EVP/SVP that isn't a
chief title. Roles are messy free text and rows can list several parties — **use judgment**,
and pull the filing via Firecrawl (Step 4) when a title is genuinely unclear. Passing the role
gate needs **any one** listed party to be C-suite.

```bash
MIN_USD=1000000
: > /tmp/qualified.jsonl

is_csuite() {  # role text -> exit 0 if C-suite
  local r; r=$(printf '%s' "$1" | tr 'A-Z' 'a-z')
  printf '%s' "$r" | grep -qE 'chief[a-z &.,/()-]*officer|\b(ceo|cfo|cto|coo|cio|cmo|clo|cao|chro|ciso|cro|cco|cdo|cgo)\b' && return 0
  printf '%s' "$r" | grep -qE '\bpresident\b' && ! printf '%s' "$r" | grep -qE 'vice[ -]president' && return 0
  return 1
}

printf '%s\n' "$ADDED_ROWS" | while IFS= read -r row; do
  [ -z "$row" ] && continue
  line=$(printf '%s' "$row" | sed 's/^+//')
  relationship=$(printf '%s' "$line" | awk -F'|' '{print $6}')
  company=$(printf '%s' "$line" | awk -F'|' '{print $4}' | sed -E 's/\[([^]]*)\].*/\1/; s/^ *//; s/ *$//')
  symbol=$(printf '%s' "$line"  | awk -F'|' '{print $5}' | sed -E 's/\[([^]]*)\].*/\1/; s/^ *//; s/ *$//')
  total_cell=$(printf '%s' "$line" | awk -F'|' '{print $9}')
  VIEW_LINK=$(printf '%s' "$line" | awk -F'|' '{print $11}' | grep -oE 'https?://[^) ]+' | head -1)

  # Total Amount -> integer USD (handles "$19,246,616", "$0", blanks)
  amount=$(printf '%s' "$total_cell" | grep -oE '[0-9][0-9,]*' | head -1 | tr -d ',')
  # Role text = relationship cell minus the [Name](link) parts (what's left is the title(s))
  role_text=$(printf '%s' "$relationship" | sed -E 's/\[[^]]*\]\([^)]*\)//g; s/<br>/ ; /g; s/^[ ;]*//; s/[ ;]*$//')

  # If role or amount is missing/ambiguous, resolve via Firecrawl (Step 4) before deciding.

  role_ok=no;  is_csuite "$role_text" && role_ok=yes
  amt_ok=no;   [ -n "$amount" ] && [ "$amount" -gt "$MIN_USD" ] && amt_ok=yes

  echo "• $symbol ($company) | role='$role_text' csuite=$role_ok | amount=\$${amount:-?} over1M=$amt_ok"
  if [ "$role_ok" = yes ] && [ "$amt_ok" = yes ]; then      # <-- AND gate (see note above)
    jq -nc --arg s "$symbol" --arg c "$company" --arg r "$role_text" \
           --argjson a "${amount:-0}" --arg v "$VIEW_LINK" \
      '{symbol:$s, company:$c, role:$r, amount_usd:$a, filing_url:$v}' >> /tmp/qualified.jsonl
  fi
done

if [ ! -s /tmp/qualified.jsonl ]; then
  echo "No new filing is BOTH C-suite AND > \$1,000,000. Exiting — stopping the routine."
  exit 0
fi
echo "QUALIFYING FILINGS:"; cat /tmp/qualified.jsonl
```

### Step 6 — Record the qualifying detection(s)

Append the high-signal hits to the detection log (audit trail) and commit.

```bash
mkdir -p detections
TS=$(date -u +%Y-%m-%dT%H:%M:%SZ)
while IFS= read -r q; do
  printf '%s\n' "$q" | jq -c --arg ts "$TS" --arg check "$CHECK_ID" '. + {detected_at:$ts, check_id:$check}' \
    >> detections/insider-sales.jsonl
done < /tmp/qualified.jsonl

git add detections/insider-sales.jsonl && \
  git commit -m "Qualifying insider sale(s) — C-suite > \$1M — check $CHECK_ID" || \
  echo "(nothing to commit or git unavailable — continuing)"
```

### Step 7 — Act on the qualifying filing(s)  ⟵ EXTENSION POINT (business action still TBD)

Only filings in `/tmp/qualified.jsonl` reach here — each is a C-suite sale over $1M. The final
action isn't defined yet (do **not** invent one). Candidates: post a summary to Slack (the
Slack MCP connector is attached to this routine — supply a channel to enable), open the linked
Form 4 for deeper analysis, or hand off to a downstream system. Until defined, stop after Step 6.

## Prototype caveats

- **Data is ~6 months delayed.** We monitor the public (unauthenticated) page, which
  `secform4.com` delays ~6 months for non-subscribers. New rows are real Form 4 sales but
  old ones, and the delayed feed likely advances ~once per trading day. Real-time detection
  needs an authenticated Insider Pro session — deferred.
- **You fire hourly regardless of change.** The Step 2/3 gates are what keep the ~23/24
  no-op fires cheap. Never skip them.
- A single-page monitor's first-ever check is `status: "new"` (baseline). Real new filings
  arrive as `status: "changed"` — which is why the gate keys on `summary.changed`, not `new`.
