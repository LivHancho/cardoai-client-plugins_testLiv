# Common Pitfalls

Defensive patterns the gateway and report runner should follow proactively. Each pitfall lists the failure mode, the rule, and where the rule is enforced.

## Ambiguous view/filter choice presented as fact

**Failure:** More than one stratification view (or more than one saved filter set) could plausibly answer the analyst's plain-English question — e.g., "principal balance" could mean total UPB, a delinquency-bucket breakdown, or a filtered "Position" view excluding liquidated loans — and the agent silently picks one and delivers it as *the* answer instead of *an* answer.

**Rule:** When `list_transaction_analytics` or `get_filterable_columns` surfaces more than one reasonable candidate for the analyst's term, name the candidates and ask which one before running `get_stratification_analytics_data` — do NOT guess and self-correct after delivery. This does not apply to mechanical parameters (`limit`, pagination) or to cases where exactly one analytics/filter combination matches the semantics — only to genuine multi-candidate ambiguity.

**Enforcement:** `abf-gateway` Gateway Step 2 narrow fast path, before the `get_stratification_analytics_data` call.

## Narrating harness work

**Failure:** The agent tells the analyst about internal steps such as skill reads, catalog loads, or MCP call sequencing. This makes responses feel like thinking traces instead of useful ABF answers.

**Rule:** Follow `knowledge/contracts/response-discipline.md`. Harness work is silent by default. Speak only for required choices, blockers, and analysis/report delivery.

**Enforcement:** `abf-gateway` Response discipline section; `abf-report-runner` Response discipline section.

## Cross-transaction analytics ID reuse

**Failure:** Reusing a `strat_view_id` from one transaction when querying another. Per the plugin's CLAUDE.md Critical Constraint, `strat_view_id` is unique per transaction — the same analytics conceptually exists on each transaction with a different UUID.

**Rule:** Cross-transaction analysis discovers analytics separately for each transaction via `list_transaction_analytics(transaction_id=...)`.

**Enforcement:** CLAUDE.md Critical Constraint.

## Filtered vs. unfiltered semantic confusion

**Failure:** Picking the unfiltered "Outstanding Balance" analytics when the analyst asks about delinquent balance. The filter is part of what the analytics represents — "outstanding balance" and "outstanding balance where DID > 0" answer different questions.

**Rule:** When the user's intent implies a filter (delinquent / 90+ DPD / defaulted / a specific bucket), prefer analytics that already encode it. If no such analytics exists, apply a filter to the closest match via `get_filterable_columns` first.

**Enforcement:** `agents/view-selector.md` Tool sequence step 2 (match by filter semantics before falling through); the gateway must follow it.

## Inventing filter operators or enum values

**Failure:** Guessing at `equals` / `==` / `BETWEEN` filter operator names, or making up delinquency-bucket / status / category labels ("0 to 30", "low / medium / high").

**Rule:** Filter operators, allowed enum / bucket / status values, and asset class values must come from a discovery tool or reference resource — verbatim. The closed lists are:

- `get_filterable_columns` — allowed filter operators and enum/choice values per column for a given analytics
- `stratification://filter-operators` — filter operators per data type
- `securitizations://asset-classes` — asset class values

**Enforcement:** Server-side validation rejects unrecognized values with `{"error": "validation", ...}`. The gateway should not reach that boundary — load `get_filterable_columns` first.

## Trusting column name over glossary

**Failure:** Interpreting a column by its display name when the response's `Glossary:` block says it means something else.

**Rule:** When the column name and glossary description disagree, the glossary wins. The glossary captures business semantics that column names can't express.

**Enforcement:** Every `get_stratification_analytics_data` response carries the glossary; readers consult it before reasoning numerically.

## Re-running the same analytics to "see more"

**Failure:** Calling `get_stratification_analytics_data` repeatedly with different `limit` / `offset` to fetch additional data, expecting per-call DB savings.

**Rule:** The full result set is computed on every call regardless of `limit` / `offset` — pagination is response-size only. Use `dataset.next_offset` to page through; do NOT re-execute the same analytics to fetch incremental data. When results are large, narrow with filters first.

**Enforcement:** `get_stratification_analytics_data` tool description.

## Oversized `get_stratification_analytics_data` result

**Failure:** Calling `get_stratification_analytics_data` with no explicit `limit`, or the full 2000-row default/max, on a wide or high-cardinality analytics. The CSV payload exceeds the client's token budget, the tool returns a `result exceeds maximum allowed tokens` error and writes the data to a local file instead of the response, and the agent then answers from data it never actually received.

**Rule:** Pass an explicit, conservative `limit` (start at 50-100 rows for exploratory/narrow questions and report blocks, which only need a handful of figures) rather than relying on the 2000 default. On an oversized-result error, retry the same call with a smaller `limit` and/or a filter from `get_filterable_columns` that narrows the row count — never read or guess at data from the local file path in the error.

**Enforcement:** `abf-gateway` SKILL.md narrow fast path step 1; `abf-report-runner` block-delivery Retrieve step.

## Misreading grain and aggregation in multi-dimension tables

**Failure:** Treating a cell measure (e.g. a per-MOB `Issue Amount` in a vintage row) as a cohort total, ratioing two cell measures to produce a cohort-level metric, or reading a cumulative curve value and a per-period dollar slice on the same row as the same time-grain. Worst on large vintage/cohort tables where no one eyeballs every row.

**Rule:** Before narrating any response with more than one `group_by` dimension, follow `../contracts/reading-stratification-tables.md` — it covers column-role classification (dimension / cell measure / cohort constant / derived ratio / benchmark), grain scoping, cumulative-vs-periodic detection, and a plausibility check before delivering an implausible number as fact.

**Enforcement:** `abf-gateway` SKILL.md Reading discipline section; `abf-report-runner` block-delivery Narrate step.

## Percentage scale confusion

**Failure:** Interpreting a percentage value as 5 when the stored scale is 0.05, or filtering by "5" when the data is `data_type: percentage` (stored 0..1).

**Rule:** Read `stratification://data-types` once per session to internalize the scales. `percentage` stores 0..1 (filter values use 0..1). `percentage_full` stores 0..100 (filter values use 0..100). CSV rows always carry the stored scale; cluster labels render with `%`.

**Enforcement:** Reference resource exists; consult before reasoning numerically about percentages.

## Multi-environment assumption

**Failure:** Defaulting an analytical query to a particular environment (UAT vs. Staging vs. Local) without asking, when multiple ABF MCP servers are connected and the user hasn't named one.

**Rule:** A read-only analytical query can default to a single connected server, but when more than one ABF server is connected and the user has not identified which to query, ask before any tool call.

**Enforcement:** Gateway Step 1.
