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
| 3 | `loss-vintage` | `../../abf-asset-class-context/knowledge/_analyses/loss-vintage.md` |
| 4 | `prepayment` | `../../abf-asset-class-context/knowledge/_analyses/prepayment.md` |
| 5 | `concentration` | `../../abf-asset-class-context/knowledge/_analyses/concentration.md` |

**Why Blocks 1–2 are inline and Blocks 3–5 are knowledge files.** Blocks 3–5 have asset-class applicability gates (`loss-vintage` requires `cf_determinism: YES`; `prepayment` requires `loan_sim_fit: YES|PARTIAL`; `concentration` bifurcates consumer vs. non-consumer). Each methodology lives alongside the asset-class catalog in `abf-asset-class-context/knowledge/_analyses/` — same authority that determines applicability. Blocks 1–2 use universal portfolio shape (balance, count, WA metrics, DQ stratification) with no asset-class gate, so the spec stays inline.

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

The block's domain is in scope. For blocks 3-5, the source methodology file has been read once during this run (do not re-read if already loaded this conversation).

## Actions

For the current block:

1. **Load the block's source.** Blocks 1–2 follow the inline specs above. Blocks 3-5 read the methodology file named in the block table (Read tool, once per run; reuse the in-memory copy after).
2. **Retrieve.** Call `get_stratification_analytics_data` for the analytics the block needs.
3. **Narrate.** Author the block's analytical narrative per the source's `## Analyze` and `## Format` sections.
4. **Visual** (when visuals are on). Produce the block's default visual per `../../abf-gateway/knowledge/reference/visuals-playbook.md`.
5. **Deliver to the analyst.** Send the narrative + visual + the source `url` for each analytics fetched in this block (mandatory — see Context window management above).
6. **Checkpoint prompt.** "Next up: [Section Title]. Continue, skip, or jump to Key Findings?" — section titles follow the sequence: Portfolio Overview → Credit Performance → Loss Vintage Analysis → Prepayment Vintage Analysis → Geographic & FICO Concentration → Key Findings & Risk Assessment.
7. **Handle the analyst's reply.** Continue → next block. Skip section → note the skip, proceed to the block after. Jump to findings → Synthesize. Stop → Handoff.

## Exit conditions

The block narrative + source links have been delivered to the analyst and the checkpoint prompt answered.

## Failure modes

- **`get_stratification_analytics_data` fails.** Surface the error to the analyst; do NOT silently skip the block. Ask for direction.
- **Source methodology hits its applicability gate.** A block's methodology file may refuse to run when its asset-class precondition fails (e.g., `loss-vintage.md` when `cf_determinism: NO`). Phase Discover's profile-flag suppression should prevent this case, but if it slips through: skip the block, surface a one-line note to the analyst ("Skipping [Section Title] — [catalog reason]"), and continue to the next block. Do NOT prompt the analyst for direction; the catalog answer is final.
- **Analyst chose "stop" mid-delivery.** Route to Handoff.

## What the runner must NOT do in this phase

- **No re-fetching of analytics data already retrieved this run.** Reuse the figures already in the conversation; re-running the same `get_stratification_analytics_data` is waste.
- **No silent block skips.** Every skipped block is called out to the analyst.

## Cross-references

- Source methodology files per the block table above
- `../../abf-gateway/knowledge/reference/visuals-playbook.md` — default visual per block
