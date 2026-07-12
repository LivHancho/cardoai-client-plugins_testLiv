# Block Delivery (template)

The runner walks blocks 1–5 in order, skipping any whose domain the analyst put out of scope. This file is the template applied per block; the per-block specifics are looked up at runtime from the block table below.

## Context window management

Do NOT reproduce full data tables. For every data point:
- Summarize 3-5 most significant figures inline
- Highlight outliers and inflection points
- **Append the source link — MANDATORY.** Every block that fetches data ends with the source `url` from each `get_stratification_analytics_data` response it drew on, so the analyst can open the full dataset on the platform. The `url` is embedded in every response; never deliver a block's figures without its source link.

Small summary tables (≤5 rows × 5 columns) are allowed when essential to the narrative. Larger tables → link to the platform instead.

## Block table

| Block index | `block_name` | Source |
|---|---|---|
| 1 | `portfolio-overview` | Inline below |
| 2 | `credit-performance` | Inline below |
| 3 | `loss-vintage` | `catalog.on_demand.loss_vintage` via `read_resource` |
| 4 | `prepayment` | `catalog.on_demand.prepayment` via `read_resource` |
| 5 | `concentration` | `catalog.on_demand.concentration` via `read_resource` |

**Why Blocks 1–2 are inline and Blocks 3–5 are served sections.** Blocks 3–5 have asset-class applicability gates (`loss-vintage` requires `cf_determinism: YES`; `prepayment` requires `loan_sim_fit: YES|PARTIAL`; `concentration` bifurcates consumer vs. non-consumer). Each methodology is served by the ABF MCP server alongside the asset-class index — the same authority that determines applicability — and the index pre-computes the gates: a gated-off block's `on_demand` key arrives as `null`. Blocks 1–2 use universal portfolio shape (balance, count, WA metrics, DQ stratification) with no asset-class gate, so the spec stays inline.

## Block 1 — Executive Summary & Portfolio Overview

**Retrieve:** outstanding balance, loan count, WA metrics views.

**Analyze:** growth trajectory (peak/current), ramp-up and amortization rates, WA characteristics drift.

**Format:** `[Transaction Name] Portfolio Performance & Analytics Report` header with originator and reporting date. 1-paragraph executive summary + 5-7 bullet-point highlights. Portfolio composition (compact table if needed). Growth-trajectory narrative with 3-4 balance snapshots.

**Default visual** (when visuals are on): summary card grid with headline figures.

## Block 2 — Credit Performance Analysis

**Retrieve:** DQ stratification, DQ trend views.

**Analyze:** DQ as % of outstanding balance per bucket. Trend direction (stable / rising / accelerating / volatile). Flag >15% MoM growth. Note negative selection (lower FICO, higher APR in delinquent loans).

**Format:** Delinquency stratification narrative (current %, largest bucket, 90+ balance) + trend narrative (direction per bucket, 90+ trajectory with 2-3 data points).

**Default visual** (when visuals are on): multi-series line chart of DQ % per bucket over time.

## Purpose

Deliver one report block to the analyst.

## Entry conditions

The block's domain is in scope. For blocks 3-5, the block's methodology section has been read once during this run (do not re-read if already loaded this conversation).

## Actions

For the current block:

1. **Load the block's source.** Blocks 1–2 follow the inline specs above. Blocks 3-5 read the methodology section named in the block table (`read_resource` on the catalog's `on_demand` key, once per run; reuse the in-memory copy after).
2. **Retrieve.** Call `get_stratification_analytics_data` for the analytics the block needs, with an explicit conservative `limit` (start ~50-100 rows — the block only summarizes a handful of figures; see `../../abf-gateway/knowledge/reference/common-pitfalls.md` on oversized results).
3. **Narrate.** Author the block's analytical narrative per the source's `## Analyze` and `## Format` sections. For Blocks 3-4 (loss-vintage, prepayment) and any other multi-dimension response, follow `../../abf-gateway/knowledge/contracts/reading-stratification-tables.md` first — these are exactly the cohort/MOB × date tables where cell-vs-cohort and cumulative-vs-periodic misreads happen.
4. **Visual** (when visuals are on).
   a. Before producing it, list every row/cohort/vintage/category dropped, capped, deduplicated, re-bucketed, or re-aggregated versus what Step 2's retrieval returned raw.
   b. If that list is non-empty and the analyst has not already specified this exact treatment (in this run or as a standing session preference), **stop — do not produce the visual or the narrative figures built from the filtered set.** Ask the analyst how to handle each item (include as-is / flag distinctly / exclude) as the entire response for this block; resume the block on their reply.
   c. Otherwise produce the block's default visual per `../../abf-gateway/knowledge/reference/visuals-playbook.md`. Any genuine reporting gap in a time series renders as a true break (`connectNulls: false` / `spanGaps: false`), never interpolated.
5. **Deliver to the analyst.** Send the narrative + visual + the source `url` for each analytics fetched in this block (mandatory — see Context window management above).
6. **Checkpoint prompt.** "Next up: [Section Title]. Continue, skip, or jump to Key Findings?" — section titles follow the sequence: Portfolio Overview → Credit Performance → Loss Vintage Analysis → Prepayment Vintage Analysis → Geographic & FICO Concentration → Key Findings & Risk Assessment.
7. **Handle the analyst's reply.** Continue → next block. Skip section → note the skip, proceed to the block after. Jump to findings → Synthesize. Stop → Handoff.

## Exit conditions

The block narrative + source links have been delivered to the analyst and the checkpoint prompt answered.

## Failure modes

- **`get_stratification_analytics_data` fails.** Surface the error to the analyst; do NOT silently skip the block. Ask for direction.
- **Block's `on_demand` key is `null`.** The index pre-computes applicability gates — a `null` key means the methodology does not apply to this asset class (e.g., loss-vintage when `cf_determinism: NO`). Phase Discover's profile-flag suppression should prevent this case, but if it slips through: skip the block, surface a one-line note to the analyst ("Skipping [Section Title] — [catalog reason]"), and continue to the next block. Do NOT prompt the analyst for direction; the catalog answer is final.
- **Analyst chose "stop" mid-delivery.** Route to Handoff.

## What the runner must NOT do in this phase

- **No re-fetching of analytics data already retrieved this run.** Reuse the figures already in the conversation; re-running the same `get_stratification_analytics_data` is waste.
- **No silent block skips.** Every skipped block is called out to the analyst.

## Cross-references

- Methodology sections per the block table above (`catalog.on_demand` keys via `read_resource`)
- `../../abf-gateway/knowledge/reference/visuals-playbook.md` — default visual per block
