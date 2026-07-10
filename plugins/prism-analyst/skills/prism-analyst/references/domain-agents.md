# Domain Agent Specifications

## Architecture Overview

```
                    ┌──────────────────────────┐
                    │     WORKING STATE        │
                    │  (shared ledger file)     │
                    └──┬───┬───┬───┬───┬──────┘
                       │   │   │   │   │
  Wave 1 (parallel):   │   │   │   │   │
    Performance Agent ──┘   │   │   │   │
    Cash Flow Agent ────────┘   │   │   │
    Composition Agent ──────────┘   │   │
                                    │   │
  Wave 2 (informed by Wave 1):     │   │
    Risk Agent ─────────────────────┘   │
                                        │
  Wave 3 (reads complete state):        │
    Reporting Agent ────────────────────┘
```

Each domain agent is an Opus-level orchestrator that:
1. Reads the Working State for cross-domain context
2. Spawns 2-3 Haiku sub-agents in parallel with different strategies
3. Receives sub-agent results, cross-validates, selects best value per metric
4. Writes verified findings to its section of the Working State
5. Logs sub-agent decision rationale (which sub-agent was chosen and why)
6. Flags gaps and red flags for other domains

---

## Domain Agent 1: Performance Agent

**Focus:** Returns, value drivers, operating metrics
**Wave:** 1 (runs in parallel, no dependencies)
**Model:** opus

### Metrics Owned
| Metric ID | Metric | Type | Comparison |
|-----------|--------|------|------------|
| PERF-01 | NAV ($m) | Point-in-time | T vs T-1Y |
| PERF-02 | NAV per share ($) | Point-in-time | T vs T-1Y |
| PERF-03 | NAV Total Return (%) | Flow | T vs T-1Y |
| PERF-04 | Private company appreciation (% ex-FX) | Flow | T vs T-1Y |
| PERF-05 | Quoted holdings change (% ex-FX) | Flow | T vs T-1Y |
| PERF-06 | LTM Revenue Growth (%) | Point-in-time | T vs T-1Y |
| PERF-07 | LTM EBITDA Growth (%) | Point-in-time | T vs T-1Y |
| PERF-08 | Top 5 positive contributors ($m) | Flow | T vs T-1Y |
| PERF-09 | Top 5 negative contributors ($m) | Flow | T vs T-1Y |
| PERF-10 | EV/EBITDA multiple (x) | Point-in-time | T vs T-1Y |
| PERF-11 | Top 10 companies % of portfolio | Point-in-time | T vs T-1Y |

### Sub-Agent Strategies
**Sub-A (Targeted):** Search for NAV, total return, per-share metrics. Query: "NAV total return per share net asset value". Use KPI filters where possible. Search both T and T-1Y.

**Sub-B (Value Drivers):** Search for top/bottom contributors, value change waterfall, performance drivers. Query: "key performance drivers value change positive negative contributors top 5 top 10". Search T period.

**Sub-C (Operating Metrics):** Search for revenue growth, EBITDA growth, valuation multiples. Query: "LTM revenue EBITDA growth weighted average EV/EBITDA multiple". Search both T and T-1Y.

### Cross-Validation Rules
- NAV must reconcile: NAV per share x shares outstanding ~ NAV total
- NAV TR should be consistent with (ending NAV + dividends) / beginning NAV - 1
- Top positive + top negative + other = total value change (approximately)
- If operating growth is positive but NAV declined, flag for Risk agent (likely multiple contraction or FX)

### Red Flag Triggers
- NAV TR < 0%
- EBITDA growth < revenue growth (margin compression)
- EV/EBITDA multiple changed > 1.0x
- Top 5 negative > 50% of top 5 positive

---

## Domain Agent 2: Cash Flow Agent

**Focus:** Realisations, investments, distributions, liquidity
**Wave:** 1 (runs in parallel, no dependencies)
**Model:** opus

