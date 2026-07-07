# Derived Metric Formulas

Reference for computing derived ABF metrics. Always show derived metrics alongside their raw inputs for verification.

## Prepayment speeds

- **Monthly SMM** = `1 - (1 - CumPrepay)^(1/Term)`
- **Annualized CPR** = `1 - (1 - SMM)^12`

## Period-over-period

- **MoM Change** = `(Current - Prior) / Prior`

## Composition

- **Concentration Drift** = `Share_{t+n} - Share_t` (expressed in basis points)

## Credit quality

- **Delinquency Rate** = `DQ Balance / Total Outstanding Balance`
- **Default Share vs Portfolio Share** = `Segment Default % / Segment Portfolio %`
  - >1.0 → over-representation (segment contributes more defaults than its size predicts)
  - <1.0 → under-representation

## Numerical formatting

- Balances >1M: round to nearest thousand (12.4M)
- Balances <1M: round to nearest unit (845,230)
- Counts: exact integers
- Rates/percentages: one decimal (3.2%)
- Basis points: integer (+99 bps)
- Convert decimals to percentages BEFORE rounding (0.0325 → 3.25%, not 0.0%)
- Currency: use the transaction's native currency if known; otherwise omit

## Labeling rules

- Observed values: state as-is.
- Projections: label as "projected based on current rates" with explicit assumptions.
- Never present derived metrics without the raw inputs they came from.
