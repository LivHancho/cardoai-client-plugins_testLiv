---
name: prism-analyst
description: This skill should be used when the user asks about a Prism deal or transaction, mentions a deal by name (e.g. "NBPE", "westlake", "volkswagen", "ftai"), asks about portfolio data, sector allocation, top companies, KPIs, NAV, financial metrics, or any question that requires querying the Prism MCP tools. It provides multi-agent verified retrieval with domain-specialized orchestrators, a shared working state, confidence scoring, and source references.
version: 2.1.0
---

# Prism Analyst v3 - Domain-Orchestrated Multi-Agent Retrieval

This skill uses a hierarchical multi-agent architecture where domain-specialized orchestrator agents (Opus) each spawn and manage their own sub-agents (Haiku), exchange data through a shared Working State file, and produce verified, confidence-scored analysis.

## Architecture

```
                    ┌────────────────────────┐
                    │     WORKING STATE      │
                    │   (shared .md file)     │
                    └──┬───┬───┬───┬───┬────┘
                       │   │   │   │   │
  Wave 1 (parallel):   │   │   │   │   │
    Performance Agent ──┘   │   │   │   │   (each spawns 2-3 Haiku sub-agents)
    Cash Flow Agent ────────┘   │   │   │
    Composition Agent ──────────┘   │   │
                                    │   │
  Wave 2 (reads Wave 1 findings):  │   │
    Risk Agent ─────────────────────┘   │
                                        │
  Wave 3 (reads complete state):        │
    Reporting Agent ────────────────────┘
```

Each domain agent is an Opus orchestrator that:
1. Reads the Working State for cross-domain context
2. Spawns 2-3 Haiku sub-agents in parallel with different search strategies
3. Cross-validates sub-agent results, selects the best value per metric
4. Logs its decision rationale (which sub-agent was chosen and why)
5. Writes verified findings to its section of the Working State
6. Flags gaps and red flags for other domain agents

## Core Protocol

### Step 0 - Clarification Gate

Before any retrieval, evaluate the question for ambiguity:

- **Deal**: Can the deal be resolved to a canonical transaction_id? If not, call `list_deals` and ask.
- **Period**: Is a specific reporting period clear? If ambiguous, ask.
- **Metric**: Is the requested data point specific enough? If vague, ask.
- **Scope**: Single value, trend, or comparison? If unclear, ask.

If ANY dimension is ambiguous, ask a concise clarifying question. Do NOT guess.

### Step 1 - Load Deal Context (MANDATORY)

1. `mcp__prism__list_deals()` - resolve deal name to transaction_id. Results are paginated (default `limit=50`); if the deal isn't in the first page, page through with `offset` before asking the user to disambiguate.
2. `mcp__prism__get_deal_skill(transaction_id=<id>)` - load operator playbook
3. `mcp__prism__list_field_values(field_name="doc_type", within_transaction_id=<id>)` - discover document types. `field_name` only accepts `transaction_id, doc_type, report_period, table_type, element_type, section, filename, entities` - there is no `entity` or `kpi_name` option here. For entity aliases and KPI names, use `get_deal_skill`'s `entity_aliases` / `kpi_names` fields instead.

Extract: periods_available, doc_types, entity aliases, active feedback bundle, enrichment overrides.

### Step 1.5 - Period Resolution

Determine comparison periods from `periods_available`:
- **T** (target): The period the user is asking about
- **T-1** (prior): Immediately preceding period in the list
- **T-1Y** (year-ago): Same calendar position ~12 months earlier
- If user asks about a full year, T = Dec year-end, T-1Y = prior Dec year-end

### Step 2 - Create Working State File

Create the Working State file at:
`~/.claude/Projects/Prism/.prism-workspace/session-{deal}-{period}-{timestamp}.md`

Use the template from `references/working-state-template.md`. Fill in the Deal Context section with data from Steps 1 and 1.5.

### Step 3 - Wave 1: Parallel Domain Agents

Spawn THREE domain agents in parallel using the Agent tool, each with `model: "opus"`:

**Performance Agent** - NAV, returns, value drivers, operating metrics
**Cash Flow Agent** - Realisations, investments, distributions, liquidity
**Composition Agent** - Holdings, sector/geography allocation, portfolio changes

Each domain agent's prompt must include:
- The deal context (transaction_id, periods, doc_types, entity aliases)
- The comparison periods (T, T-1, T-1Y)
- The user's question
- Its domain specification (from `references/domain-agents.md`)
- Instructions to spawn 2-3 Haiku sub-agents internally, cross-validate, and return structured findings

