# Insider Trading Agent — Demo Overview

> A Claude Code agent, powered by Firecrawl, that watches SEC Form 4 insider-sale
> filings, filters for the signal that matters (a C-suite officer dumping >$1M of
> discretionary stock), researches the context, and DMs you a formatted report.
>
> Sponsored build for the Firecrawl video. Optimized as a **demo showpiece** —
> clear decision points, a satisfying payoff (a PDF landing in Slack).

---

## The one-liner

**"When a top executive sells over a million dollars of their own company's stock
on the open market, I want a researched brief in my Slack — automatically."**

Most insider sales are noise: mid-level VPs, tiny amounts, or pre-scheduled
(10b5-1) auto-sales. This agent throws all of that away and only surfaces the
rare, high-signal event.

---

## The pipeline at a glance

```
  Firecrawl Monitor                    Claude Code routine (webhook-triggered)
  ┌───────────────┐    webhook    ┌──────────────────────────────────────────┐
  │ watch          │  "changed"   │  1. Re-scrape table, diff → NEW rows      │
  │ secform4.com/  │─────────────▶│  2. GATE A (cheap, on row data):          │
  │ insider-sales  │              │        strict C-suite?  AND  value >$1M?  │
  └───────────────┘              │        └── no ─▶ EXIT (logged, on-camera)  │
                                  │  3. Firecrawl scrape/extract detail page  │
                                  │  4. GATE B: 10b5-1 planned sale?          │
                                  │        └── yes ─▶ EXIT (discretionary only)│
                                  │  5. Firecrawl /search → bull/bear research│
                                  │  6. Build PDF → Slack DM to Lucas          │
                                  └──────────────────────────────────────────┘
```

The whole thing is a **funnel**: each stage is cheaper than the next, so the
expensive work (detail scrape, web research, PDF) only runs for the ~1-in-hundreds
filing that actually matters.

---

## Stage-by-stage, with the handoff contracts

### Stage 0 — Source & trigger (Firecrawl Monitoring)
- **Firecrawl surface:** `create_monitor` on `https://www.secform4.com/insider-sales`
- Schedule: natural-language cadence (5-min minimum). We use a short interval so
  the demo feels live.
- Notification: **webhook**.
- **Design decision:** we treat the webhook as a *"the table changed"* trigger
  only. The docs don't publish the webhook body schema, so we do **not** depend on
  it carrying the new-row content. This keeps the handoff robust.

**Hands off →** an HTTP POST to our routine's webhook endpoint.

### Stage 1 — Orchestrator: find what's new
- Claude Code routine wakes on the webhook.
- Firecrawl-scrapes the insider-sales table fresh, diffs against the last-seen
  snapshot, and produces a list of **new rows**.
- Each new row already carries everything Gate A needs — no detail page yet.

**Handoff object (per new row):**
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

### Stage 2 — GATE A: the cheap filter (on row data)
Two checks, both on the row above — no API cost:
1. **Strict C-suite?** Title maps to CEO / CFO / COO / CTO / CIO / President.
   (Fuzzy match — titles arrive as `"Director, EVP and COO"`, `"EVP, Worldwide
   Field Ops"`, `"10% Owner"`, so the agent classifies rather than string-equals.)
2. **Total value > $1,000,000?**

Fail either → **early exit**, logged. *This is a deliberate on-camera beat:* show
a non-qualifying filing getting rejected so the filter is visibly working.

**Hands off →** `filing_url` for each survivor.

### Stage 3 — Detail extraction (Firecrawl scrape/extract)
- **Firecrawl surface:** `/scrape` with a JSON extraction schema against the
  individual filing page.
- Pulls the full transaction breakdown, footnotes, and — critically — whether the
  sale was made under a **Rule 10b5-1 trading plan** (this is *not* on the table).

**Handoff object adds:**
```json
{
  "is_10b5_1": false,
  "transactions": [ { "code": "S", "shares": ..., "price": ..., "date": ... } ],
  "footnotes": [ "..." ]
}
```

### Stage 4 — GATE B: exclude planned sales
- `is_10b5_1 == true` → **early exit.** We only want discretionary, open-market
  sales — the genuinely bearish signal.
- Survivors are now confirmed: **strict C-suite, >$1M, discretionary.**

### Stage 5 — Deep research (Firecrawl Search)
- **Firecrawl surface:** `/search` with `scrapeOptions` enabled so each result
  returns full-page markdown, not just a snippet. The Claude Code agent runs
  several targeted queries and synthesizes.
- Research targets: recent company news, price action / recent performance, other
  recent insider activity at the same company, and any guidance/litigation events.
- Output: a bull-vs-bear read on *why this sale might matter*, alongside the plain
  facts of what the executive did.

> **Open item:** research depth — quick 3–5 source scan vs. a fuller multi-angle
> sweep. Default assumed: a focused sweep (~6–10 sources across news + price +
> insider-cluster), capped for demo pacing.

### Stage 6 — Report & delivery
- Render a **formatted PDF**: exec + transaction summary up top, then the research
  brief with bull/bear framing.
- **Slack:** DM to Lucas (channel `D07D9AXBPTL`, DLM Holdings workspace) with the
  PDF attached and a one-line text summary of the findings.

---

## Decisions locked in

| Decision | Choice |
|---|---|
| C-suite scope | **Strict**: CEO, CFO, COO, CTO/CIO, President only |
| $1M threshold basis | **Total filing value** (sum of all sale lines) |
| 10b5-1 planned sales | **Exclude** (discretionary only) |
| Build target | **Demo showpiece** (happy-path clarity over hardening) |

## Demo-day notes (because strict + $1M + discretionary is rare)
- **Curate the trigger.** Hand-pick a known discretionary C-suite >$1M filing to
  run the live demo against, so the pipeline actually completes on camera.
- **Show one early-exit on purpose.** Run a non-qualifying filing through Gate A as
  the "filter is working" beat, then run the curated one for the payoff.
- All three sample rows currently on the page (NVIDIA/Puri-EVP, Workday/Duffield-10%
  owner, Halliburton/Shannon-COO-$771K) would each early-exit — good proof the
  filter is doing real work, bad as a live happy-path. Hence the curation.

## Still open
1. **Research depth** — confirm the 3–5 vs. fuller-sweep call above.
2. **Webhook plumbing** — confirm how the Claude Code routine is triggered (cloud
   routine endpoint) once we start building Stage 1.
3. **10b5-1 for the video** — keep hard-exclude, or soften to *flag-don't-exclude*
   just for the recording so the happy path always completes?
