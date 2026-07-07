# Working State Template

This file is the template for the shared working state file created at the start of each analysis session. The actual state file is written to:
`~/.claude/Projects/Prism/.prism-workspace/session-{deal}-{period}-{timestamp}.md`

---

## Template

```markdown
# PRISM Working State
## Session: {deal_id} | T: {target_period} | T-1Y: {year_ago_period}
## Created: {timestamp}
## Query: {user_question}

---

## Deal Context
- transaction_id: {deal_id}
- asset_class: {asset_class}
- T (target): {target_period}
- T-1 (prior): {prior_period}
- T-1Y (year-ago): {year_ago_period}
- doc_types: {doc_types}
- entity_aliases: {aliases}
- enrichment_overrides: {overrides}

---

## Performance Domain
**Agent:** Performance | **Wave:** 1 | **Status:** PENDING

### Verified Metrics
| ID | Metric | T Value | T-1Y Value | Delta | % Chg | Source | Confidence | Flag |
|----|--------|---------|------------|-------|-------|--------|------------|------|

### Sub-Agent Decisions
| Metric | Sub-A | Sub-B | Sub-C | Selected | Reason |
|--------|-------|-------|-------|----------|--------|

### Gaps
- [ ] (auto-populated after agent runs)

### Red Flags
- (auto-populated)

### Figure References
- (chunk_uuid of any [CHART/IMAGE] data points, for render_figure corroboration)

---

## Cash Flow Domain
**Agent:** Cash Flow | **Wave:** 1 | **Status:** PENDING

### Verified Metrics
| ID | Metric | T Value | T-1Y Value | Delta | % Chg | Source | Confidence | Flag |
|----|--------|---------|------------|-------|-------|--------|------------|------|

### Sub-Agent Decisions
| Metric | Sub-A | Sub-B | Sub-C | Selected | Reason |
|--------|-------|-------|-------|----------|--------|

### Gaps
-

### Red Flags
-

### Figure References
-

---

## Composition Domain
**Agent:** Composition | **Wave:** 1 | **Status:** PENDING

### Verified Metrics
| ID | Metric | T Value | T-1Y Value | Delta | % Chg | Source | Confidence | Flag |
|----|--------|---------|------------|-------|-------|--------|------------|------|

### Sub-Agent Decisions
| Metric | Sub-A | Sub-B | Sub-C | Selected | Reason |
|--------|-------|-------|-------|----------|--------|

### Gaps
-

### Red Flags
-

### Figure References
-

---

## Risk Domain
**Agent:** Risk | **Wave:** 2 | **Status:** PENDING

### Cross-Domain Inputs (from Wave 1)
- Performance: (populated after Wave 1)
- Cash Flow: (populated after Wave 1)
- Composition: (populated after Wave 1)

### Verified Metrics
| ID | Metric | T Value | T-1Y Value | Delta | % Chg | Source | Confidence | Flag |
|----|--------|---------|------------|-------|-------|--------|------------|------|

### Sub-Agent Decisions
| Metric | Sub-A | Sub-B | Sub-C | Selected | Reason |
|--------|-------|-------|-------|----------|--------|

### Gaps
-

### Red Flags
-

### Figure References
-

---

## Cross-Domain Red Flags
(Populated by parent after merging Wave 1 + Wave 2 results)

| # | Flag | Source Domain | Severity | Detail |
|---|------|-------------|----------|--------|

---

## Narrative Fragments
(GP quotes collected by any agent for the Reporting Agent to audit)

| Quote | Source | Page | Context |
|-------|--------|------|---------|

---

## Chart/Image Data Points
(All [CHART/IMAGE] tagged data, pending corroboration)

| Metric | Value | Source | Corroboration Status |
|--------|-------|--------|---------------------|

---

## Report Draft
**Agent:** Reporting | **Wave:** 3 | **Status:** PENDING

(Reporting Agent fills this section with the final synthesized output)
```

---

## State File Lifecycle

1. **CREATE**: Parent creates the state file from this template at session start
2. **POPULATE CONTEXT**: Parent fills in Deal Context from Phase 1
3. **WAVE 1**: Performance, Cash Flow, Composition agents run in parallel
   - Each agent reads the state (Deal Context section)
   - Each agent writes to its own domain section
   - Parent merges results, identifies cross-domain red flags
4. **WAVE 2**: Risk agent runs
   - Reads all Wave 1 findings from the state
   - Uses cross-domain inputs to guide targeted searches
   - Writes to Risk section
5. **WAVE 3**: Reporting agent runs
   - Reads complete state
   - Fills gaps, corroborates charts, audits narrative
   - Writes final Report Draft section
6. **OUTPUT**: Parent reads Report Draft, presents to user
7. **PERSIST**: State file remains in .prism-workspace for future reference

## State File Naming Convention
`session-{deal_id}-{target_period}-{YYYYMMDD-HHMMSS}.md`

Example: `session-NBPE-2025-12-31-20260508-143022.md`
