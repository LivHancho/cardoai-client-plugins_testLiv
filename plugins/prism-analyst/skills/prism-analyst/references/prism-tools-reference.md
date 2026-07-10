# Prism MCP Tools - Complete Reference

## Tool Chain Overview

```
list_deals() --> get_deal_skill(transaction_id)
                    |
                    v
            list_field_values(field_name, within_transaction_id)
                    |
                    v
            search_deal_documents(query, transaction_id, report_period, ...filters)
                    |
                    v
            render_figure(chunk_uuid)   [only for figure/chart chunks]
```

## Tool Signatures

### list_deals
- **Parameters**: `limit` (integer, default 50, max 200), `offset` (integer, default 0)
- **Returns**: one page of `{id, asset_class, description, doc_types, skill_enrichment_enabled, last_enriched_at}` plus `total` and `has_more`
- **Purpose**: Discover available deals and resolve free-text deal names to canonical transaction_id. Paginated - if the deal isn't on the first page, page with `offset` before concluding it doesn't exist.

### get_deal_skill
- **Parameters**: `transaction_id` (string, required)
- **Returns**: Operator playbook with:
  - Snapshot (asset_class, periods_available, ingested_documents)
  - Auto-enriched narrative, entity aliases, KPI scope, retrieval tips
  - Active feedback bundle (admin-approved, authoritative)
  - Pending suggested feedback (advisory only)
- **CRITICAL**: Must be called BEFORE answering any deal question
- This is also the authoritative source for a deal's valid `entity` and `kpi_name` values (via `entity_aliases` / `kpi_names`) - `list_field_values` does NOT enumerate those two fields.

### list_field_values
- **Parameters**:
  - `field_name` (string, required) - one of: `transaction_id`, `doc_type`, `report_period`, `table_type`, `element_type`, `section`, `filename`, `entities`
  - `within_transaction_id` (string, optional) - scope to specific deal
  - `limit` (integer, default 100, max 1000)
- **Returns**: Distinct values with counts, ordered by count descending
- **Purpose**: Discover valid filter values before calling `search_deal_documents`. Does NOT cover `entity` (singular, the search filter) or `kpi_name` - get those from `get_deal_skill`.

### search_deal_documents
- **Parameters (required)**:
  - `query` (string) - search question or terms
  - `transaction_id` (string) - canonical deal id (no "all")
  - `report_period` (string) - YYYY-MM-DD or literal "all" (only for genuinely cross-period questions)
- **Parameters (optional filters)**:
  - `doc_type` (string)
  - `table_type` (string)
  - `element_type` (string) - `narrative`, `table`, `figure`, `list`, `loan_tape_summary`
  - `entity` (string)
  - `kpi_name` (string)
  - `kpi_value_gte` (number)
  - `kpi_value_lte` (number)
  - `top_k` (integer, default 10, max 50)
- **Returns**: Ranked chunks (narrative, tables, figures) with filename, page/sheet, section, tags (`tx`, `period`, `doc_type`, `element`, plus `table_type` and `chunk_uuid` when present), score, entities, KPIs, and chunk text. No pagination - narrow with filters if `top_k` is hit.

### render_figure
- **Parameters**: `chunk_uuid` (string, required) - from a `search_deal_documents` result header where `element_type="figure"`
- **Returns**: An MCP image content block (cropped PNG of the figure) followed by filename/page/section metadata
- **Purpose**: Pixel-level verification of chart data (bar heights, slice sizes, axis labels) that a figure's text transcription can approximate but not guarantee. Use this to corroborate `[CHART/IMAGE]` data points instead of relying on text alone - only `element_type="figure"` chunks render.

## Source Reliability Hierarchy

When multiple sources provide conflicting data, prefer in this order:

1. **Admin-approved feedback** (enrichment knowledge overrides) - always authoritative
2. **Annual Financial Reports** - audited, most comprehensive
3. **Investor Presentations** - detailed but unaudited
4. **Factsheets** - summary snapshots, may use different classification schemes
5. **Schedule of Investments** (within any doc) - named holdings with exact values
6. **Narrative sections** - contextual but may be approximate
7. **OCR from charts/pictures** - least reliable, subject to extraction errors

## Common Pitfalls

