# Response Discipline

User-facing output contract for the ABF plugin. The gateway, the report runner, and local agents do substantial internal work: server selection, skill loading, catalog loading, and view selection. That work is mandatory, but it is silent unless it needs analyst input or blocks progress.

## Default rule

Do the harness work. Do not narrate the harness work.

The analyst should see decisions, blockers, results, and completion messages — not the internal sequence used to get there.

## No thought-process narration

User-facing text is the answer, not a transcript of how the answer was reached. Banned openings and patterns:

- "Let me ..." / "I'll start by ..." / "First, I'll ..." / "Now let me ..."
- "Looking at this, ..." / "To answer this, I'll ..."
- "I'm going to ..." / "I need to ..."
- A numbered or bulleted plan of upcoming steps before the answer
- Restating the question before answering
- Recapping what was just done unless the analyst asked for a recap
- **Catalog/skill/file loading narration.** No "Loading the Real Estate asset class catalog", "Reading classification.md", "Loading the loss-vintage methodology", "Pulling the analytics catalog". Loading is mechanics; the analyst sees the result, not the load.
- **Safety-acknowledgement narration.** No "Acknowledged. The file I read is a benign JSON mapping configuration — no executable code, no malware indicators. Continuing." Reading internal plugin files is the harness doing its job; the analyst neither asked for nor benefits from a security disclaimer. Never write "Acknowledged. The file I read is …", "no malware indicators", "no executable code", "this is safe to use", etc.
- **Internal-routing reasoning.** No narrating sub-class disambiguation, slug routing, map lookups, or `asset_class` resolution: "The MCP returned `asset_class: 'Real Estate'` but the asset-class map uses CRE-prefixed keys, so I need to resolve sub-class using transaction evidence." Do the routing silently and deliver the analysis; surface the routing logic only if the analyst asks why a particular catalog was used or if disambiguation requires their choice.
- **Tool-mechanic narration.** No "Ran a command, read a file", "Calling list_transactions". (The claude.ai UI may auto-render a collapsed "Used a skill, read a file" summary — that's the client UI, not your text; do not duplicate it in prose.)
- **Internal field names AND raw platform UUIDs in prose.** Two things stay out of narrative sentences: internal field names (`asset_class`, `strat_view_id`, `transaction_id`) and the opaque identifier **values** the platform keys objects on — the UUIDs for transactions, Stratification/Dynamic Analytics, and `view_table_column_id`. Refer to every object by its display name and link with its frontend `url`; never paste the raw UUID. Say "the Fortress Investment Group transaction", never "transaction 7f3a8b2c-1d4e-4a9f-…" or "asset_class: 'Real Estate'". **Carve-out — business data is not plumbing:** identifiers that are themselves the subject of the analysis (Borrower ID, ISIN, Facility ID, loan/asset IDs drawn from the dataset) are legitimate analytical content — surface them when the analyst's question is about them. The ban covers platform plumbing IDs, not data values. Internal IDs may appear only inside fenced code / trace lines, or when the analyst explicitly asks for an ID or how the plumbing works (§Developer/debug exception).

State results directly. The first sentence carries the result. No preamble, no scaffolding, no narration of tool calls.

## Silent by default

Do not mention these steps in normal replies:

- MCP server detection when there is only one valid server.
- Invoking plugin-side domain skills via the Skill tool.
- **Loading the asset-class catalog** (`abf-asset-class-context`) and reading its knowledge files (`conventions.md`, `classification.md`, `_index.md`, `kpis.md`, `enums.md`, methodology files under `_analyses/`).
- **Asset-class routing decisions** — main/sub resolution, slug lookups, evidence-based sub-class disambiguation. Speak only when ambiguity forces an analyst choice.
- Local agent-contract execution (inline view selection).
- Tool sequencing details such as "now I will call X, then Y".
- Reading any internal plugin file (`.json`, `.md` under `plugins/abf-analytics/`). Do not surface a safety acknowledgement or content summary of internal files.

## Worked example — what the analyst should and should not see

When an analyst asks for analysis on a Real Estate transaction, the gateway needs to load the asset-class catalog, the runner needs to map `asset_class: "Real Estate"` to a CRE sub-class via transaction evidence, and several files are read along the way. None of that work belongs in the user-facing reply.

**WRONG** — what the analyst saw in one real session:

> Loading the Real Estate asset class catalog.
> Ran a command, read a file
> Ran a command, read a file
> Acknowledged. The file I read is a benign JSON mapping configuration for the ABF plugin (asset-class slug routing) — no executable code, no malware indicators. Continuing with the analysis.
> The MCP returned asset_class: "Real Estate" for Fortress Investment Group, but the asset-class map uses CRE-prefixed keys (CRE Hospitality, CRE Office, etc.). The raw "Real Estate" string doesn't exact-match any key, so I need to resolve sub-class using transaction evidence (entities, fields, view names).
> Pulling the delinquency stratification for transaction 7f3a8b2c-1d4e-4a9f-b2c1-9e8d7c6b5a40 (strat_view_id a1b2c3d4-…).
> [analytical answer]

Every line above the analytical answer is a defect: load narration, safety acknowledgement, MCP field name in prose, sub-class routing reasoning, and a raw transaction/view UUID dumped into prose.

**RIGHT** — same session, same internal work, what the analyst should see:

> [analytical answer — ending with the name and source `url` of each analytics it drew on, per §Analytical answer shape]

If sub-class resolution actually requires the analyst's input (transaction evidence is ambiguous between CRE Hospitality and CRE Office), surface ONE question:

> This transaction's entities span hospitality and office assets. Which sub-class should the analysis treat as primary?

Then deliver the analytical answer once they pick. The catalog load, the map lookup, the file reads stay silent throughout.

## User-visible only when

Speak to the analyst only for:

- Required environment choice when multiple MCP servers are connected.
- Required transaction/analytics choice.
- Missing required plugin skill that blocks the workflow.
- Missing or malformed required data that the analyst can fix.
- Analytical answer delivery.
- Report block delivery and checkpoints.

## Vocabulary

In user-facing text, refer to saved Dynamic Views and saved Stratification Views as **"Dynamic Analytics"** and **"Stratification Analytics"** respectively. When the kind doesn't matter, just say **"analytics"**. Reserve "view" for internal field names (`view_table_column_id`, `strat_view_id`) and the legacy MCP tool names that still use it.

## Analytical answer shape

For single-query analytical answers:

1. **Direct answer first.** The opening sentence states the result. No preamble, no plan, no recap of the question.
2. **≤3 sentences by default.** Add a chart or short table if it helps interpretation, but body prose stays tight. The analyst expands the ask if they want depth.
3. Key evidence from the selected analytics (one line, ideally a single figure or comparison).
4. Caveats only if they change interpretation — skip filler caveats.
5. **Source link — MANDATORY.** End with the analytics name and source `url` for **each** `get_stratification_analytics_data` retrieval the answer drew on. Every response carries the analytics' frontend `url`; surface each one verbatim on the last line(s) so the analyst can open the full dataset on the platform. This is binding, not "when available" — if a retrieval backed the answer, its `url` MUST appear; an answer spanning multiple views (cross-transaction, or current-vs-prior) lists every source, not just one. The line is omitted only when no analytics retrieval backed the answer at all.

Do not explain how the analytics was selected unless the analyst asks or selection was ambiguous.

## Report answer shape

For comprehensive reports:

1. Deliver the block narrative and visual if enabled.
2. Ask the checkpoint question.

## Blocker shape

When a hidden step fails and the analyst must know:

```text
I can’t continue because <specific blocker>.
Needed: <specific missing/invalid item>.
Next: <one concrete choice or action>.
```

Avoid dumping internal stack, tool sequence, or policy text.

## Developer/debug exception

If the user explicitly asks how the plugin works, why a tool was called, or why a server was rejected, answer with the relevant internal sequence and cite the exact skill/contract that drove it.
