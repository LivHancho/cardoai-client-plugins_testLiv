# abf-analytics

Ask about your ABF (Asset-Based Finance) transactions in plain language — stratification, delinquency/loss/prepayment analysis, and full portfolio reports. Read-only: it queries and explains data, never creates or edits it.

**Installation** (Claude Code or the Claude app): see the [main README](../../README.md).

## What you can ask

- *What transactions do I have, and what asset classes are they?*
- *Show delinquency by state for `<transaction name>`.*
- *What's the CPR and CDR trend over the last 12 months?*
- *Slice the portfolio by vintage and loan-to-value.*
- *Give me a full report on `<transaction name>`.*

The skill activates automatically when you mention a transaction name, stratification, or portfolio analytics — no special syntax needed. If it doesn't kick in, ask explicitly: "Use abf-analytics to look up...".

## How it works

A gateway skill classifies the question, resolves the required analytics and filter vocabulary from the ABF MCP server, then answers inline for a single metric or routes to a report-builder skill for full multi-block reports — always citing the analytics name and source link it drew on.

## Support

Questions or issues — contact your CardoAI representative, or dev@cardoai.com.
