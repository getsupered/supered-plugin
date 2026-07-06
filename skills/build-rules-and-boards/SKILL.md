---
name: build-rules-and-boards
description: >
  Create and audit Supered sync-engine rules and rulesets, and the process boards / composite
  boards that track CRM records against them. Use when the user wants to "flag records", "score
  deals", "find records missing X", "build a process board", "set up a ruleset", audit existing
  rules, or track CRM data quality. Requires a connected CRM (see connect-integrations).
---

# Build rules & boards

> Argument shapes differ between clients (the raw MCP flattens nested params, e.g.
> `pagination__page_size`, while some hosts present them nested like
> `pagination: {page_size: …}`). Treat the guidance below as **concepts** and map them to
> whatever shape your tool schema actually exposes — don't hardcode one form.

## Concepts

- **Rule** (`sync_engine_rule_create` / `_update`) — a DSL condition that flags, scores, or
  categorizes CRM records. Has a `type` (`record_error`, `record_warning`, `record_success`,
  `record_info`) and `field_configurations` that drive board columns.
- **Ruleset** (`sync_engine_ruleset_create` / `_update`, `_add_rules`, `_remove_rules`) — a named
  group of rules with optional **entry**/**exit** logic (DSL) that scopes which records apply.
- **Process board** (`process_board_create` / `_update`) — a Kanban view of CRM records matching
  linked rules/rulesets. Provider and record type are fixed at creation.
- **Composite process board** (`composite_process_board_*`) — groups several boards into one view.
- **Conditions** (`process_board_condition_create` / `_delete`) — link a board to rules/rulesets/tags.
- **Streaks / analytics** — see the **analyze-boards** skill.

## Before writing any rule

1. Confirm the CRM is connected (`whoami`); if not, run **connect-integrations**.
2. Load the **`dsl_specification`** prompt for exact syntax. **Do not guess DSL.** If a create
   fails, re-read the spec rather than retrying a different guess.
3. Call `list_provider_record_types`, then `list_provider_record_fields` for the target record
   type. Use exact field `name` values (never the human label) in DSL, and option `value` (not
   label) for enum comparisons.

## Audit before you propose (don't create duplicates)

For broad asks ("help me build a hygiene ruleset", "what rules should I add for deals?"), first
list what already exists with `sync_engine_rules`:

- Scope with the human-readable `record_type_label` filter (e.g. `["Deal"]`, `["Ticket"]`,
  `["Company"]`) — it's the most forgiving. **Do not** use a bare `provider_record_type` like
  `"0-3"`; that filter needs the combined `"<provider>::<record_type>"` form (e.g.
  `"hubspot::0-3"`), and a malformed value silently returns zero matches.
- **Paginate properly.** The default page size is small (~10) and most teams have dozens of
  rules. Request a large page size, turn on the total count, read that count, and page through
  until you've seen everything. Never conclude "you have no X rules" from one partial page — if
  you got zero where the user clearly has some, you either used the wrong filter or didn't
  paginate far enough. Rulesets come back in the same list as `sync_engine_ruleset` entries.
- If a rule with the same intent already exists, surface it and ask whether to adjust/replace
  rather than duplicating. Only propose the gaps. Show each existing rule's name + a one-line
  summary, and don't propose additions until the user confirms.

## Pipeline scoping for deals & tickets (critical)

HubSpot deals (`0-3`) and tickets (`0-5`) carry a `pipeline` field. A rule/ruleset that omits it
runs across **every** pipeline — almost always noisy or wrong (a sales-pipeline rule firing on
CS deals, etc.).

1. **Ask first.** Before constructing anything, ask which pipeline(s) to scope to. Surface the
   `pipeline` enum values from `list_provider_record_fields` so the user picks by label.
2. **Scope at the ruleset.** For a set of related rules, put the pipeline filter in the ruleset's
   `entry_logic` (using the pipeline option's `value`, not its label) so every rule inherits it,
   rather than repeating it per rule.
3. **Explicit opt-out only.** Only ship account-wide deal/ticket rules if the user says so
   outright. The silent default (unbounded) is the wrong default.

## field_configurations (board columns)

Always include `field_configurations` when creating/updating a rule — without it, linked boards
show only record name and owner, no field columns. For each CRM field the rule's DSL references,
add an entry with the field's exact `name` and `label` (from `list_provider_record_fields`), the
same `provider` and `record_type` as the rule, and highlight visibility for zero-state. Reuse the
field metadata you already fetched.

## Naming (product voice)

For `title` (boards, composite boards), `name` (rulesets, rules), and `display_title` (rules),
lead with one domain-relevant emoji + a space, then the title — evoking what the thing checks,
not generic decoration. Examples: `🏢 Company Hygiene`, `🧹 Core Data Completeness`,
`🕸️ No Activity in 90+ Days`, `📞 Missing Phone Number`, `🩺 Pipeline Health`. One emoji, at the
start only; skip it if nothing fits (better plain than forced); never add emojis to descriptions,
DSL text, or field labels.

## Workflow

1. Verify CRM + load DSL spec + fetch fields (above).
2. Audit existing rules/rulesets; propose only the gaps and confirm.
3. Create/choose a ruleset; set `entry_logic` for pipeline scope.
4. Create rules with real field names + `field_configurations`.
5. Create the board and link it via `process_board_condition_create`.
6. Show the board and confirm records populate as expected.

## Guardrails

- Never create unscoped deal/ticket rules without confirming the pipeline.
- Confirm each rule's intent before creating; propose the outline first for anything non-trivial.
- Prefer `*_update` and enable/disable (`disabled_at`) over deleting and recreating.
