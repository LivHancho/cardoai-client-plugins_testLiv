# Phase — Discover

**Type:** silent.
**Speaker:** none.
**Owns:** `list_transaction_analytics` calls; domain grouping (held in memory).

## Purpose

Enumerate all available analytics views for the active transaction and group them by domain so the Confirm-scope phase can present scope options to the analyst.

## Entry conditions

The runner has just been invoked by `abf-gateway` for a broad analytical request. This is the runner's entry phase.

## Actions

Call `list_transaction_analytics(transaction_id=<active>, limit=100)`, paging via
`next_offset` until exhausted. Do NOT retrieve analytics data yet.

**Canonical block domains.** Every analytics is mapped to exactly one of the five
canonical block names that the rest of the pipeline (block table, synthesis)
consumes:

`portfolio-overview` · `credit-performance` · `loss-vintage` · `prepayment` · `concentration`

The mapping is computed by combining catalog evidence (preferred) with name-keyword
fallback (when no catalog is loaded). The catalog never replaces these names — it
informs which analytics belong under each.

**Step 1 — Map each analytics to a canonical domain.**

For each returned analytics, assign exactly one canonical domain:

| Canonical domain | Catalog evidence (preferred) | Keyword fallback (when no catalog) |
|---|---|---|
| `portfolio-overview` | `catalog.stratifications[]` matching balance / loan count / WA metrics | analytics name contains `outstanding balance`, `loan count`, `WA ` |
| `credit-performance` | `catalog.risk_metrics[]` for delinquency / DPD / DQ trend | analytics name contains `delinquen`, `DPD`, `days in delay` |
| `loss-vintage` | `catalog.risk_metrics[]` for loss / cohort default | analytics name contains `loss`, `default`, `cohort × term` |
| `prepayment` | `catalog.risk_metrics[]` for CPR / SMM / prepayment / MPR | analytics name contains `CPR`, `SMM`, `prepay`, `MPR` |
| `concentration` | `catalog.stratifications[]` for state / FICO / obligor / sector | analytics name contains `state`, `FICO`, `obligor`, `sector`, `concentration` |

Anything that matches no domain goes under `portfolio-overview` (the catch-all for
general portfolio shape).

**Step 2 — Apply profile-flag suppression (catalog only).**

Check `catalog.profile`:

- `loan_sim_fit: NO` → remove `prepayment` from the offered list; note the suppression
  (reason `loan_sim_fit: NO`) so Confirm-scope can mention it.
- `cf_determinism: NO` → same treatment for `loss-vintage`.
- `loan_sim_fit: PARTIAL` (or other PARTIAL flags) → keep the domain in the offered
  list but hold the affected analytics's catalog caveat so the Confirm-scope and
  Block-delivery phases can surface it.

**Step 3 — Compute the offered domains.**

The offered set is the subset of the five canonical names where:
- at least one live analytics mapped to that domain, AND
- the domain was not suppressed in Step 2.

**Step 4 — Note catalog gaps (informational).**

Any `catalog.risk_metrics[]` or `catalog.stratifications[]` entry with no matching
live analytics (after Step 1) is a catalog gap — a metric the asset-class catalog
expects but that this transaction has no analytics for. Hold these as informational
caveats; the report notes them where relevant so the analyst knows the coverage
boundary. Suppressed-domain gaps are excluded.

Proceed to Confirm scope.

## Exit conditions

The offered domains are known. The runner transitions to Confirm scope.

## Failure modes

- **`list_transaction_analytics` returns empty.** Surface to analyst; do not proceed — there are no analytics to report on.
- **Pagination fails mid-page.** Surface to analyst for direction.

## What the runner must NOT do in this phase

- **No analytics data retrieval (`get_stratification_analytics_data`).** Discovery only.
- **No analyst message.** Confirm scope owns the first conversational moment.

## Cross-references

- `../SKILL.md` §Phase index
