---
name: abf-setup
description: >
  Use when the user wants to connect to the ABF CardoAI MCP server, runs /abf-setup,
  or is told that ABF MCP is not connected. Guides through platform-specific connection steps.
---

# ABF Setup

Guide the user through connecting to the ABF CardoAI MCP server.

## MCP Server URL

The ABF MCP server URL follows this pattern:

```
https://abf-backend.{tenant}.cardoaiapps.com/mcp
```

Ask the user to replace `{tenant}` with their tenant name (provided by CardoAI).

## Step 1: Client

Ask: "Are you on **Claude Desktop / claude.ai** or **Claude Code** (the CLI)?"

## Claude Desktop / claude.ai

1. Open **Settings** → **Connectors**
2. Click **Add custom connector**
3. Paste your MCP URL with your tenant name substituted
4. Expand **Advanced settings**
5. Set **OAuth Client ID** to `abf-mcp`
6. Click **Connect** and complete the OAuth login
7. Start a new conversation — the connection loads at session start

## Claude Code

Run in your terminal, replacing `{tenant}` with your tenant name:

```bash
claude mcp add --transport http --scope user --client-id abf-mcp abf https://abf-backend.{tenant}.cardoaiapps.com/mcp
```

`--client-id abf-mcp` is **required** — the ABF backend does not support Dynamic Client Registration, so the OAuth handshake fails without it. `abf` is just the local name the connection is saved under (not the client ID). Your browser opens to sign in and authorize.

Run `/mcp` to confirm `abf` shows as connected. If you see an auth error or the OAuth flow loops, the client ID is the first thing to check.
