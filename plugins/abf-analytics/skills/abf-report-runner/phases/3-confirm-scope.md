# Phase — Confirm Scope

**Type:** analyst conversation (the first analyst-visible phase in every run).
**Speaker:** runner (presents scope); analyst (confirms).
**Owns:** scope confirmation (held in memory).

## Purpose

Present the discovered domains to the analyst and ask whether they want the full report or specific sections. Record the confirmed scope before any block delivery begins.

## Entry conditions

The offered domains are known from the Discover phase.

## Actions

This is the analyst's first conversational moment in the run. Keep the framing brief before presenting scope:

> "I'll walk through the report by section. You can continue, skip, or jump to key findings at each checkpoint."

Present the discovered domains to the analyst and ask whether they want the full report or specific sections. If Discover suppressed a domain via a catalog flag, or recorded an applicability caveat, mention it here so scope is set with eyes open.

After the analyst replies:
- Record which domains are in scope and which were skipped (held in memory for the rest of the run).
- For every skipped domain, skip the block whose `block_name` matches it — each canonical domain (`2-discover.md`) shares its name with exactly one `block_name` in `4-8-block-delivery.md`.
- Proceed to the first in-scope block (Block delivery).

## Exit conditions

In-scope and skipped domains are recorded. The runner transitions to Block delivery.

## Failure modes

- **Analyst skips all domains.** Acknowledge and route to Handoff (no blocks to deliver).
- **Analyst's reply is ambiguous.** Clarify before recording scope — do not assume.

## What the runner must NOT do in this phase

- **No analytics data retrieval.** Block delivery phases own that.
- **No scope assumption.** Always wait for analyst confirmation.

## Cross-references

- `4-8-block-delivery.md` — the block sequence this scope selects from
