---
name: abf-asset-class-context
description: >
  Use when an ABF transaction's asset class is known and you need its KPI catalog, entity model,
  applicability profile, or enum vocabulary. Triggers when list_transactions reveals a transaction's
  asset_class, when an analyst names an asset class (auto loans, credit cards, CRE hospitality, trade
  finance, cell tower, etc.), or asks "what KPIs / stratifications / dimensions apply to X". The
  knowledge ships on disk with this skill; this skill maps the asset_class string to the right files
  via lib/asset-class-map.json and loads only what the question needs.
---

# ABF Asset-Class Context — Gateway

The asset-class catalog **ships on disk with this skill**, under `knowledge/`. The MCP server
does **not** serve it — the server only exposes the list of asset-class *names* (via
`list_transactions` and the `securitizations://asset-classes` resource). This skill is the
**protocol** for loading the local knowledge progressively: resolve the transaction's
`asset_class` string to a `{main, sub}` slug through `lib/asset-class-map.json`, then read only
that class's files with the `Read` tool. Never load the whole catalog. Never load another
class's files. There is **no `read_resource` for the catalog**.

## When to invoke

Load on ANY of:
- An ABF transaction is identified and its `asset_class` is known (from `list_transactions`).
- The analyst names an asset class ("auto loans report", "hospitality stratifications").
- The analyst asks "what KPIs / metrics / dimensions / data apply to {asset_class}".
- The `abf-report-runner` requests catalog injection.

Skip when:
- This conversation already loaded the catalog for the same asset class — reuse it.
- No transaction is active AND no asset class has been named — wait.
- The analyst asks about a non-ABF topic.

## Where the knowledge lives

Everything is a local file read with the `Read` tool, relative to this skill's directory:

- `lib/asset-class-map.json` — the **routing index**. Maps each platform `asset_class` string
  to `{main, sub}` slugs plus synonyms. The entry point; read once per session.
- `knowledge/_global/` — global tier (`conventions.md`, `classification.md`), loaded for every class.
- `knowledge/{main}/_index.md` — family entity model + the **Sub-classes table** carrying each
  sub's `CF Det.` and `Sim Fit` applicability flags (the profile `cf_determinism` / `loan_sim_fit`
  that report-runner and the `_analyses/` gates consume).
- `knowledge/{main}/common/kpis.md`, `knowledge/{main}/{sub}/{kpis,enums,data-model}.md` — the
  resolved class's sections.
- `knowledge/_analyses/{loss-vintage,prepayment,concentration}.md` — cross-cutting methodology,
  gated by the profile flags.

## Protocol

### 1. Read the routing index (once per session)

`Read lib/asset-class-map.json`. Cache it. Its `mappings` key each platform `asset_class` string
("Auto Loans", "CRE Hospitality") to `{main, sub, synonyms}`.

### 2. Load the global tier

`Read knowledge/_global/conventions.md` and `knowledge/_global/classification.md`.
`classification.md` **is** the resolution algorithm — follow it in the next step.

### 3. Resolve exactly one asset class

Source the `asset_class` string from the active transaction (`list_transactions`; the valid
enum is `securitizations://asset-classes`) or the analyst's phrasing. Apply `classification.md`:
exact map-key match → synonym match → fuzzy match → entity-shape heuristic → unresolved
fallback. No confident match → load only the global tier, surface the closest candidates, and
ask the analyst to pick. Never guess a `{main, sub}`.

### 4. Load the resolved class — and only it

In resolution order: `Read knowledge/{main}/_index.md`, `knowledge/{main}/common/kpis.md`,
`knowledge/{main}/{sub}/kpis.md`, `knowledge/{main}/{sub}/enums.md`. For CRE, also
`knowledge/cre/_addendum/data-model.md`. **Never read a file under a different `{main}/{sub}`.**
The sub's row in `{main}/_index.md` supplies its `CF Det.` / `Sim Fit` profile flags.

### 5. Load on-demand sections only when the question needs them

- `knowledge/{main}/{sub}/data-model.md` — data-availability / "what fields exist" / field-grounding.
- `knowledge/_analyses/loss-vintage.md` — loss-curve / cohort-default questions. Gated by the sub's
  `CF Det.` flag; if `NO`, loss-vintage does not apply — say so, do not invent a curve.
- `knowledge/_analyses/prepayment.md` — prepayment-speed / adverse-selection questions. Gated by
  `Sim Fit`; if `NO`, the standard CPR/SMM model does not apply — name the class's own proxy instead.
- `knowledge/_analyses/concentration.md` — geographic / FICO / obligor / sector concentration.

### 6. Family-compare guard

When comparing multiple sub-classes **within one family**, read ONLY that family's `_index.md`
and `common/kpis.md`. Do not read the individual sub-class sections.

### 7. Override rule

When a class's own `{sub}/kpis.md` and its family `common/kpis.md` define a KPI with the same
`### name`, the class-specific one wins.

## Output contract — never leak internal routing

The `{main, sub}` slugs, family names, tier names, filenames, and the map structure are
**internal routing metadata**. Surface NONE of them.

- Show ONE asset-class label — the map KEY ("Auto Loans", "CRE Hospitality", "Cell Tower").
  Never "consumer — auto-loans", a `main` name, or a filename.
- KPI / enum counts and dimension lists are merged across loaded files — do not split by tier or
  say which file a metric came from.
- Never say "inherits from", "tier", "common", "section", or expose a file path.
- "Where does this metric come from?" → "Cardo standard catalog."
- Surface `CF Det.` / `Sim Fit` only as plain applicability statements (e.g. "Credit Cards uses
  revolving master-trust dynamics, so standard CPR/CDR/vintage models don't apply — use Monthly
  Payment Rate framing instead").

## Authority boundary

The catalog defines what is **relevant** for an asset class; it is not a calculation authority.
Ground every number in live MCP data (`get_stratification_analytics_data`). A catalog
`computation` string is a framing reference, not a substitute for the server's computed value.

## Never

- Resolve to more than one asset class, or read a second class's files after resolving one.
- Read a `{main}/{sub}` that the map did not hand you for the resolved class.
- Load on-demand sections speculatively.
- Surface slugs, family/main names, tier names, filenames, or "inherits from" to the analyst.
- Present a catalog `computation` as an observed value.
- Call `read_resource` for the catalog — it is on disk; read it with `Read`.
