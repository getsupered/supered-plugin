# Supered plugin for Claude

Set up and manage a [Supered](https://supered.io) environment directly from Claude. This repo is
both a **plugin** and a **marketplace** — installing it connects the Supered MCP and gives Claude
a set of skills for onboarding and building out a Supered workspace.

## What it does

- **Connects the Supered MCP** (`https://app.supered.io/mcp`, OAuth2) so Claude can read and
  write your Supered environment. Tools are feature-flagged per team.
- **Guides environment setup** — collections, content, guides, page triggers, process boards,
  sync-engine rules/rulesets, and action plans — with Supered best practices baked in.

## Install

```
/plugin marketplace add getsupered/supered-plugin
/plugin install supered@supered
```

Then **activate and sign in**:

1. Restart Claude Code so the plugin's MCP server loads. (On recent versions you can run
   `/reload-plugins` instead of restarting; if it reports "Unknown slash command," your Claude
   Code is older than that feature — just restart, or run `claude update` first.)
2. Run `/mcp`, select **Supered**, and choose **Authenticate** to complete the OAuth sign-in in
   your browser. (Shell equivalent: `claude mcp login supered`.)
3. Confirm with the `whoami` tool. Tokens are handled by Claude Code — you never paste them into chat.

Manual MCP connection (other clients, or without the plugin):

```
claude mcp add --transport http Supered https://app.supered.io/mcp
```

## Skills

| Skill | Purpose |
| --- | --- |
| `supered:setup-supered` | Orchestrator / entry point. Detects state and routes to the others. |
| `supered:connect-supered-mcp` | Connect & verify the Supered MCP (OAuth). |
| `supered:connect-integrations` | Connect the CRM (HubSpot/Salesforce) and other MCPs. |
| `supered:build-content-and-guides` | Collections, cards, folders, guides, page triggers. |
| `supered:build-rules-and-boards` | Sync-engine rules/rulesets and process boards (needs CRM). |
| `supered:build-action-plans` | Phases, sections, tasks, roles, assignments, templates. |

Start with: `/supered:setup-supered`

## Layout

```
supered-plugin/
├── .claude-plugin/
│   ├── plugin.json          # plugin manifest
│   └── marketplace.json     # marketplace catalog (this repo hosts one plugin)
├── .mcp.json                # Supered MCP server (http + OAuth)
├── skills/
│   ├── setup-supered/SKILL.md
│   ├── connect-supered-mcp/SKILL.md
│   ├── connect-integrations/SKILL.md
│   ├── build-content-and-guides/SKILL.md
│   ├── build-rules-and-boards/SKILL.md
│   └── build-action-plans/SKILL.md
└── README.md
```

## Local development

Test without installing from a marketplace:

```
claude --plugin-dir ./supered-plugin
```

Then run `/reload-plugins` after edits, and validate before publishing:

```
claude plugin validate ./supered-plugin
```

## Distribution & updates

- Bump `version` in `.claude-plugin/plugin.json` on each release so installed users update only
  on real releases (omit it and every commit becomes a new version via the git SHA).
- Enable auto-update on the marketplace so improvements to the skills propagate to users.
- To list publicly, submit via Anthropic's community-marketplace form (requires a Team/Enterprise
  org with directory-management access).

## Single source of truth for the in-app assistant

Because this repo is public (required for a marketplace), the Supered app can treat these skill
files as the canonical best-practice content: a GenServer fetches them from the public repo on
init and every X minutes (with a last-known-good fallback), so the in-app Supered Assistant and
external Claude share one set of instructions and never drift. Edit the guidance here once.
