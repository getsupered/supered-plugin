---
name: convert-data-audit-to-composite-board
description: >
  Fetch a DataAudit report via MCP and turn it into a Supered Composite Process Board containing process boards, rulesets, and rules
  (HubSpot/CRM completeness & hygiene scoring). Use when the user wants to "turn this audit into
  boards", "build rules from the data audit", "track data hygiene", or references a data audit by
  name/domain/ID. Requires a connected CRM (see connect-integrations).
---

# Convert Data Audit to Composite Board

## Concepts

- **DataAudit** — a report of `results`, each with a `data_audit_point_definition` (the audit
  check: `label`, `object_display_name`, `tags`) and a score. This is the *only* source of
  object types, function tags, and rule names — never invent, rename, or reformat any of these
  values.
- **Function tags** (`data_audit_point_definition.tags`) — categories like `Data Hygiene`,
  `Sales`. Used to filter which audit points become rules. These are audit categories, not
  Supered tags; never apply them to rules.
- **Object type** (`data_audit_point_definition.object_display_name`) — e.g. `Contacts`,
  `Companies`. One board + one ruleset per object type included.
- **Rule name** (`data_audit_point_definition.label`) — becomes the rule's name, exactly as
  written.
- **`Data Audit` tag** — the Supered tag this skill puts on every rule and ruleset it creates.
  It's how later runs find what already exists, even if a run stopped partway or rules were
  renamed or moved. Boards and composite boards can't be tagged.

## Retrieve the audit

1. If the user names or links a specific audit, or gives an ID, fetch it with `data_audit`.
2. Otherwise, list audits with `data_audits` and ask the user which one to use — show name,
   domain, and provider for each, don't guess. Paginate if needed; don't conclude "none exist"
   from a single partial page.
3. If no audit exists yet and the user wants one run, that's a separate write action. Pick an
   audit template with `audit_templates` and confirm it with the user, then call
   `audit_run_create`. To re-run an existing audit, read its `audit_template_id` with `audit_run`
   and pass that; an audit with no template can't be re-run here. Poll `audit_run` until `status`
   is `completed` or `failed`, then fetch the result with `data_audit` using the run's `id` before
   continuing.
4. `data_audit_update` is for editing an existing audit's own fields (e.g. name) — not part of
   the board-building flow, only use if the user explicitly asks to rename/edit the audit itself.
5. If the retrieved audit is undefined/empty, throw an error — do not proceed with placeholder data.

## Find what already exists

Do this silently before asking the user anything.

1. List everything tagged `Data Audit` with `sync_engine_rules`, filtering on
   `tags: ["Data Audit"]`. The results include both rules and rulesets. Page through all of them
   with a large page size and the total count on; never conclude "nothing exists" from one page.
2. **Rules.** An audit point already has a rule when a tagged rule's name is
   `Data Audit: [object_display_name]: [label]` for that point. Match on that name, not on the
   rule's ruleset, since rules can be moved.
3. **Rulesets.** An object type already has a ruleset when a tagged ruleset is named
   `Data Audit: [object_display_name]`. If more than one matches, use the one that holds the most
   of that object type's matched rules.
4. **Boards.** Find each existing ruleset's board with `process_boards`: the board whose
   `process_board_conditions` include that ruleset's id with object type `RULESET`. Boards can't be
   tagged, so never look them up by name alone.
5. **Composite.** Find the composite with `composite_process_boards`: the one whose `entries`
   include those boards. If the boards sit in different composites, use the one with the most of
   them.

Anything found here is reused, not recreated. Rules that exist under the same name but aren't
tagged `Data Audit` don't count as matches; leave them alone.

## Before creating anything

1. If any object type in the audit isn't supported by process rules/boards, state that in one
   line (which types are excluded and why) — don't elaborate further.
2. Ask which function tags AND which object types to include in a single message, using
   distinct prefixes per list so answers are never ambiguous, for example:
   - `F1. Data Hygiene`
   - `F2. Sales`
   - `F3. All`
   - `O1. Contacts`
   - `O2. Companies`
   - `O3. Deals`
   - `O4. Tickets`
   - `O5. All`
   Tell the user to answer using the prefix+number (e.g. "F3, O5" or "F1, O2"), or full names,
   or "all" for either list. Never present two lists that both start numbering at 1 without a
   distinguishing prefix. Only list object types that are actually supported.
3. If everything in the selected scope already exists, say so in one line, offer to sync the
   composite, and stop.
