# Phase — Synthesize

**Type:** analyst conversation.
**Speaker:** runner (delivers key takeaways).
**Owns:** key-takeaways delivery.

## Purpose

Synthesize the delivered blocks into key takeaways and deliver them to the analyst. Uses only the block narratives and figures already delivered this conversation — no new MCP data retrieval.

## Entry conditions

At least one block has been delivered this run (some may be skipped). The analyst either completed all blocks or jumped directly to findings.

## Synthesis guidelines

- **Find contradictions** — surface metrics that diverge from underlying data (e.g., low overall DQ rate but rapid 30-60 bucket growth).
- **Chain second-order effects** — link cause to downstream effect (adverse selection → composition drift → increased loss sensitivity).
- **Flag anomalies** — with 2-3 plausible explanations, not just one.
- **Quantify everything** — every claim needs a specific number. No qualitative statements.
- **Name the mechanism** — differential prepayment speeds, underwriting drift, negative selection. Name what drives the observation.
- **Cross-reference** — pull from 2-3 data domains when making a takeaway.
- **End with resolution signal** — what would confirm or alleviate the concern (e.g., "watch the 30-60 pipeline for forward migration to 60-90").

## Output shape (per takeaway)

- **Bolded title** naming the phenomenon (e.g., "The 651-700 FICO Concentration Trap").
- 1-2 paragraphs with data and mechanism.
- Closing sentence with confirmation/alleviation signal.

## Report structure

- **Positive Indicators** — numbered list (3-5 items) with supporting data.
- **Risk Factors & Watchpoints** — numbered list (3-6 items) with specific thresholds.
- **Key Takeaways** — 4-6 named, bolded findings with 1-2 paragraphs each.

## Scope rule

Only reference data from sections that were actually delivered in this session. If a section was skipped, explicitly note:

> "Prepayment analysis was skipped; findings do not include adverse selection or composition drift assessment."

## Example takeaway

**The 651-700 FICO Concentration Trap**
The 651-700 bucket — already 63% of the portfolio — has the lowest prepayment speed of any FICO tier (0.59% monthly SMM, 5.72% cumulative by T10). Every other bucket exits faster, so the 651-700 share mechanically grows by ~99 bps over 12 months. Meanwhile the 801-850 bucket is evaporating: 26.3% cumulative prepayment by T3, 9.67% monthly SMM. This compounds with the DQ acceleration: 90+ DPD went from 0% to 2.24% in six months, and 97.8% of defaults sit in sub-750. A reversal would require prepayment speeds to converge across FICO or a new origination vintage reweighting the pool toward prime.

## Actions

Apply the synthesis guidelines above against the block narratives and figures already delivered this conversation. The runner does NOT retrieve new data — synthesis works from what has already been delivered.

After delivery, proceed to Handoff.

## Exit conditions

The key takeaways have been delivered to the analyst. The runner transitions to Handoff.

## Failure modes

- **No blocks delivered (all skipped or analyst jumped here immediately).** Still produce key takeaways from whatever partial data is available; surface a note that the synthesis is based on limited scope.

## What the runner must NOT do in this phase

- **No `get_stratification_analytics_data` calls.** Synthesis reads from what has already been delivered.
- **No reload of any skill already read this conversation.**

## Cross-references

- `4-8-block-delivery.md` — the blocks this phase synthesizes
