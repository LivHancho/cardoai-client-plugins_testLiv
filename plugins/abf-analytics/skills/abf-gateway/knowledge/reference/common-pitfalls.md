# Common Pitfalls

Defensive patterns the gateway and report runner should follow proactively. Each pitfall lists the failure mode, the rule, and where the rule is enforced.

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

## Percentage scale confusion

**Failure:** Interpreting a percentage value as 5 when the stored scale is 0.05, or filtering by "5" when the data is `data_type: percentage` (stored 0..1).

**Rule:** Read `stratification://data-types` once per session to internalize the scales. `percentage` stores 0..1 (filter values use 0..1). `percentage_full` stores 0..100 (filter values use 0..100). CSV rows always carry the stored scale; cluster labels render with `%`.

**Enforcement:** Reference resource exists; consult before reasoning numerically about percentages.

## Multi-environment assumption

**Failure:** Defaulting an analytical query to a particular environment (UAT vs. Staging vs. Local) without asking, when multiple ABF MCP servers are connected and the user hasn't named one.

**Rule:** A read-only analytical query can default to a single connected server, but when more than one ABF server is connected and the user has not identified which to query, ask before any tool call.

**Enforcement:** Gateway Step 1.