### Metrics Owned
| Metric ID | Metric | Type | Comparison |
|-----------|--------|------|------------|
| CF-01 | Total realisations ($m) | Flow | T vs T-1Y |
| CF-02 | Realisations as % of opening portfolio | Flow | T vs T-1Y |
| CF-03 | New investments deployed ($m) | Flow | T vs T-1Y |
| CF-04 | Follow-on investments ($m) | Flow | T vs T-1Y |
| CF-05 | Dividends paid ($m) | Flow | T vs T-1Y |
| CF-06 | Dividends per share ($) | Flow | T vs T-1Y |
| CF-07 | Share buybacks ($m) | Flow | T vs T-1Y |
| CF-08 | Total capital returned ($m) | Flow | T vs T-1Y |
| CF-09 | Full exits (names + proceeds) | Flow | n/a |
| CF-10 | Partial exits (names) | Flow | n/a |
| CF-11 | Realisation MOIC (x) | Flow | T vs T-1Y |

### Sub-Agent Strategies
**Sub-A (Realisations):** Search for exits, realisations, proceeds, liquidity events. Query: "realisations exits proceeds cash received full partial sale". Search T and T-1Y.

**Sub-B (Deployments):** Search for new investments, follow-ons, commitments. Query: "new investments deployed follow-on commitments capital calls". Search T and T-1Y.

**Sub-C (Returns to Shareholders):** Search for dividends, buybacks, capital return. Query: "dividends share buyback capital returned shareholders repurchase". Search T and T-1Y.

### Cross-Validation Rules
- Total capital returned ~ dividends + buybacks
- Realisations should be >= capital returned (funding source)
- Named exits should reconcile with Schedule of Investments changes

### Red Flag Triggers
- Realisations < dividends + buybacks (funding gap)
- New investments > realisations (net cash drain)
- Realisation MOIC < 1.0x (exits below cost)

---

## Domain Agent 3: Composition Agent

**Focus:** Portfolio construction, holdings, sector/geography/vintage allocation
**Wave:** 1 (runs in parallel, no dependencies)
**Model:** opus

### Metrics Owned
| Metric ID | Metric | Type | Comparison |
|-----------|--------|------|------------|
| COMP-01 | Total portfolio value ($m) | Point-in-time | T vs T-1Y |
| COMP-02 | Number of holdings | Point-in-time | T vs T-1Y |
| COMP-03 | Top 10 holdings (names + values) | Point-in-time | T vs T-1Y |
| COMP-04 | Sector allocation (%) | Point-in-time | T vs T-1Y |
| COMP-05 | Geography allocation (%) | Point-in-time | T vs T-1Y |
| COMP-06 | Vintage year distribution (%) | Point-in-time | T vs T-1Y |
| COMP-07 | % in private vs public | Point-in-time | T vs T-1Y |
| COMP-08 | Average holding period (years) | Point-in-time | T vs T-1Y |
| COMP-09 | Top 30 concentration (% of FV) | Point-in-time | T vs T-1Y |
| COMP-10 | New entries since T-1Y | n/a | Diff |
| COMP-11 | Exits since T-1Y | n/a | Diff |

### Sub-Agent Strategies
**Sub-A (Top Holdings):** Search for Schedule of Investments, top 10/20/30 tables. Query: "schedule investments top companies holdings fair value". Search T and T-1Y.

**Sub-B (Allocation):** Search for sector, geography, vintage breakdowns. Query: "sector allocation geography vintage distribution portfolio diversification". Search T and T-1Y.

**Sub-C (Changes):** Search for new investments, exits, portfolio changes. Query: "new investments exits realisations portfolio changes during year". Search T.

### Cross-Validation Rules
- Sum of sector allocations should = ~100%
- Top holdings values should sum to < total portfolio value
- Holdings count should reconcile with exits/entries

### Red Flag Triggers
- Single holding > 10% of NAV (concentration)
- Top 5 holdings > 30% of NAV
- Any sector > 25% (concentration)
- Average holding period > 6 years

---

## Domain Agent 4: Risk Agent

**Focus:** Leverage, FX, debt, covenants, stress factors
**Wave:** 2 (runs AFTER Wave 1, reads their findings)
**Model:** opus