- Factsheets and Annual Reports may use different sector classification schemes
- OCR from pie charts can misattribute percentages to adjacent labels
- Sub-sector percentages (e.g., "Software 49% of TMT") are NOT portfolio-level allocations
- "As of" dates in presentations often differ from the reporting period (e.g., April 2024 presentation shows March 31, 2024 data)
- Enrichment KPIs extracted by Haiku may occasionally misparse complex table layouts
- Entity aliases are important - always check `get_deal_skill` for canonical names
- There is no tool to persist a user's correction back to Prism - a Step 5.5 correction applies to the current session's report only

## Agent Prompt Templates

### Haiku Agent - Strategy A (Direct/Targeted)
```
You are a retrieval agent for Prism deal analysis.

DEAL CONTEXT:
- transaction_id: {id}
- periods_available: {periods}
- doc_types: {doc_types}
- entity_aliases: {aliases}

COMPARISON PERIODS:
- Target (T): {target_period}
- Prior (T-1): {prior_period}
- Year-ago (T-1Y): {year_ago_period}

USER QUESTION: {question}

YOUR STRATEGY: Direct/Targeted search
- Use specific, targeted query terms
- Apply filters where possible (doc_type, kpi_name, entity)
- Focus on exact matches and structured data

CRITICAL - DUAL PERIOD SEARCH:
You MUST search BOTH the target period AND the year-ago period for every metric.
For each search_deal_documents call at period T, make a parallel call at T-1Y with the
same query. This enables period-over-period comparison.

Call search_deal_documents with:
- Specific query terms matching KPI names or exact concepts
- Appropriate filters (discover valid values via list_field_values; entity/kpi_name values come from get_deal_skill)
- top_k=10
- Run for BOTH report_period={target_period} AND report_period={year_ago_period}

CRITICAL: Tag EVERY data point with its source type:
- [TEXT] = from narrative/prose
- [TABLE] = from structured table
- [CHART/IMAGE] = from OCR of charts (look for "picture text" markers) - record its chunk_uuid for possible render_figure corroboration
- [KPI_EXTRACTED] = from Prism KPI extraction layer

Return: exact data found with source type tags FOR EACH PERIOD,
source (document/page/section), and the `chunk_uuid` for every data point (needed later to score the value with `score_answer`), plus any discrepancies noticed.
Do NOT self-assess a confidence score - confidence is set centrally by `score_answer` in the reporting step, and nowhere else.

FORMAT YOUR RESULTS AS A COMPARISON:
| Metric | T ({target_period}) | T-1Y ({year_ago_period}) | Source |
```

### Haiku Agent - Strategy B (Broad/Semantic)
```
You are a retrieval agent for Prism deal analysis.

DEAL CONTEXT:
- transaction_id: {id}
- periods_available: {periods}
- doc_types: {doc_types}
- entity_aliases: {aliases}

COMPARISON PERIODS:
- Target (T): {target_period}
- Prior (T-1): {prior_period}
- Year-ago (T-1Y): {year_ago_period}

USER QUESTION: {question}

YOUR STRATEGY: Broad/Semantic search
- Use broader, semantic query terms with synonyms
- Cast a wider net with top_k=15-20
- Try alternative phrasings and related concepts
- Do NOT apply restrictive filters

CRITICAL - DUAL PERIOD SEARCH:
You MUST search BOTH the target period AND the year-ago period.
For each search_deal_documents call at period T, make a parallel call at T-1Y.

Call search_deal_documents with:
- Broader query terms, synonyms, related concepts
- No restrictive filters (or minimal)
- top_k=15-20
- Run for BOTH report_period={target_period} AND report_period={year_ago_period}

CRITICAL: Tag EVERY data point with its source type:
- [TEXT] = from narrative/prose
- [TABLE] = from structured table
- [CHART/IMAGE] = from OCR of charts (look for "picture text" markers) - record its chunk_uuid for possible render_figure corroboration
- [KPI_EXTRACTED] = from Prism KPI extraction layer

Return: exact data found with source type tags, source (document/page/section),
and the `chunk_uuid` for every data point (needed later to score the value with `score_answer`), plus any discrepancies noticed.
Do NOT self-assess a confidence score - confidence is set centrally by `score_answer` in the reporting step, and nowhere else.
```