4. Otherwise show the plan as a bulleted list, every rule on its own bullet line — never
   comma-separated, never grouped, regardless of list length. List only what will be created;
   mark an object type whose ruleset and board already exist with `(existing)` after its name. Do
   NOT put a rule count next to each object type's header; only the totals line has numbers:
   - `Data Audit: [object_display_name]`
     - `[label]`
     - `[label]`
   - `Data Audit: [object_display_name] (existing)`
     - `[label]`
   Then one totals line, counted from what was actually listed above, naming only what will be
   created, e.g.:
   `N rules, N rulesets, N boards, and 1 composite board will be created and tagged.`
   If anything already exists, add one line: `N rules already exist and will be reused.`
   Ask for confirmation once.

## Naming

- Board/ruleset per object type: `Data Audit: [object_display_name]`.
- Rule: `Data Audit: [object_display_name]: [label]`. This name is how later runs match rules to
  audit points, so never change its format.
- Composite board: `Data Audit: [audit name]`.

## Workflow (after confirmation)

Create in this order — rules first as flat batches, then structure, then wire them together.
Creating many rules directly into rulesets/boards in one pass is unreliable; creating the rules
first in small batches and organizing them after is more resilient to partial failures. Skip
every step for objects that already exist.

1. Create the missing rules in batches of 5, unattached to any ruleset. In the same batch, right
   after each create, call `sync_engine_rule_tags_update` with `add_tag_names: ["Data Audit"]`
   on the new rule's id. Wait for each batch to fully complete before starting the next. Do this
   across all object types before moving to step 2 — don't interleave rule creation with
   ruleset/board creation.
2. Create each missing ruleset (`Data Audit: [object_display_name]`) and tag it with
   `sync_engine_ruleset_tags_update` (`add_tag_names: ["Data Audit"]`).
3. Add each object type's new rules into its ruleset, new or existing, in batches of 5, waiting
   for each batch to complete before the next.
4. Create a process board for each object type that has none and link it to its ruleset via a
   process board condition.
5. Create the composite board if none exists. Otherwise add the new boards to the existing
   composite with `composite_process_board_entry_create`.
6. Confirm completion with the same totals line, worded as "created and tagged."
7. Boards linked to rulesets after creation are incomplete until they sync. Offer to
   sync the composite with `process_board_sync_create` (`composite_process_board_id`), and
   report its status with `process_board_syncs` when asked. See **build-rules-and-boards** →
   *Syncing boards after changes*.

## Guardrails

- Create a rule for every data audit point in the selected scope that doesn't already have one.
- Tag every rule and ruleset you create `Data Audit`. An untagged rule is invisible to the next
  run and will be duplicated. If a tag call fails, retry it before moving on.
- Never tag rules or rulesets you didn't create in this run, and never remove `Data Audit` from
  anything.
- Do not ask about specific pipelines for deals or tickets.
- Only use properties present in the data audit's own results — never invent object types,
  tags, or rule names not present in the retrieved audit data.
- Only one approval gate: the plan + totals in step 4 of "Before creating anything." Proceed
  through all creation without pausing, and without asking again — batching is an internal
  execution detail, not a new approval point.
- Never create or attach more than 5 rules in a single batch/call, in any step.
- If any individual creation call fails partway through, don't restart from scratch — report
  what succeeded vs. failed at the end and offer to retry only the failed items. A re-run of this
  skill picks up where the last one stopped.

## Communication style

- Never narrate internal process: don't say what you're about to fetch, don't describe your
  plan for calling tools, don't say "let me check X" / "now let me Y" before any tool call,
  don't announce each batch ("now creating batch 2 of 3"), don't explain how many approval
  steps there will be or what each one will cover. Do all retrieval, checking, and batching
  silently; speak only at the points defined above (unsupported-types note, the
  everything-exists note, and the plan/confirmation).
- Never show self-correction, recounting, or arithmetic in the response, in any form (e.g.
  "wait, that's 15, let me recount", "actually that's..."). If a number and a list would ever
  disagree, this means the number was written before the list — don't write per-section counts
  at all, only the totals line, and derive it by counting the finished list, not by predicting
  it in advance.
- Don't editorialize about the plan ("Good news...", "Before I fire off...", "Let me..."). State
  facts only.
- Every rule, in every object type's section, gets its own bullet line — never comma-separated,
  never grouped onto one line, regardless of how many rules that object type has.
- Keep everything else out of the response — no restating the workflow, no previewing future
  steps, no meta-commentary on the process itself.
