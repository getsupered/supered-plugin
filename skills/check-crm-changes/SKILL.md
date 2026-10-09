---
name: check-crm-changes
description: >
  Check CRM records against Supered process rules before and after changing them, and fix
  flagged records. Use when you are about to create or update HubSpot, Salesforce, or Pipedrive
  records with another tool, when the user asks "what would happen if I change X", "will this
  update break any rules", "fix this flagged record", "resolve the records on this board", "clear
  this deal off the board", or wants bulk CRM edits checked against their process rules. Requires
  the Supered MCP and a connected CRM.
metadata:
  surfaces: mcp
---

# Check CRM changes against process rules

`sync_engine_rule_check` evaluates CRM records against the team's process rules using live CRM
data. It never writes anything. Use it as the guardrail around any CRM write you make with another
tool (a HubSpot, Salesforce, or Pipedrive MCP, or an API).

> Argument shapes differ between clients (the raw MCP flattens nested params, e.g.
> `scope__process_board_ids`, while some hosts present them nested like
> `scope: {process_board_ids: …}`). Map the guidance below to the shape your tool exposes.

## Inputs

- `provider` and `record_type`: from `whoami` and `list_provider_record_types` (HubSpot uses ids
  like `0-3` for deals, Salesforce uses API names like `Opportunity`).
- `record_ids`: CRM record ids, at most 20 per call. Batch larger sets.
- `scope` (optional): limit to the rules on `process_board_ids`, a `composite_process_board_id`,
  `ruleset_ids`, or `rule_ids`. Scope to the board when the user is working from one. Omit it to
  check every active rule for the record type.
- `proposed_changes` (optional): JSON object of record id to the values you intend to write, e.g.
  `{"123": {"dealtype": "newbusiness", "closedate": "2026-12-31T00:00:00Z"}}`. Use API field names
  and enum option values from `provider_record_detail`, ISO dates, `null` to clear, and a list for
  multi-select. Associated records can't be changed this way.

## Before and after every CRM write

1. **Check first.** Call `sync_engine_rule_check` with `proposed_changes`. Base the plan on this
   check, not on what you read earlier in the conversation: CRM automation and other users can
   change records at any time.
2. **Read the result per record.**
   - `triggered_rules`: what the record triggers now. If a rule you meant to fix is already gone,
     don't make the change; say so.
   - `newly_triggered_rules`: rules the change would start triggering. Don't write changes that add
     `record_error` or `record_warning` rules without the user's OK. `record_success` and
     `record_info` rules triggering is usually fine.
   - `no_longer_triggered_rules`: rules the change resolves. Mention them; they're the payoff. If a
     record's change resolves nothing, say so and ask before writing it, even if the user asked to
     change "each one".
   - `unknown_fields`: names that don't exist on the record type. Fix the typo and check again.
   - `read_only_fields`: fields the CRM calculates. Writing them will fail; see **Calculated
     fields**.
3. **Confirm with the user.** Show each record's changes alongside the rules they start and stop
   triggering, and get approval before writing.
   - A direct instruction that names the records and the values ("mark these 5 as Disqualified, Not
     a good fit") approves exactly that change.
   - Anything you fill in yourself is not approved until the user sees it: email or note subjects and
     bodies, activity timestamps, values you derived or chose, and records the user didn't name.
     Show them, then write.
   - If the CRM tool requires its own confirmation (a change table, a confirmation status), follow
     it. Fold the rule impact into that step rather than asking twice.
4. **Write** with the CRM tool, and only to the records and fields you checked. A record you add
   later needs its own check.
5. **Check again** without `proposed_changes` and compare with the prediction. Report any
   difference instead of assuming the write worked. CRM automation and calculated fields can lag
   behind a write (for example, HubSpot moving a lead's stage after an email is logged). Never
   report that something didn't change based on a read taken right after a write; wait briefly and
   read again first.
6. **Offer a board sync.** Boards pick up CRM changes on their next poll. To show the result right
   away, offer `process_board_sync_create` for the affected board. A sync re-reads every record on
   the board, so for a few edits on a large board, say it can take a minute or two.

## Fixing a flagged record

1. Call `sync_engine_rule_check` without `proposed_changes`. `audit` lists, for each triggered rule,
   the criteria with `field`, `operator`, `expected_value`, and `matched`. Supered never returns CRM
   field values, so read the record's current values for those fields from the CRM.
2. Work out the change that flips each matched criterion. Ask the user for any value you can't
   derive from the record or the conversation. Never invent amounts, dates, or other business data
   to clear a flag.
3. Run the before-and-after loop above with those changes.

Some criteria come from the rule's ruleset (its entry and exit logic) rather than the rule itself.
Changing those fields can move the record out of the ruleset's scope instead of fixing it. Call that
out to the user.

## Working through a board

When the user wants to resolve the records on a board:

1. **Confirm the scope.** A board link with both a composite id and a `board_id` means the user is
   looking at one member board of a composite. Ask whether they mean that board or the whole
   composite before you start.
2. Call `process_board` for each board's `provider`, `record_type`, and `primary_field`
   (`composite_process_board` lists a composite's member boards). Member boards can track
   different record types.
3. Call `process_board_records` for each record's `process_board_id`, `record_id`, and the rules
   it triggers. Pass `process_board_id` for one board, `composite_process_board_id` alone for every
   member board, or both for one member board seen through the composite. Page through with a page
   size of 100; use `rule_ids` to focus on one rule.
4. Supered returns only identifiers. To show the user which record is which, read each record's
   `primary_field` (and anything else you need) from the CRM, then refer to records by name.
5. Fix them with the steps above, then sync the board. To confirm it's clear, call
   `process_board_records` or `process_board_analytics` again after the sync completes. Before
   finishing, mention any boards in the scope you haven't worked through.

## Calculated fields

A criterion with `field_read_only: true` is on a field the CRM calculates, so you resolve it with
the action that drives the value:

| Field (HubSpot) | Action that changes it |
| --- | --- |
| `num_associated_contacts`, `num_associated_deals`, other `num_associated_*` | Associate (or remove) the related record |
| `notes_last_updated`, `hs_last_sales_activity_timestamp`, `num_contacted_notes` | Log a note, call, email, or meeting on the record |
| `hs_lastmodifieddate`, `createdate` | Not actionable; flag rules on these to the user |

On Salesforce, formula and roll-up fields work the same way: change the source fields or child
records the formula reads.

You can still put the expected calculated value in `proposed_changes` (e.g.
`"num_associated_contacts": 1`) to predict whether the action clears the rule. Calculated values
can take up to a minute to update after the action, so if the check right after still shows the
rule, wait briefly and check once more before reporting it unresolved.

## Guardrails

- The check is read-only. It doesn't need approval, but every CRM write does.
- Check before writing even when the user didn't ask, for any CRM create or update you make.
- Batch at most 20 records per call, and keep the CRM tool's own batch limits.
- Association criteria (`type: "association"`) can't be predicted with `proposed_changes`. Make the
  association, then check again.
