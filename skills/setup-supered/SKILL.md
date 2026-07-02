---
name: setup-supered
description: >
  Entry point for setting up or onboarding a Supered environment with Claude. Use when a
  user is new to Supered, wants to "set up Supered", "onboard", "get started", connect the
  Supered MCP, or build out their first collections, content, guides, page triggers, process
  boards, rules, or action plans. This skill orchestrates the others (connect-supered-mcp,
  connect-integrations, build-content-and-guides, build-rules-and-boards, build-action-plans).
---

# Set up Supered

You are helping the user stand up or extend a Supered environment. Work **incrementally and
confirm as you go** — never create a batch of objects without showing the user what you intend
to create first.

## Step 1 — Confirm the MCP is connected

Call `whoami`. If it fails or no Supered tools are available, run the **connect-supered-mcp**
skill first, then come back here.

On success, note from the response:
- `user` (name/email) and `team.name` — confirm you're pointed at the right workspace.
- `team.hubspot_connection` / `team.salesforce_connection` — whether a CRM is connected. This
  determines whether process boards and sync-engine rules are usable yet (they operate on CRM
  records). If neither is present, flag it and offer the **connect-integrations** skill.

## Step 2 — Assess the current state

Before proposing anything, get the lay of the land (use pagination — see
build-rules-and-boards for the page_size/include_count convention):
- `collections` — existing Bases and content areas.
- `sync_engine_rulesets` / `sync_engine_rules` — existing CRM logic.
- `process_boards` / `composite_process_boards` — existing tracking views.
- `action_plans` — existing plans and templates.

Summarize what already exists in 3–5 lines. A brand-new team will have little or nothing; an
existing team may just want to add one thing.

## Step 3 — Pick a path with the user

Ask what they want to accomplish first. Map their goal to the right build skill:

| Goal | Skill |
| --- | --- |
| Knowledge content, cards, folders, in-app guides | **build-content-and-guides** |
| Surface cards/guides inside HubSpot/Salesforce by URL | **build-content-and-guides** (page triggers) |
| Flag/score CRM records; Kanban process tracking | **build-rules-and-boards** |
| Client/customer onboarding or task sequences | **build-action-plans** |
| Connect HubSpot/Salesforce or other tools | **connect-integrations** |

For a true from-scratch onboarding, a good default first-run order is: connect CRM → one
Collection with a couple of cards → one page trigger → one ruleset + rule → one process board.
This mirrors Supered's own in-app onboarding checklist (extension installed, first card, first
trigger, CRM connected, first rule).

## Step 4 — Build, confirming each object

Delegate to the relevant build skill. After each object is created, show its title/id and ask
before moving to the next. Prefer `*_update` tools over recreating objects.

## Grounding your guidance in live docs

When the user asks how a Supered feature works, or you're unsure of current behavior, use
`help_center_search` and `help_center_card` to cite the live product docs rather than guessing.
Human-readable docs also live at https://help.supered.io.

## Principles

- Confirm the target team before creating anything.
- Show intent before writing; create one thing at a time for a first-time setup.
- Respect per-team entitlements — the MCP only exposes tools the team is authorized for, so if
  a tool isn't available, that capability isn't enabled for them.
- Never touch access controls or sharing — those are managed in the Supered UI.
