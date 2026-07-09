# CLAUDE.md — Insider Trading Agent

Instructions for the Claude Code agent that runs this pipeline. This file is the
workflow contract; `OVERVIEW.md` is the narrative design doc and
`architecture.excalidraw` is the diagram. When they disagree, this file wins for
*how to run*, OVERVIEW.md wins for *why*.

## What this agent does

Watches **public SEC Form 4 insider-sale filings** (via secform4.com), surfaces
the rare high-signal event — a strict C-suite officer selling **> $1M** of stock
in a **discretionary** (non-10b5-1) open-market sale — researches the context,
and delivers a formatted PDF brief to Slack.

> Scope note: This works exclusively with **publicly disclosed** Form 4 filings
> that companies are legally required to file. It is a monitoring/summarization
> tool over public data — not a trading system and not an execution engine.

## The funnel (run stages in order; exit early and cheap)

Each stage is more expensive than the last. Only the ~1-in-hundreds filing that
clears every gate reaches the expensive work.

| # | Stage | Surface | Exit condition |
|---|-------|---------|----------------|
| 0 | Trigger | Firecrawl Monitor webhook on the insider-sales page | — |
| 1 | Diff for new rows | Firecrawl scrape + snapshot diff | no new rows → done |
| 2 | **Gate A** (cheap) | classify on row data only | not strict C-suite **or** value ≤ $1M → exit |
| 3 | Detail extract | Firecrawl `/scrape` + JSON schema on filing page | — |
| 4 | **Gate B** | check `is_10b5_1` | planned sale (`true`) → exit |
| 5 | Deep research | Firecrawl `/search` with `scrapeOptions` | — |
| 6 | Report & deliver | build PDF → Slack DM | — |

## Stage details

### Stage 0 — Trigger
- Firecrawl `create_monitor` on `https://www.secform4.com/insider-sales`,
  notification via **webhook**.
- Treat the webhook as a *"the table changed"* signal only. **Do not** depend on
  the webhook body carrying row content — its schema isn't published. Always
  re-scrape in Stage 1.

### Stage 1 — Find what's new
- Re-scrape the insider-sales table, diff against the last-seen snapshot, emit a
  list of **new rows**. Each row must carry the fields below (Gate A needs no
  detail page):
```json
{
  "insider_name": "Jeffrey Shannon",
  "title": "Director, EVP and COO",
  "company": "Halliburton",
  "ticker": "HAL",
  "transaction_date": "2026-07-08",
  "shares": 23895,
  "avg_price": 32.30,
  "total_value": 771808,
  "filing_url": "https://www.secform4.com/filings/45012/0001970357-26-000004.htm"
}
```

### Stage 2 — Gate A (cheap filter, no API cost)
1. **Strict C-suite?** Classify `title` → one of: CEO, CFO, COO, CTO, CIO,
   President. Titles arrive messy (`"Director, EVP and COO"`, `"10% Owner"`) — so
   **classify**, don't string-match.
2. **Total value > $1,000,000?** (basis = total filing value, sum of all sale lines)

Fail either → log and exit. Pass → hand off `filing_url`.

### Stage 3 — Detail extraction
- Firecrawl `/scrape` with a JSON extraction schema on the individual filing page.
- Must pull whether the sale was under a **Rule 10b5-1 plan** (not on the table),
  the per-line transactions, and footnotes.
```json
{ "is_10b5_1": false,
  "transactions": [ { "code": "S", "shares": 0, "price": 0.0, "date": "" } ],
  "footnotes": [] }
```

### Stage 4 — Gate B
- `is_10b5_1 == true` → exit. We only want **discretionary** open-market sales.
- Survivors are confirmed: **strict C-suite, > $1M, discretionary.**

### Stage 5 — Deep research
- Firecrawl `/search` with `scrapeOptions` enabled (full-page markdown, not
  snippets). Run several targeted queries and synthesize.
- Targets: recent company news, price action, other recent insider activity at
  the same company, guidance/litigation events.
- Output: a **bull-vs-bear** read on why the sale might matter, plus the plain
  facts of what the executive did.
- Depth: focused sweep, ~6–10 sources across news + price + insider-cluster,
  capped for pacing.

### Stage 6 — Report & delivery
- Render a **PDF**: exec + transaction summary on top, research brief with
  bull/bear framing below.
- Slack DM to Lucas (channel `D07D9AXBPTL`, DLM Holdings workspace) with the PDF
  attached and a one-line text summary.

## Locked decisions
- C-suite scope: **strict** (CEO, CFO, COO, CTO/CIO, President only)
- $1M basis: **total filing value**
- 10b5-1: **exclude** (discretionary only)
- Build target: **demo showpiece** — happy-path clarity over hardening

## Conventions for the agent
- **Fail cheap.** Never run a later stage's API calls before the earlier gate
  passes. The whole point is the funnel.
- **Log every exit** with the row and the reason — the on-camera "filter is
  working" beat depends on visible rejections.
- **Never fabricate filing data.** Every number in the report must trace to a
  scraped source. If a field is missing, say so — don't infer it.
- Secrets (Firecrawl key, Slack token) come from env / `.env`, never hard-coded
  and never committed.

## Still open
1. Research depth — confirm 3–5 quick vs. the ~6–10 focused sweep.
2. Webhook plumbing — how the cloud routine endpoint is triggered.
3. 10b5-1 for the video — hard-exclude vs. flag-don't-exclude for the recording.
