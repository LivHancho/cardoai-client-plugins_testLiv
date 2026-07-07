# CardoAI Client Plugins

Plugins from [CardoAI](https://cardoai.com) for asking Claude about your data.

| Plugin | What it does |
|---|---|
| [`prism-analyst`](plugins/prism-analyst/README.md) | Ask questions about your Prism deals in plain language — verified answers with confidence scores and source citations. |

## Before you start

You need your **Prism endpoint URL**. CardoAI sends it to you directly — it's specific to your account, so it's not published here. It looks like `https://prism.<your-tenant>.cardoaiapps.com/mcp/`.

Wherever setup asks for an **OAuth client ID**, enter `prism-mcp` — it's the only value Prism accepts, the same for every client.

There are two ways to install. Pick the app you use:

- **[Claude Code](#install-in-claude-code)** — the terminal, where you type commands like `/plugin`.
- **[Claude app](#install-in-the-claude-app-claudeai)** — claude.ai in the browser, or the desktop chat app (no terminal).

## Install in Claude Code

**Step 1 — inside Claude Code**, add the CardoAI plugin catalog, then install the plugin:

```
/plugin marketplace add CardoAI/cardoai-client-plugins
/plugin install prism-analyst@cardoai-client-plugins
```

**Step 2 — in your terminal**, connect your Prism data:

```
claude mcp add --transport http --scope user --client-id prism-mcp prism https://prism.<your-tenant>.cardoaiapps.com/mcp/
```

| Part | What it is |
|---|---|
| `--client-id prism-mcp` | The OAuth client ID. **Required** — Prism only accepts `prism-mcp`. |
| `prism` | The local name the connection is saved under. Any name works — this is not the client ID. |
| `--scope user` | Makes the connection work in every folder, not just where you ran the command. |
| the URL | Your Prism endpoint — paste yours in place of the example. |

Your browser opens to sign in and authorize — one time per machine.

**Step 3 — verify**, back inside Claude Code:

```
/plugin list        # prism-analyst is installed
/mcp                # prism is connected
```

Then just ask, e.g. *"What's the NAV for `<deal name>` this period, compared to a year ago?"*

## Install in the Claude app (claude.ai)

No terminal needed. You add two things: the **prism-analyst plugin** (from the CardoAI marketplace) and the **Prism custom connector** (the link to your data).

> Requires a paid plan (Pro, Max, Team, or Enterprise). On Team/Enterprise, only an organization **Owner** can add the connector, and your admin may restrict which plugins you can install — ask your admin if you don't see an option. Answers in the app are sourced and confidence-scored, but the full multi-agent cross-checking runs only in Claude Code.

**1. Install the plugin.**

- Open **Customize** in the left sidebar, then go to the **Plugins** tab.
- Under **Personal plugins**, click **＋** → **Add marketplace** → **Add from a repository**.
- Enter this repository: `CardoAI/cardoai-client-plugins` (the full GitHub URL works too)
- Browse the new CardoAI marketplace and click **Install** on **prism-analyst**.

**2. Add the connector.** Prism is a **custom connector** — you won't find it in Claude's built-in connector directory, so don't search for it there. Go to Settings → **Connectors**, scroll past the pre-built connectors, and click **Add custom connector**.

- **Name:** `prism` (any name works — this is just the label, not the client ID)
- **URL:** your Prism endpoint from CardoAI
- Open **Advanced settings** and set **OAuth Client ID** to `prism-mcp`. **This is required — the connection fails without it.** Leave the client secret blank unless CardoAI gave you one.

Click **Add**, then **Connect** and sign in.
*(Team/Enterprise: an Owner adds it under Organization settings → Connectors → Add → Custom → Web, with the same values — including **OAuth Client ID** `prism-mcp` under Advanced settings; each member then clicks **Connect**.)*

**3. Ask.** In a new chat, ask e.g. *"What's the NAV for `<deal name>` this period?"* — type **/** to see the skills your plugins added.

## Troubleshooting

- **Skill doesn't seem to activate** — mention a deal name, or ask explicitly: "Use prism-analyst to look up...". Claude Code: confirm it's installed with `/plugin list`. Claude app: check **Customize → Plugins** shows prism-analyst as installed.
- **Connection / auth errors** — first confirm the OAuth client ID is exactly `prism-mcp` (the most common mistake). Claude Code: run `/mcp`; if the connection shows disconnected or unauthorized, re-run the `claude mcp add` command above and re-authorize in the browser. Claude app: Settings → Connectors → open the Prism connector, check the Client ID under Advanced settings, then click **Connect** again.
- **Plugin seems outdated** — Claude Code: run `/plugin marketplace update`, then `/plugin install prism-analyst@cardoai-client-plugins` again. Claude app: **Customize → Plugins**, refresh the CardoAI marketplace and reinstall.

## Support

Questions or issues — contact your CardoAI representative, or dev@cardoai.com.

## License

Proprietary — for licensed use by CardoAI clients only. Contact CardoAI for terms.
