---
name: abf-gateway
description: >
  Use when the user asks about ABF CardoAI data, transactions, securitizations, deals, or portfolio analytics.
  Triggers on: Transaction lookups, stratification analytics, portfolio analytics, delinquency/loss/prepayment analysis,
  comprehensive reports, cross-transaction comparison, browsing available transactions, exploring available analytics,
  understanding filterable columns, waterfall/cash-flow/tranche/distribution analysis.
---

# ABF MCP Gateway

Single entry-point for all ABF CardoAI platform analytical requests. Invokes plugin-side domain skills (analytics selection, asset-class context, report synthesis) before any MCP tool call so vocabulary, the analytics catalog, and the required tool chains are respected. This plugin is **read-only** — it reads, queries, and explains ABF analytics data; it never creates, edits, or persists analytics objects.

## When to invoke

Trigger on ANY of:
- ABF / transaction / securitization / deal / portfolio questions
- "Slice / stratify / chart / show me by [dimension]"
- Waterfall / cash-flow / tranche / distribution analysis
- "What transactions do I have?" / "What asset classes?" / "What analytics are available?"
- "What columns/fields can I filter on?" / exploring the data model

## Non-negotiable: source url
Every analytical answer that draws on a `get_stratification_analytics_data` retrieval ends with that retrieval's source `url`, verbatim, on the last line(s) — one per analytics used. No exceptions, no "when available." Full rule and UUID-in-prose carve-out: `knowledge/contracts/response-discipline.md`.

Skip when:
- The user is asking about the platform CODEBASE (architecture, file paths) — that's code reading, not MCP
- The relevant skill was already loaded this conversation — don't reload
- No ABF MCP server is connected — say so and stop

## Response discipline

Follow `knowledge/contracts/response-discipline.md` for every user-facing reply. The short version: internal harness work is silent by default. Do not narrate server detection, skill loading, or tool sequencing. Speak only when the analyst must choose, when a blocker requires action, when delivering analysis or report content, or when confirming completion.

## Reading discipline

Follow `knowledge/contracts/reading-stratification-tables.md` before narrating any `get_stratification_analytics_data` response with more than one `group_by` dimension (vintage, cohort, MOB × date). Grain and aggregation-function misreads on these tables are the leading cause of wrong analysis, especially once the row count is large enough that no one eyeballs every row.

## Step 1: Confirm ABF MCP availability and environment

Find the ABF server by looking for ABF-specific MCP tools — `list_transactions`, `list_transaction_analytics`, `get_stratification_analytics_data`, `get_filterable_columns`, `list_resources`, `read_resource` — among the available tools. If none are present, say so and stop.

Call `list_resources` once per session, before the first `read_resource` call, to discover the actual resource URIs and templates this server exposes — including any `securitizations://` catalogs. Do not guess a resource URI from a tool description; confirm it exists in `list_resources`' output first.

If multiple ABF MCP servers are connected (e.g., both UAT and Staging) and the user has not identified which environment to query, **ask before any tool call** — including read-only analytical queries. Once confirmed, use only that server's tools for the rest of the session — including `read_resource` for knowledge and reference resources; never mix one environment's catalog with another environment's data.

## Step 2: Classify scope, then resolve asset class only when needed

Classify the question scope — verb/noun based, needs no catalog:

- **`narrow`** — a single metric / dimension / time-point ("What's the CPR?", "Show delinquency by state").
- **`broad`** — a report, overview, or multiple KPI areas ("Analyse this portfolio").
- Default to `narrow` when uncertain.

**Narrow fast path — answer inline, no skill hops.** For `narrow`, do NOT load `abf-asset-class-context` and do NOT enter `abf-report-runner`:

1. Run the required chain directly: `list_transaction_analytics` → `get_filterable_columns` (call this whenever a filter / bucket / status / category is implied — it returns the allowed labels; never guess them) → `get_stratification_analytics_data` with an explicit, conservative `limit` (start ~50-100 rows, not the 2000 default/max — see `knowledge/reference/common-pitfalls.md`). Trust the CSV glossary over column names.
2. Use the active `transaction_id` from session context. Call `list_transactions` only when the transaction must still be identified, or for cross-transaction work.
3. Deliver the answer, ending with the analytics name and source `url` for each retrieval it drew on.

