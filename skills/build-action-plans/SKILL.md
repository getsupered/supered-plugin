---
name: build-action-plans
description: >
  Create and structure Supered action plans — phases, sections, tasks, roles, and assignments —
  and work with templates and instances. Use when the user wants to "build an action plan",
  "create an onboarding plan", "set up a task sequence", "make a template", or "assign a plan to
  a client".
---

# Build action plans

## Concepts

- **Action plan** (`action_plan_create` / `action_plan_update`) — a structured sequence of work.
  A plan is either a **template** (reusable) or an **instance** (assigned/live). Convert between
  them with `action_plan_convert_to_template`, and spin up a live copy from a template with
  `action_plan_create_instance_from_template`.
- **Phase** (`action_plan_phase_create` / `_update`) — top-level stage within a plan.
- **Section** (`action_plan_section_create` / `_update`) — grouping within a phase.
- **Task** (`action_plan_task_create` / `_update`) — the work item; can be nested under a parent
  task, has due dates, and a status (`in_progress` / `completed`).
- **Roles** (`action_plan_roles`) — named participant types (e.g. "CSM", "Account Executive").
- **Assignments** (`action_plan_set_assignments`) — link a plan to users/emails/domains with
  permission levels.
- List with `action_plans`; see plans assigned to the current user with `assigned_action_plans`.

## Ordering

Phases, sections, and tasks use a string-encoded decimal `order` sort key. When inserting
between two items, pick a value between their keys rather than renumbering everything.

## Dates

- **Templates** use `due_at_duration` — an integer number of days from plan start.
- **Instances** use a concrete `due_at` ISO 8601 timestamp.
Use the right one for the plan type you're building.

## Workflow

1. Decide template vs instance with the user. For anything reused across clients, build a
   template, then instantiate per client with `action_plan_create_instance_from_template`.
2. Create the plan (`action_plan_create`), then its phases, then sections, then tasks — top
   down. Confirm the outline with the user before filling in all tasks.
3. Define roles (`action_plan_roles`) if the plan has distinct participant types.
4. Assign with `action_plan_set_assignments` **only after confirming the recipients** with the
   user (see guardrail).

## Guardrails

- Assigning a plan surfaces it to those people — treat `action_plan_set_assignments` as a
  send-like action and confirm the exact recipients and permission levels with the user first.
- Prefer `*_update` (including marking tasks complete via task `status`) over recreating items.
- Edits are last-write-wins; re-fetch a plan's body before overwriting if others may be editing.