### Metrics Owned
| Metric ID | Metric | Type | Comparison |
|-----------|--------|------|------------|
| RISK-01 | Net Debt/EBITDA (x) | Point-in-time | T vs T-1Y |
| RISK-02 | Interest coverage ratio (x) | Point-in-time | T vs T-1Y |
| RISK-03 | % cov-lite debt | Point-in-time | T vs T-1Y |
| RISK-04 | FX impact ($m) | Flow | T vs T-1Y |
| RISK-05 | Debt maturity profile | Point-in-time | n/a |
| RISK-06 | Near-term maturities ($m or %) | Point-in-time | T vs T-1Y |
| RISK-07 | Share price discount to NAV (%) | Point-in-time | T vs T-1Y |
| RISK-08 | Unrealised losses ($m) | Flow | T vs T-1Y |
| RISK-09 | Credit facility utilisation | Point-in-time | T vs T-1Y |
| RISK-10 | LTV ratio | Point-in-time | T vs T-1Y |

### Cross-Domain Inputs (from Working State)
- From Performance: If NAV declined, investigate leverage and FX as causes
- From Performance: If EBITDA grew but NAV flat, investigate multiple contraction
- From Cash Flow: If realisations < distributions, check credit facility draw
- From Composition: If concentration increased, flag single-name risk

### Sub-Agent Strategies
**Sub-A (Leverage & Debt):** Search for leverage, debt, covenants, maturity. Query: "net debt EBITDA leverage covenant debt maturity interest coverage". Search T and T-1Y.

**Sub-B (FX & Market):** Search for FX impact, currency exposure, public holdings. Query: "foreign exchange FX impact currency USD EUR GBP quoted public holdings discount NAV". Search T and T-1Y.

**Sub-C (Stress & Targeted):** Based on Wave 1 findings - search for specific risks flagged by other agents. Dynamically constructed query.

### Red Flag Triggers
- Leverage > 6.0x
- Interest coverage < 2.0x
- Near-term maturities > 20% of total debt
- FX impact > 5% of NAV
- Discount to NAV > 30%
- Unrealised losses > unrealised gains

---

## Domain Agent 5: Reporting Agent

**Focus:** Synthesize complete state into final report
**Wave:** 3 (runs AFTER all others, reads complete state)
**Model:** opus

### Responsibilities
1. Read the complete Working State
2. Verify completeness - identify any remaining PENDING metrics
3. Spawn gap-filler sub-agents (Haiku) for missing data
4. Run chart/image corroboration for any [CHART/IMAGE] data
5. Compute all period-over-period deltas and flags
6. Score every metric's confidence by calling `score_answer(question, value, chunk_uuids)`; record the band as a circle (🟢 high / 🟡 medium / 🔴 low)
7. Detect narrative vs. numbers inconsistencies
8. Generate the final structured response

### Sub-Agent Strategies
**Sub-A (Gap Filler):** Search for specific metrics still marked PENDING in the state. Dynamically constructed queries.

**Sub-B (Chart Corroboration):** For each [CHART/IMAGE] data point, search for text/table confirmation.

**Sub-C (Narrative Audit):** Search for GP narrative statements and compare against the verified numbers. Flag inconsistencies.

### Output Format
The Reporting Agent produces the final answer using the standard format:
- Period Comparison table (with flags)
- Confidence Assessment table (with sources)
- Methodology
- Caveats (all `low` and ! items)

---

## Sub-Agent Decision Protocol

Every domain agent must follow this protocol when receiving sub-agent results:

1. **Collect** all sub-agent outputs for the same metric
2. **Compare** values - do they agree?
3. **If agree:** Use the value from the most authoritative source (TABLE > TEXT > KPI > CHART)
4. **If disagree:** Apply these tiebreakers in order:
   a. Audited financial statements always win
   b. TABLE extraction beats TEXT beats CHART/IMAGE
   c. Annual Report beats Presentation beats Factsheet
   d. If still tied, use the value confirmed by more sub-agents
5. **Log the decision** in the Working State:
   ```
   PERF-01 NAV: Sub-A=$1,209m (TABLE, AR p.62) | Sub-C=$1,200m (TEXT, AR p.12)
   SELECTED: Sub-A ($1,209m) - TABLE source from financial statements, more precise
   ```
6. **Flag unresolved conflicts** in the state - an unresolved value has no cleanly-grounded source, so it will score `low`
