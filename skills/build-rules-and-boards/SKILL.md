---
name: build-rules-and-boards
description: >
  Create and audit Supered sync-engine rules and rulesets, and the process boards / composite
  boards that track CRM records against them. Use when the user wants to "flag records", "score
  deals", "find records missing X", "build a process board", "set up a ruleset", audit existing
  rules, track CRM data quality, or "sync a board" / check whether a board sync is done. Requires a
  connected CRM (see connect-integrations).
---

# Build rules & boards

> Argument shapes differ between clients (the raw MCP flattens nested params, e.g.
> `pagination__page_size`, while some hosts present them nested like
> `pagination: {page_size: …}`). Treat the guidance below as **concepts** and map them to
> whatever shape your tool schema actually exposes — don't hardcode one form.

## Concepts

- **Rule** (`sync_engine_rule_create` / `_update`) — a DSL condition that flags, scores, or
  categorizes CRM records. Has a `type` (`record_error`, `record_warning`, `record_success`,
  `record_info`, or `ai` — see **AI rules**) and `field_configurations` that drive board columns.
- **Resolution prompt** — rules of an *actionable* type (`record_error`, `record_warning`,
  `custom`) get an auto-generated AI "resolution prompt" that powers the in-app **Fix with
  Claude / Breeze** action on a flagged record. Supered writes it from the rule logic and
  regenerates it when the logic changes. `resolution_prompt` is an optional override field on
  `sync_engine_rule_create` / `_update` — see Guardrails; normally you leave it off. (**AI rules**
  are the exception — their prompt is authored by hand and never auto-generated; see below.)
