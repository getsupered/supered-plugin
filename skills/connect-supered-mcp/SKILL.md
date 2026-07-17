---
name: connect-supered-mcp
description: >
  Connect Claude to the Supered MCP server so it can read and write the user's Supered
  environment. Use when Supered tools aren't available, `whoami` fails, the user asks to
  "connect Supered", "set up the Supered MCP", "authorize Supered", or is starting onboarding
  and hasn't linked their account yet.
---

# Connect the Supered MCP

The Supered MCP is an HTTP server at `https://app.supered.io/mcp`, authenticated with OAuth2.
It exposes the Supered public API (collections, content, guides, page triggers, process boards,
sync-engine rules, action plans, help center, and more) as tools. Available tools are
**feature-flagged per team**, so a user only sees what their plan enables.

## If this plugin is installed

The plugin ships an `.mcp.json` that registers the `supered` server automatically. To activate
and sign in:

1. **Activate it.** Restart Claude Code (quit and reopen) so the new plugin's MCP server loads.
   On recent versions you can instead run `/reload-plugins` without restarting — but if that
   returns "Unknown slash command," the Claude Code version is older than that feature; just
   restart (or update Claude Code) instead.
2. **Authenticate.** The OAuth sign-in is not always automatic. Run `/mcp`, select the
   **Supered** server, and choose **Authenticate** — this opens the browser sign-in. (Shell
   equivalent, outside a session: `claude mcp login supered`.) Tokens are managed by Claude Code;
   you never handle them directly.
3. **Verify** by calling `whoami`. A successful response with the user's `team` confirms the link.

## Manual connection (no plugin, or other MCP clients)

If the server isn't registered, add it directly:

```
claude mcp add --transport http Supered https://app.supered.io/mcp
```

This is the same command surfaced on Supered's own `/mcp` landing page. Then complete the OAuth
sign-in and verify with `whoami`.

## Verifying and troubleshooting

- **Verify:** `whoami` returns `user` and `team`. If it returns the wrong team, the user is
  signed into the wrong Supered account — have them re-auth via `/mcp`.
- **No tools after installing:** the plugin's MCP server hasn't loaded yet — restart Claude Code
  (or `/reload-plugins` on versions that support it), then authenticate via `/mcp`.
- **Tools present but every call fails auth:** the server isn't authenticated — run `/mcp`,
  select Supered, and Authenticate.
- **A specific tool is missing:** it's likely gated behind a feature flag the team doesn't have.
  Don't work around it — tell the user that capability isn't enabled for their plan.

## Guardrails

- Do not ask the user to paste tokens, API keys, or passwords into chat. Authentication happens
  through the OAuth browser flow only.
- If OAuth needs to be (re)authorized, ask the user to complete it themselves in the browser.
