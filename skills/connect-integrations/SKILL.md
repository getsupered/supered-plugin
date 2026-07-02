---
name: connect-integrations
description: >
  Connect Supered to third-party tools during setup — primarily the CRM (HubSpot or Salesforce)
  that process boards and sync-engine rules depend on, and other MCPs (e.g. Slack) the user
  wants alongside Supered. Use when the user asks to "connect HubSpot", "connect Salesforce",
  "link my CRM", "why are my boards empty", or wants to wire up other tools during onboarding.
---

# Connect integrations

## CRM (HubSpot / Salesforce) — required for boards & rules

Supered's process boards and sync-engine rules operate on CRM records, so a CRM connection is a
prerequisite for those features.

1. Check current status: call `whoami` and read `team.hubspot_connection` and
   `team.salesforce_connection`. A non-null value with an `id` means it's connected.
2. **The CRM OAuth connection is made in the Supered UI, not over the API.** If it's missing,
   direct the user to connect it in Supered (Settings → integrations), then re-run `whoami` to
   confirm. Do not attempt to create or modify the connection yourself.
3. Once connected, confirm what's reachable:
   - `list_provider_record_types` — the CRM object types available (e.g. HubSpot contacts/deals,
     Salesforce Contact/Opportunity).
   - `list_provider_record_fields` — field metadata you'll need when writing rule DSL and
     configuring board columns.

After the CRM is connected, hand off to **build-rules-and-boards**.

## Other MCPs (Slack, etc.)

Supered's plugin coordinates onboarding; it does not reimplement other tools. If the user wants
Claude connected to another service alongside Supered:

- Point them to that tool's own Claude plugin / MCP server and let Claude Code's MCP flow handle
  auth, rather than trying to proxy it through Supered.
- Keep Supered's role scoped to Supered objects; reference the external tool by name in guidance
  (e.g. "post this board summary to Slack") once its MCP is connected separately.

## Guardrails

- Never enter CRM or third-party credentials on the user's behalf — connections are authorized
  by the user in the respective product's own UI/OAuth flow.
- Don't modify integration settings, webhooks, or sharing/permission scopes; surface what needs
  changing and let the user do it.
