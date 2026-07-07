# Interactive Visuals Playbook

Reference for choosing what interactive chart, table, diagram, or timeline to produce. Consult when producing a visual and the current MCP client supports inline artifacts. Text-only clients fall back to compact markdown tables — keep the analytical structure the same.

## When to produce a visual

Produce a visual whenever the data is visualization-friendly:

| Analysis pattern | Default visual |
|---|---|
| Trend over time (DQ %, balance, CPR by month) | Multi-series line chart |
| Segment comparison (state, FICO, cohort) | Grouped or stacked bar chart |
| Distribution / concentration | Bar chart or donut chart |
| Vintage × term matrix (loss curves, prepayment curves) | Heatmap |
| DPD bucket stratification | Stacked bar or horizontal bar |
| Composition drift over time | Stacked area chart |
| Wide tabular output (>10 rows or >5 columns) | Interactive sortable table |
| Process / pipeline (cash flow, waterfall stages) | Flowchart or sankey |
| Portfolio overview | Summary card grid with headline figures |

Don't produce a visual for:
- Single-value status answers ("balance is 12.4M") — text is faster.
- Responses where the user already has the number they asked for.

## Trigger phrasing

When you offer or produce a visual, use clear language that tells the client what to render:
- "Let me chart this data"
- "I'll draw this as a diagram"
- "Here's how this changes over time"
- "Let me render this as an interactive table"

Then include the structured data the renderer needs — typed headers, sorted rows, explicit axes. Do not produce a visual "in the abstract"; always provide the data shape alongside the offer.

## How to offer (mid-session, per analysis)

If the user enabled visuals at session start but the current response is borderline (e.g., 3-4 numbers), ask a compact one-liner:

> "Want me to chart this?"

If the user confirmed visuals are on and the analysis is visualization-friendly (per the table above), skip the ask and produce it directly.

## How to structure the visual content

For each visual, emit:
1. The textual analysis first (headline + key figures).
2. The visual-trigger sentence ("Let me chart this — [chart description].").
3. The raw data (table or structured list) so the renderer has material.

Example, for a DQ trend response:

> Headline: "30+ DPD rose from 1.4% to 3.8% over the last 12 months."
>
> Key figures: [3 bullets with specific month-to-month deltas]
>
> "Let me chart this — monthly DQ % broken out by bucket."
>
> | Month | 30-60 | 60-90 | 90+ |
> | 2025-03 | 0.8% | 0.4% | 0.2% |
> | 2025-04 | 0.9% | 0.5% | 0.2% |
> | ... | ... | ... | ... |

The client's renderer picks up the prompt + table and produces the interactive line chart.

## Visual type defaults per ABF analysis

Use these as the first-choice visual for each domain skill. The canonical default for each skill lives in the skill's own **Default visual** section; this table is a quick-reference copy — keep both in sync when updating either.

- Loss vintage (`_analyses/loss-vintage.md`) → **heatmap** of cohort (row) × term-on-book (column), values = cumulative default rate. Call out the elbow cell explicitly.
- Prepayment (`_analyses/prepayment.md`) → **grouped bar chart** (FICO bucket × SMM / CPR side-by-side) + **stacked area chart** for projected composition drift.
- Concentration (`_analyses/concentration.md`) → **horizontal bar chart** for top-N states (or top-N obligors for non-consumer). **Donut** for FICO distribution — consumer asset classes only; skip for commercial, CRE, or energy.
- `abf-report-runner` focused answer — trend queries → line chart. Segmentation queries → bar chart. Bucket queries → stacked bar.
- `abf-report-runner` comprehensive blocks — one signature visual per block (see per-skill default).