**Broad — build the catalog first.** When an active `transaction_id` is known:

1. Call `list_transactions()` and find the entry whose `id` matches `transaction_id` (no ID filter — scan the list; use `list_transactions(name=<name>)` if named). Extract `asset_class`. **Pagination:** default `limit` is 20; pass an explicit `limit` (e.g. `limit=200`) or page with `offset` for large tenants.
2. Follow the `abf-asset-class-context` skill protocol (index → resolve one class → parallel section batch) to build `{asset_class_catalog}` in memory.
3. Reuse the catalog for `(transaction_id, asset_class_label)` if already loaded this session — never reload the same key.
4. Route to `abf-report-runner`.

If no transaction is active and no asset class is named, resolve it once a transaction is identified.

## Step 3: Visuals

Deliver visuals opportunistically by default — text and markdown tables for simple answers, and a chart, heatmap, or interactive table when the data is clearly visualization-friendly (`knowledge/reference/visuals-playbook.md`). If the analyst says "turn visuals off", "no charts", or "always chart this", honor it for the rest of the session.

**Mandatory pre-render gate — run this before calling any chart/table rendering tool, every time:**

1. List, explicitly, every row/cohort/vintage/category dropped, capped, deduplicated, re-bucketed, or re-aggregated versus what the retrieved analytics returned raw.
2. If that list is non-empty and the user did not explicitly request that exact treatment in this conversation, **stop. Do not render.** Ask the user how to handle each item instead (include as-is / flag distinctly / exclude). This question IS the entire response — do not ask and then render anyway in the same turn.
3. If the list is empty, or the user already specified the treatment, proceed to render.
4. Any gap in a reported time series (no data for some x-axis span) must render as a true break (`connectNulls: false` / `spanGaps: false` or equivalent) — never an interpolated line across missing periods.

This gate is not satisfied by mentioning the filtering choice in prose after the chart is shown. See `knowledge/reference/common-pitfalls.md` § "Silent filtering of rows, cohorts, or vintages" for failure examples.

## Step 4: Follow chain references

Loaded skills hand off to others ("use the `abf-Y` skill"). Invoke each named skill once via the Skill tool. Never reload a skill already used this conversation.

Routing:

| Request type | Handler |
|---|---|
| Analytical — narrow | answered inline by the gateway (Step 2 fast path) — no skill hop |
| Analytical — broad | `abf-report-runner` — catalog + question drive response width up to a full multi-block report |

## Step 5: Treat loaded skills as binding

- **Required tool chains** — follow the MCP server's documented chain order (`list_transaction_analytics` → `get_filterable_columns` → `get_stratification_analytics_data`); don't shortcut.
- **Vocabulary and enums** — call `get_filterable_columns` for allowed filter labels; never guess delinquency-bucket / status / category values.

When a loaded skill says "never X" or "always Y first", that is binding behavior, not a suggestion.

## Local agent contracts

- `agents/view-selector.md` — consumed by `abf-report-runner`; selects the stratification analytics + filters for a single analytical question. Inline only.

## Reference knowledge

On-demand references; the gateway does not preload them.

- `knowledge/reference/vocabulary.md` — ABF business terms.
- `knowledge/reference/common-pitfalls.md` — defensive patterns.
- `knowledge/reference/derived-metrics.md` — SMM/CPR/MoM/concentration-drift formulas.
- `knowledge/reference/visuals-playbook.md` — chart-type defaults per analysis pattern.
- `knowledge/reference/artifact-style.md` — colors, typography, and page layout for any rendered artifact (HTML dashboard, exported document, image). Consult whenever producing one.

## Never

- Answer an ABF MCP request from priors without reading the relevant skill. The vocabulary and analytics catalog are not in training data.
- Reuse a `strat_view_id` across transactions.
- Re-run the same analytics with different `limit`/`offset` to "see more" — pagination is response-size only; page with `dataset.next_offset` or narrow with filters.
