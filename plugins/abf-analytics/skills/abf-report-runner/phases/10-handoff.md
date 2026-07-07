# Phase — Handoff

**Type:** analyst conversation (terminal acknowledgement).
**Speaker:** runner (terminal message).
**Owns:** terminal acknowledgement.

## Purpose

Acknowledge the terminal outcome to the analyst and return control to the gateway. This phase is the end of the runner's lifecycle for this report run.

## Entry conditions

The report is complete (every in-scope block delivered + synthesis) or the analyst stopped the run early.

## Actions

- **Completed** — every in-scope block delivered + synthesis. Acknowledge to the analyst with a one-line close.
- **Stopped early** — analyst said "stop" mid-run. Acknowledge with a brief note on what was covered.

The runner does not initiate further work. Subsequent analyst questions go through the gateway from the top.

## Exit conditions

The analyst has received the terminal acknowledgement. The runner has returned control to the gateway.

## What the runner must NOT do in this phase

- **No new workflow initiation.** Subsequent requests go through the gateway.
- **No reload of any skill already read this conversation.**

## Cross-references

- `../../abf-gateway/SKILL.md` — gateway re-entry point for subsequent requests
