# ABF Vocabulary

Business terms used in ABF analyst conversations and on the server-side skill definitions. Use these in user-facing text; refer to every object by its display name and keep internal platform UUIDs (transaction / analytics / column IDs) out of prose entirely — link with the frontend `url`, not the raw ID. Business-data identifiers from the dataset (Borrower ID, ISIN, Facility ID) are legitimate analytical content and exempt.

## Core entities

- **Transaction** — a securitization deal. Sometimes called "deal" or "securitization" by analysts. The unit of analytical scope; a stratification analytics is always scoped to one or more transactions.
- **Originator** — the institution that originated the underlying loans / receivables.
- **Asset class** — the type of underlying asset (e.g. financial credit, real estate, vehicle, commercial credit). Closed list — see `securitizations://asset-classes` resource.
- **Tranche** — a securitization layer with a defined seniority and cash-flow priority. Surfaced by waterfall tools.

## Analytics objects

- **Stratification Analytics** (analysts may still call it "strat view" or "view" out of habit) — a saved analytical slice on a Dynamic Analytics. Defines aggregations, group-bys, filters, and chart types. Returns a Count column plus the metric columns.
- **Dynamic Analytics** — the entity-relational substrate underneath stratification analytics. Defines the row grain, joined entities, and the column projection (Standard / Custom / Calculated / Aggregation).
- **Aggregation** — a metric computed over rows (sum / avg / count / percentile / etc.). Has a `name`, `aggregation_function`, and optionally `filters`.
- **Group-by** — a dimension columns rows are grouped along. Date dimensions use a `date_cluster_type`; numeric dimensions can use bucket clusters.
- **Filter** — a predicate applied to rows. Three stages: `pre_aggregation` (Simple / Custom columns, applied to source rows), `post_aggregation` (Aggregation columns, removes groups by computed value), `post_projection` (Calculated columns, applied to derived expressions).

## Time and reporting

- **Reporting date** — the as-of date the analytics snapshot represents. Stratification Analytics data is always returned AS OF the transaction's most recent asset synchronization date.
- **Vintage / cohort** — origination month/quarter/year of the underlying loans, used as a group-by dimension in vintage analyses.
- **Term on book / seasoning** — months elapsed since origination, the dimension paired with vintage in cumulative-loss/prepayment matrices.

## Performance metrics

- **Outstanding balance** — current principal balance on the underlying assets.
- **DPD (Days Past Due)** / **DID (Days In Delay)** — days a loan is delinquent. Bucketed in stratifications (0-30, 30-60, 60-90, 90+).
- **Default** — terminal credit event. The exact detection rule varies per integration (servicer-flag / DPD threshold / charge-off date).
- **Write-off / charge-off** — accounting recognition of a default.
- **Repurchase** — originator buys back a loan, often for an early-payment-default or rep-and-warranty breach.
- **Prepayment** — early voluntary repayment of principal, partial or full.
- **SMM (Single Monthly Mortality)** — monthly prepayment rate, derived as `1 - (1 - CumPrepay)^(1/Term)`.
- **CPR (Conditional Prepayment Rate)** — annualized prepayment rate, derived as `1 - (1 - SMM)^12`.

## Credit segmentation

- **FICO bucket** — credit-score band. Common ABF cuts: prime (>740), near-prime (670–740), sub-prime (<670).
- **Negative selection / adverse selection** — when prepayment-driven attrition leaves the pool with a worse credit mix than originally underwritten.
- **Composition drift** — change in segment shares (FICO, state, originator) over time, often driven by differential prepayment speeds.

## Data conventions

- **Percentage scale** — `data_type: percentage` is stored as 0..1 (e.g. 0.05 = 5%). `data_type: percentage_full` is stored as 0..100. Cluster labels render with `%`; CSV rows carry the stored scale. Filter values use the stored scale.
- **Amount conventions** — raw currency units (e.g. 1500000.0). Cluster labels abbreviate (`1.5M`); CSV rows carry raw values.
- **Glossary block** — every `get_stratification_analytics_data` response includes a `Glossary:` block in the CSV mapping column display names to business meaning. **Trust the glossary over column names when they disagree.**

## Cross-references

- Filter operators per data type: read resource `stratification://filter-operators`
- Filter stage semantics: read resource `stratification://filter-stages`
- Data type interpretation (percentage scale, amount conventions): read resource `stratification://data-types`
- Aggregation operators: read resource `stratification://aggregation-operators`
- Asset class values: read resource `securitizations://asset-classes`