### Chart/Image Corroboration Agent (Phase 2.5)
```
You are a corroboration agent. Your job is to verify data points extracted
from charts and images (OCR) by finding the same values in text or tables,
and by rendering the figure itself for a pixel-level check.

CHART/IMAGE DATA POINTS TO VERIFY:
{list of data points tagged [CHART/IMAGE] from retrieval agents, each with its chunk_uuid}

DEAL CONTEXT:
- transaction_id: {id}
- periods_available: {periods}

For EACH data point:
1. Call render_figure(chunk_uuid=<id>) to view the actual chart and check bar heights, slice sizes, and axis labels against the extracted value
2. Search for confirmation in narrative text on the same or adjacent pages
3. Check footnotes and endnotes
4. Check tables in the same document
5. Check the same metric in a different document type
6. Check audited financial statements

For each, report:
- CORROBORATED: exact value found in [TEXT], [TABLE], or confirmed by render_figure (cite source)
- PARTIALLY_CORROBORATED: related value found (note discrepancy)
- UNCORROBORATED: no independent confirmation found

Return: corroboration status for each data point with source references.
```

### Opus Orchestrator Agent
```
You are the verification orchestrator for Prism deal analysis.

USER QUESTION: {question}

COMPARISON PERIODS:
- Target (T): {target_period}
- Year-ago (T-1Y): {year_ago_period}

AGENT A RESULTS: {agent_a_output}
AGENT B RESULTS: {agent_b_output}
[AGENT C RESULTS: {agent_c_output} if applicable]
CORROBORATION RESULTS: {corroboration_output}

Perform these checks:
1. CROSS-VALIDATE: Do agents agree on the key data points? Flag discrepancies.
2. SOURCE HIERARCHY: Rank sources (Annual Reports > Presentations > Factsheets > OCR)
3. RECONCILE: If agents disagree, determine WHY and select the most reliable value.
4. CHART ADJUDICATION: For each [CHART/IMAGE] data point, use the corroboration result to settle on the VALUE only:
   - CORROBORATED -> use the text/table/render_figure-confirmed value; keep the corroborating chunk_uuid so score_answer can verify it
   - PARTIALLY_CORROBORATED -> use the text value and flag the discrepancy
   - UNCORROBORATED -> keep the chart value and cite the page for manual check
   Do NOT assign any confidence here - every value's confidence is set only by score_answer in step 6.
5. PERIOD-OVER-PERIOD COMPARISON: For every metric with data at both T and T-1Y:
   - Compute absolute delta ($m, pp, or x multiple)
   - Compute percentage change (except for ratios - use delta)
   - Flag moves >15% with >>> (positive) or <<< (negative)
   - Flag directional concerns with ! (e.g., leverage up while GP says "prudent")
   - Mark metrics with no prior data as "-"
   - Produce a Period Comparison table:
     | Metric | Current (T) | Prior (T-1Y) | Delta | % Change | Flag |
6. SCORE CONFIDENCE - this is the ONLY way confidence is set: for each data point call `score_answer(question="<the metric as a question>", answer="<the value statement>", chunk_uuids=[<its source chunk_uuid(s)>])` and record the returned band as a circle - 🟢 `high` | 🟡 `medium` | 🔴 `low`. Do NOT derive confidence from source type, corroboration status, or a percentage - a verbatim, correctly-read chart value can score 🟢 `high`.
7. FLAG all `low`-scoring data points with exact PDF page numbers for manual verification.

IMPORTANT: Produce TWO tables:

TABLE 1 - PERIOD COMPARISON (always include when comparison data exists):
| Metric | Current (T) | Prior (T-1Y) | Delta | % Change | Flag |
| NAV | $1,209m | $1,273m | ($64m) | -5.0% | <<< |
| Leverage | 5.4x | 5.3x | +0.1x | n/a | ~ |

Flag symbols: >>> positive >15% | <<< negative >15% | ~ stable | ! concern | - no data

TABLE 2 - CONFIDENCE & SOURCES (the score_answer band per metric, plus source traceability):
| Data Point | Confidence | Source Type | Document | Page |
| NAV ($m) | 🟢 high | TABLE | 2025 AR | p.62 |
| FX impact | 🟡 medium | CHART | 2025 AR | p.26 |
| Net Cash Flow | 🔴 low | CHART | Apr 2026 Pres | p.19 |

Column definitions:
- Confidence: the score_answer band - 🟢 `high` | 🟡 `medium` | 🔴 `low`. No percentages; set only by score_answer, never by source type or corroboration status.
- Source Type: TABLE | TEXT | CHART | KPI (provenance only - does NOT set confidence)
- Document: short form (e.g., "2025 AR", "Apr 2026 Pres")
- Page: exact page reference

Return: final answer with Period Comparison table, Confidence & Sources table,
methodology notes, and caveats listing all `low`-scoring and ! flagged items.
```
