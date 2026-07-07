# prism-analyst

Ask questions about your Prism deals directly in Claude Code — NAV, cash flows, portfolio composition, risk metrics — and get answers with confidence scores and source citations back to the originating document and page.

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) installed.
- This marketplace added (see the [root README](../../README.md) — one-time `/plugin marketplace add CardoAI/cardoai-client-plugins`).
- Your Prism MCP endpoint URL. CardoAI provides this to you directly — it is not published in this repo, since it's specific to your account.

## Install

```
/plugin install prism-analyst@cardoai-client-plugins
claude mcp add --transport http --client-id prism prism-mcp https://prism.<your-tenant>.cardoaiapps.com/mcp/
```

Replace `<your-tenant>` with the endpoint CardoAI gave you. The skill expects the MCP server to be registered under the name `prism-mcp` exactly as shown — don't rename it.

On the `claude mcp add` step, Claude Code will open your browser to sign in and authorize access. This is a one-time login per machine.

## Verify it worked

```
/plugin list        # confirms prism-analyst is installed
/mcp                # confirms prism-mcp is connected
```

Then just ask a question, e.g.:

> What's the NAV for `<deal name>` this period, compared to a year ago?

The skill activates automatically whenever you mention a deal name, ask about portfolio data, NAV, sector allocation, KPIs, or similar — no special syntax needed.

## How it works

Behind each answer, several specialized agents retrieve and cross-check data independently (performance, cash flow, portfolio composition, risk), then a final agent reconciles their findings, applies a confidence score (🟢 reliable / 🟡 use with caution / 🔴 verify manually) to every data point, and cites the source document and page. Chart/image data is treated with extra caution and never scored 🟢, since it comes from automated extraction rather than structured data.

If any data point comes back low-confidence, you'll be asked to confirm or correct it before the final report is generated.

## Troubleshooting

- **Skill doesn't seem to activate** — mention a deal name, or ask explicitly: "Use prism-analyst to look up...". Confirm it's installed with `/plugin list`.
- **MCP connection / auth errors** — run `/mcp` to check `prism-mcp`'s status. If it shows disconnected or unauthorized, re-run the `claude mcp add` command above; you may need to re-authorize in the browser.
- **"Already up to date" but you expected new content** — run `/plugin marketplace update` to refresh the marketplace, then `/plugin install prism-analyst@cardoai-client-plugins` again.

## Support

Questions or issues — contact your CardoAI representative, or dev@cardoai.com.
