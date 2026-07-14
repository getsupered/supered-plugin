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
  values. Note: `data_audit_logic` / `total_logic` may be stripped from the sanitized payload —
  don't expect or ask for rule DSL from this data; boards created here group by audit result,
  not by re-deriving the underlying rule logic.
- **Function tags** (`data_audit_point_definition.tags`) — categories like `Data Hygiene`,
  `Sales`. Used to filter which audit points become rules.
- **Object type** (`data_audit_point_definition.object_display_name`) — e.g. `Contacts`,
  `Companies`. One board + one ruleset per object type included.
- **Rule name** (`data_audit_point_definition.label`) — becomes the rule's name, exactly as
  written.

## Retrieve the audit

1. If the user names or links a specific audit, or gives an ID, fetch it with `data_audit`.
2. Otherwise, list audits with `data_audits` and ask the user which one to use — show name,
   domain, and provider for each, don't guess. Paginate if needed; don't conclude "none exist"
   from a single partial page.
3. If no audit exists yet and the user wants one run, that's a separate write action — confirm
   with the user before calling `data_audit_create` (or `data_audit_perform` on an existing
   audit id) — then re-fetch with `data_audit` before continuing.
4. `data_audit_update` is for editing an existing audit's own fields (e.g. name) — not part of
   the board-building flow, only use if the user explicitly asks to rename/edit the audit itself.
5. If the retrieved audit is undefined/empty, throw an error — do not proceed with placeholder data.

## Before creating anything

1. Silently check for existing boards/rulesets/rules tagged `Data Audit`. Only speak up if
   duplicates are found (warn in one line, ask to proceed, stop if declined) — if none exist,
   say nothing about having checked.
2. If any object type in the audit isn't supported by process rules/boards, state that in one
   line (which types are excluded and why) — don't elaborate further.
3. Ask which function tags AND which object types to include in a single message, using
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
4. Show the plan as a bulleted list, every rule on its own bullet line — never comma-separated,
   never grouped, regardless of list length. Do NOT put a rule count next to each object type's
   header; only the grand total at the end is a number. List first, count never:
   - `Data Audit: [object_display_name]`
     - `[label]`
     - `[label]`
   - `Data Audit: [object_display_name]`
     - `[label]`
   Then one totals line, counted from what was actually listed above:
   `1 composite board, N boards, N rulesets, N rules will be created and tagged Data Audit.`
   Ask for confirmation once.

## Naming & tagging

- Prefix everything `Data Audit: `.
- Board/ruleset per object type: `Data Audit: [object_display_name]`.
- Rule: `Data Audit: [object_display_name]: [label]`.
- Ensure a `Data Audit` tag exists (create via MCP if not) before creating anything else.
  Creating an item does not auto-tag it — after creating each rule, ruleset, and board, make
  a separate call applying the `Data Audit` tag to it.

## Workflow (after confirmation)

Create in this order — rules first as flat batches, then structure, then wire them together.
Creating many rules directly into rulesets/boards in one pass is unreliable; creating the rules
first in small batches and organizing them after is more resilient to partial failures.

1. Ensure the `Data Audit` tag exists; create it if missing.
2. Create every selected rule in batches of 5 at a time, unattached to any ruleset, tagging
   each rule `Data Audit` as it's created within that same batch. Wait for each batch to fully
   complete before starting the next. Do this across all object types before moving to step 3
   — don't interleave rule creation with ruleset/board creation.
3. Once all rules exist, create one ruleset per selected object type (`Data Audit:
   [object_display_name]`), tag it `Data Audit`.
4. Add each object type's already-created rules into its matching ruleset, in batches of 5
   at a time, waiting for each batch to complete before the next.
5. Create one process board per object type, tag it `Data Audit`, and link it to its ruleset
   via a process board condition.
6. Create the composite process board grouping all the object-type boards, tag it `Data Audit`.
7. Confirm completion with the same totals line, worded as "created and tagged."

## Guardrails

- Create a rule for every data audit point in the selected scope.
- Do not ask about specific pipelines for deals or tickets.
- Only use properties present in the data audit's own results — never invent object types,
  tags, or rule names not present in the retrieved audit data.
- Only one approval gate: the plan + totals in step 4 of "Before creating anything." Proceed
  through all creation without pausing, and without asking again — batching is an internal
  execution detail, not a new approval point.
- Never create or attach more than 5 rules in a single batch/call, in any step.
- If any individual creation call fails partway through, don't restart from scratch — report
  what succeeded vs. failed at the end and offer to retry only the failed items.

## Communication style

- Never narrate internal process: don't say what you're about to fetch, don't describe your
  plan for calling tools, don't say "let me check X" / "now let me Y" before any tool call,
  don't announce each batch ("now creating batch 2 of 3"), don't explain how many approval
  steps there will be or what each one will cover. Do all retrieval, checking, and batching
  silently; speak only at the two points defined above (unsupported-types note, and the
  plan/confirmation).
- Never show self-correction, recounting, or arithmetic in the response, in any form (e.g.
  "wait, that's 15, let me recount", "actually that's..."). If a number and a list would ever
  disagree, this means the number was written before the list — don't write per-section counts
  at all, only the single final total, and derive that total by counting the finished list, not
  by predicting it in advance.
- Don't editorialize about the plan ("Good news...", "Before I fire off...", "Let me..."). State
  facts only.
- Every rule, in every object type's section, gets its own bullet line — never comma-separated,
  never grouped onto one line, regardless of how many rules that object type has.
- Keep everything else out of the response — no restating the workflow, no previewing future
  steps, no meta-commentary on the process itself.
