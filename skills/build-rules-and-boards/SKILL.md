---
name: build-rules-and-boards
description: >
  Create Supered sync-engine rules and rulesets, and the process boards / composite boards that
  track CRM records against them. Use when the user wants to "flag records", "score deals",
  "find records missing X", "build a process board", "set up a ruleset", or track CRM data
  quality. Requires a connected CRM (see connect-integrations).
---

# Build rules & boards

## Concepts

- **Rule** (`sync_engine_rule_create` / `_update`) — a condition in Supered's DSL that flags,
  scores, or categorizes CRM records (e.g. "deal missing amount"). Has a `type` such as
  `record_error`, `record_warning`, `record_success`, `record_info`, and optional
  `field_configurations` for board columns.
- **Ruleset** (`sync_engine_ruleset_create` / `_update`, `_add_rules`, `_remove_rules`) — a
  named group of related rules, with optional **entry** and **exit** logic (DSL) that scopes
  which records the ruleset applies to.
- **Process board** (`process_board_create` / `_update`) — a Kanban-style view of CRM records
  matching linked rules/rulesets. Provider and record type are fixed at creation.
- **Composite process board** (`composite_process_board_*`) — groups several boards into one
  dashboard.
- **Conditions** (`process_board_condition_create` / `_delete`) — link a board to rules,
  rulesets, or tags.
- **Streaks** (`process_board_streak_stats`, `board_streak_rollup`, `my_board_streaks`) — track
  consecutive clean periods for process-quality metrics.

## Before writing any rule — get the DSL spec and CRM fields

1. Load the Supered **DSL specification** prompt exposed by the MCP for exact syntax. Do not
   guess DSL — reference the spec.
2. Call `list_provider_record_types` and `list_provider_record_fields` to use real field names
   for the target CRM object.

## Best practices (ported from the Supered assistant)

**Pipeline scoping is critical for deals/tickets.** A rule written without scoping runs against
*every* pipeline, which is almost never intended. Prefer putting pipeline/stage scoping in the
**ruleset entry logic** so all rules in the set inherit it, rather than repeating it in each
rule. Before proposing deal/ticket rules, **ask the user which pipeline(s)** they mean.

**Be suggestion-driven.** For anything ambiguous (which pipeline, which stage, what counts as
"stale"), ask a clarifying question and propose the rule for confirmation before creating it —
don't create speculative rules.

**Pagination when scanning existing rules.** When listing rules to audit or find duplicates, use
`page_size` of 50+ and `include_count: true`, and iterate through all pages before concluding.
Don't report "there are no rules that…" from a single partial page.

## Workflow

1. Confirm CRM is connected (`whoami`); if not, run **connect-integrations**.
2. Confirm provider + record type and the target pipeline(s).
3. Create/choose a ruleset; set entry logic for scoping.
4. Create rules with `sync_engine_rule_create`, using DSL from the spec and real field names;
   set `field_configurations` for the columns the user wants to see.
5. Create a process board and link it via `process_board_condition_create`.
6. Show the user the board and confirm records are populating as expected.

## Guardrails

- Never create unscoped deal/ticket rules without confirming the pipeline.
- Prefer `*_update` and enable/disable (`disabled_at`) over deleting and recreating.
- Confirm each rule's intent with the user before creating it.