Each domain agent must return:
- Verified metrics table (metric, T value, T-1Y value, source, confidence)
- Sub-agent decision log (which sub-agent was selected per metric and why)
- Gaps (metrics it couldn't find)
- Red flags (anomalies detected)
- Narrative fragments (GP quotes for later audit)
- Chart/image data points (tagged for corroboration, with `chunk_uuid` when the source is a figure - needed for `render_figure`)

### Step 3.5 - Merge Wave 1 & Detect Cross-Domain Signals

After all Wave 1 agents return:

1. **Read** all three domain agent outputs
2. **Write** their findings to the Working State file (each to its own section)
3. **Detect cross-domain signals:**
   - If Performance shows NAV declined but EBITDA grew -> flag "multiple contraction" for Risk
   - If Cash Flow shows realisations < distributions -> flag "funding gap" for Risk
   - If Composition shows concentration increased -> flag for Risk
   - If Performance shows large FX impact -> flag for Risk with specific amount
4. **Write** cross-domain signals to the Risk Agent's "Cross-Domain Inputs" section

### Step 4 - Wave 2: Risk Agent

Spawn ONE Risk Agent with `model: "opus"`. Its prompt must include:
- Everything from Wave 1 (via the Working State or direct output)
- The cross-domain signals identified in Step 3.5
- Instructions to investigate these specific signals with targeted sub-agents
- Its domain specification

The Risk Agent:
1. Reads Wave 1 findings and cross-domain signals
2. Spawns 2-3 Haiku sub-agents, including at least one targeted at the flagged signals
3. Cross-validates, writes verified risk metrics to the state
4. Flags any new red flags discovered

### Step 4.5 - Update Working State

Write Risk Agent findings to the Working State. Compile the complete Cross-Domain Red Flags table.

### Step 5 - Wave 3: Reporting Agent

Spawn ONE Reporting Agent with `model: "opus"`. Its prompt must include:
- The complete Working State (all domain findings)
- All chart/image data points needing corroboration
- All narrative fragments for audit
- Instructions to:
  1. Fill any remaining PENDING metrics (spawn gap-filler Haiku sub-agents)
  2. Run chart/image corroboration (spawn corroboration Haiku sub-agents): for each flagged `[CHART/IMAGE]` data point, call `mcp__prism__render_figure(chunk_uuid=<id>)` on its `chunk_uuid` (from the originating `search_deal_documents` result header) to check the actual chart pixels, in addition to text/table corroboration
  3. Audit GP narrative against verified numbers
  4. Compute all period-over-period deltas and flags
  5. Apply confidence scoring (GREEN/AMBER/RED) to every metric
  6. Produce the final structured response

### Step 5.5 - User Validation Gate (MANDATORY when any RED data exists)

Before generating any final report, the parent MUST check if any data point scored 🔴 RED (< 50%) or has an `!` directional concern flag. If so, present ALL low-confidence items to the user for confirmation in this EXACT format:

```
## Data Validation Required

The following data points have low confidence and need your confirmation
before I generate the final report.

For each item: confirm (Y), correct the value, or skip (S).

| # | Metric | Extracted Value | Confidence | Source | Issue |
|---|--------|----------------|------------|--------|-------|
| 1 | FX impact | +$34m | 🔴 40% | `[CHART/IMAGE]` 2025 AR, p.26 | OCR from waterfall chart, sign uncertain |
| 2 | Total value change | -$10m | 🔴 45% | `[CHART/IMAGE]` 2025 AR, p.26 | Does not reconcile to 5% NAV TR without adjustments |
| 3 | Net Cash Flow | ($117m) | 🔴 35% | `[CHART/IMAGE]` Apr 2026 Pres, p.19 | Bar label attribution uncertain |

**For each row, reply with:**
- `1: Y` to confirm the value is correct
- `1: +$34m` to confirm with the same value
- `1: -$34m` to correct the value
- `1: S` to skip (exclude from report)

Example response: `1: Y, 2: -$15m, 3: S`
```

**Rules for this gate:**
- NEVER generate the final report until the user responds to this validation
- NEVER skip this step when RED items exist - it is mandatory
- Include ALL data points scoring < 50% AND all `!` flagged items
- Group items logically (e.g., all waterfall chart items together)
- Show enough context (source, page, issue) for the user to make a decision
- If the user corrects a value, use the corrected value in the final report. There is no tool to persist the correction back to Prism for future queries - the correction applies to this session's report only.
- If the user confirms, upgrade the confidence to 🟡 65% (user-validated chart data)
- If the user skips, exclude the data point from the report and note it as "excluded by user"
- After user responds, proceed to Step 6

### Step 6 - Structured Response

The Reporting Agent produces, and the parent presents (incorporating any user corrections from Step 5.5):

**Period Comparison** table (always included when comparison data exists):

| Metric | Current (T) | Prior (T-1Y) | Delta | % Change | Flag |
|--------|------------|-------------|-------|----------|------|

Flag symbols: `>>>` significant positive (>15%) | `<<<` significant negative (>15%) | `~` stable | `!` directional concern (narrative vs. numbers) | `-` no prior data

**Answer** - clear, direct response with analysis and interpretation

**Confidence & Sources** - table with separate columns for confidence scoring and source traceability:

| Data Point | Confidence | Source Type | Document | Page | Corroboration | Rationale |
|------------|------------|-------------|----------|------|---------------|-----------|
| NAV ($m) | 🟢 95% | `TABLE` | 2025 Annual Report | p.62 | n/a | Audited balance sheet |
| FX impact | 🟡 60% | `CHART` | 2025 Annual Report | p.26 | CORROBORATED | Chart confirmed in narrative |
| Net Cash Flow | 🔴 35% | `CHART` | Apr 2026 Pres | p.19 | UNCORROBORATED | Bar label uncertain |

Column definitions:
- **Confidence**: 🟢 Reliable (70%+) | 🟡 Caution (50-69%) | 🔴 Verify manually (< 50%)
- **Source Type**: `TABLE` | `TEXT` | `CHART` | `KPI`
- **Document**: Document name (use short form: "2025 AR", "Apr 2026 Pres", "Dec 2024 FS")
- **Page**: Exact page for traceability
- **Corroboration**: n/a | CORROBORATED | PARTIALLY | UNCORROBORATED

**Overall Confidence: XX%**

**Methodology** - which domain agents contributed, how sub-agent conflicts were resolved, chart corroboration results

**Caveats** - all 🔴 RED and `!` flagged items with page references

## Scoping Rules

Not every question requires all five domain agents. The parent should scope based on the query:

| Query Type | Agents to Spawn |
|------------|----------------|
| Specific metric (e.g., "what is the NAV?") | Relevant domain agent only (e.g., Performance) |
| Company deep-dive (e.g., "Solenis performance") | Performance + Composition (Wave 1), then Risk (Wave 2) |
| Full review / scan | All five agents (full protocol) |
| Risk question | Performance (Wave 1 for context) + Risk (Wave 2) |
| Comparison question | Relevant domain(s) with emphasis on T-1Y retrieval |

When scoping down, still create the Working State file - just leave unused domain sections empty.

## Sub-Agent Decision Protocol

Every domain agent follows this when its sub-agents return conflicting data:

1. **Collect** all sub-agent values for the same metric
2. **Compare** - do they agree?
3. **If agree:** Use value from most authoritative source (TABLE > TEXT > KPI > CHART)
4. **If disagree:** Apply tiebreakers:
   a. Audited financial statements always win
   b. TABLE > TEXT > KPI_EXTRACTED > CHART/IMAGE
   c. Annual Report > Presentation > Factsheet
   d. If still tied, use value confirmed by more sub-agents
5. **Log** the decision with rationale
6. **Flag** unresolved conflicts as RED

## Chart/Image Data Rules

- NEVER assign GREEN to any `[CHART/IMAGE]` data point
- Chart data starts at AMBER (if corroborated by text) or RED (if not)
- Detection markers: `picture intentionally omitted`, `Start/End of picture text`
- Corroboration is handled by the Reporting Agent in Wave 3

## General Rules

- NEVER skip the Working State file - it's the backbone of cross-agent communication
- NEVER contradict admin-approved feedback from the deal skill
- NEVER fabricate data - if not found, mark as PENDING/gap
- NEVER generate the final report when RED data points exist without running Step 5.5 (User Validation Gate) first
- ALWAYS include period comparison when comparison data exists
- Use hyphens with spaces instead of em-dashes (user preference)

## References

- `references/domain-agents.md` - Full domain agent specifications, metrics, sub-agent strategies, red flag triggers
- `references/working-state-template.md` - Working State file template and lifecycle
- `references/prism-tools-reference.md` - Prism MCP tool signatures and parameters
