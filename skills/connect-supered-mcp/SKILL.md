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

The plugin ships an `.mcp.json` that registers the `supered` server automatically. To activate:

1. Run `/reload-plugins` (or restart Claude Code).
2. On first use, Claude Code opens the OAuth flow in the browser — the user signs in to Supered
   and approves access. Tokens are managed by Claude Code; you never handle them directly.
3. Verify by calling `whoami`. A successful response with the user's `team` confirms the link.

## Manual connection (no plugin, or other MCP clients)

If the server isn't registered, add it directly:

```
claude mcp add --transport http Supered https://app.supered.io/mcp
```

This is the same command surfaced on Supered's own `/mcp` landing page. Then complete the OAuth
sign-in and verify with `whoami`.

## Verifying and troubleshooting

- **Verify:** `whoami` returns `user` and `team`. If it returns the wrong team, the user is
  signed into the wrong Supered account — have them re-auth.
- **No tools after connecting:** run `/reload-plugins`.
- **A specific tool is missing:** it's likely gated behind a feature flag the team doesn't have.
  Don't work around it — tell the user that capability isn't enabled for their plan.

## Guardrails

- Do not ask the user to paste tokens, API keys, or passwords into chat. Authentication happens
  through the OAuth browser flow only.
- If OAuth needs to be (re)authorized, ask the user to complete it themselves in the browser.