- **Ruleset** (`sync_engine_ruleset_create` / `_update`, `_add_rules`, `_remove_rules`) — a named
  group of rules with optional **entry**/**exit** logic (DSL) that scopes which records apply.
- **Process board** (`process_board_create` / `_update`) — a Kanban view of CRM records matching
  linked rules/rulesets. Provider and record type are fixed at creation.
- **Composite process board** (`composite_process_board_*`) — groups several boards into one view.
- **Conditions** (`process_board_condition_create` / `_delete`) — link a board to rules/rulesets/tags.
- **Board sync** (`process_board_sync_create` / `process_board_syncs`) — re-evaluates a board's
  CRM records against its current rules. See **Syncing boards after changes**.
- **Streaks / analytics** — see the **analyze-boards** skill.

## AI rules (`type: "ai"`)

A fundamentally different type: an `ai` rule **deploys AI** against records in a desired state
rather than flagging them. It is **evaluate-only** — when a record matches, Supered surfaces an
**"Solve with AI"** button in the browser extension seeded with the rule's prompt; it never
persists a flag, and is **exempt from process boards, streaks, counts, and analytics** (it can sit
under a ruleset for organization, but stays hidden from any board that ruleset is on). So it has no
"records" to review and won't show up on a board you build.

When creating one:

- **Confirm intent first.** This deploys an AI action against every matching record — treat it as
  more consequential than a flag rule and get explicit confirmation.
- **The `resolution_prompt` IS the rule.** Author it **by hand** on create — it's the instructions
  the downstream AI runs with. It is **never auto-generated** for `ai`, so omitting it ships a dead
  button. (This inverts the normal "leave it auto-generated" guidance.)
- **Requires an AI resolution target.** The rule only surfaces when the team has a valid resolution
  target for the rule's provider (Salesforce → Agentforce/Claude, HubSpot → Breeze/Claude),
  configured in **Settings → AI**. Without it the rule is created but never appears. If you can't
  confirm one is set, tell the user it's a prerequisite.
- **Skip `field_configurations`** — they drive board columns, which AI rules never have.

## Before writing any rule

1. Confirm the CRM is connected (`whoami`); if not, run **connect-integrations**.
2. Load the **`dsl_specification`** prompt for exact syntax. **Do not guess DSL.** If a create
   fails, re-read the spec rather than retrying a different guess.
3. Call `list_provider_record_types`, then `provider_record_detail` for the target record
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
   `pipeline` enum values from `provider_record_detail` so the user picks by label.
2. **Scope at the ruleset.** For a set of related rules, put the pipeline filter in the ruleset's
   `entry_logic` (using the pipeline option's `value`, not its label) so every rule inherits it,
   rather than repeating it per rule.
3. **Explicit opt-out only.** Only ship account-wide deal/ticket rules if the user says so
   outright. The silent default (unbounded) is the wrong default.

## field_configurations (board columns)

Always include `field_configurations` when creating/updating a rule — without it, linked boards
show only record name and owner, no field columns. For each CRM field the rule's DSL references,
add an entry with the field's exact `name` and `label` (from `provider_record_detail`), the
same `provider` and `record_type` as the rule, and highlight visibility for zero-state. Reuse the
field metadata you already fetched.

## Naming (product voice)

For `title` (boards, composite boards), `name` (rulesets, rules), and `display_title` (rules),
lead with one domain-relevant emoji + a space, then the title — evoking what the thing checks,
not generic decoration. Examples: `🏢 Company Hygiene`, `🧹 Core Data Completeness`,
`🕸️ No Activity in 90+ Days`, `📞 Missing Phone Number`, `🩺 Pipeline Health`. One emoji, at the
start only; skip it if nothing fits (better plain than forced); never add emojis to descriptions,
DSL text, or field labels.

## Syncing boards after changes

CRM records are evaluated against a rule when they change in the CRM, or when a board syncs.
`process_board_create` starts a sync on its own. After these changes, affected boards are
incomplete until a sync runs, because records that haven't changed since were never evaluated
against the new logic:

- creating or editing rules, or changing a ruleset's rules or entry/exit logic
- linking rules, rulesets, or tags to an existing board (`process_board_condition_create`). This
  only copies in records already known to trigger them.

**When to offer.** When a task that made one of these changes is done, offer to sync the affected
boards. Affected boards are the ones you created or linked, plus any board whose
`process_board_conditions` reference a rule or ruleset you changed (check `process_boards`). Name
the boards in the offer, e.g. "Want me to sync 🩺 Pipeline Health and 🏢 Company Hygiene so their
records reflect the new rules?" Skip the offer when the only change was `process_board_create`
with its conditions passed inline, since that board is already syncing.

**Ask first.** A sync pulls every record on the board from the CRM and can take minutes on large
boards. Don't start one without a yes. If your mutations go through an approval prompt (the
in-app assistant), proposing the `process_board_sync_create` call is the ask; don't ask in text
first as well.

**Starting it.** Call `process_board_sync_create` once for all affected boards:

- Boards that make up one composite: pass `composite_process_board_id`. This syncs every member
  board using the composite's owner settings.
- Otherwise: pass `process_board_ids`.

The user must have access to every board in the call, including each member of a composite. If one
is inaccessible the call fails and nothing starts; tell the user which board, and offer to sync the
rest with `process_board_ids`.

A board has at most one active sync. A sync that is still `ENQUEUED` for the same request (same
composite, or no composite) is returned as-is. Any other active sync is cancelled and restarted, so
it picks up the latest rules and the right owner settings. Don't call it again for a board that is
already syncing unless rules changed since.

**Showing progress.** If you have a `process_board_sync_watch` action (the in-app Supered
Assistant), call it with every returned sync `id` right after each `process_board_sync_create`. It
shows a live progress card under your reply, so don't poll `process_board_syncs` to report
progress.

**Reporting status.** Keep the returned sync `id`s. When the user asks how it's going, or before
you report results that depend on the new rules, call `process_board_syncs` with those `ids`.
Report per board: status, `percent_complete`, and once finished, `target_record_count` evaluated
and `result_record_count` flagged. `COMPLETED` means the board is current. On `FAILED`, offer to
start it again. On `CANCELLED`, first check whether a newer sync replaced it (`process_board_syncs`
with `process_board_ids`, newest first). If not, check the CRM connection (`whoami`): a sync is
also cancelled when the board's provider isn't connected. Don't poll in a loop; check when asked or
at a natural point in the conversation.

## Workflow

1. Verify CRM + load DSL spec + fetch fields (above).
2. Audit existing rules/rulesets; propose only the gaps and confirm.
3. Create/choose a ruleset; set `entry_logic` for pipeline scope.
4. Create rules with real field names + `field_configurations`.
5. Create the board and link it via `process_board_condition_create`.
6. Offer to sync the affected boards (see **Syncing boards after changes**), then report the sync
   status and confirm records populate as expected.

## Guardrails

- Never create unscoped deal/ticket rules without confirming the pipeline.
- Confirm each rule's intent before creating; propose the outline first for anything non-trivial.
- Prefer `*_update` and enable/disable (`disabled_at`) over deleting and recreating.
- **Leave `resolution_prompt` auto-generated** for `record_*` / `custom` rules. Omit it on
  create/update — Supered writes it from the rule logic. Only pass a value when the user
  *explicitly* asks to customize the fix instructions; doing so marks the rule "manually
  customized," after which automatic updates make only minimal edits to keep it correct. **AI rules
  are the opposite** — always author their `resolution_prompt` by hand; it is never auto-generated.
