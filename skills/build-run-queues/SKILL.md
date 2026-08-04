---
name: build-run-queues
description: >
  Create a Supered run queue — an ordered, personal worklist of CRM records for the current user
  to review one at a time, snapshotted from one or many process boards or a composite board. Use
  when the user wants to "start a run queue", "give me my flagged records to work through", "queue
  up the deals I need to fix", "work my board", or "make a to-do list from these boards". Requires
  a connected CRM and at least one process board (see build-rules-and-boards).
---

# Build run queues

> Argument shapes differ between clients (the raw MCP flattens nested params, e.g. `params__…`,
> while some hosts present them nested like `params: {…}`). Treat the guidance below as **concepts**
> and map them to whatever shape your tool schema actually exposes — don't hardcode one form.

## Concepts

- **Run queue** (`run_queue_create`) — a short-lived, ordered snapshot of CRM records for **one
  person** to work through, one record at a time. Think "here are the 40 flagged deals assigned to
  me, in order — let me clear them." The create call returns the full queue (its `items`,
  `total_count`, `total_available`, and `launch_url`), so no follow-up read is needed.
- **Creating ≠ starting.** Creating a queue just builds the list; the user still has to *launch* it
  to work the records. **How you deliver the launch depends on the tools you have** — pick the one
  that matches your surface:
  - **If you have a `run_queue_start` action** (the in-app Supered Assistant): call it with the new
    queue's id right after creating. It surfaces a "Start run queue" **button** under your reply.
    Don't paste the `launch_url` here — the button is the launch; describing it in prose is redundant.
  - **Otherwise** (MCP / external agents, which have no button): give the user the queue's
    `launch_url` — a ready-to-share link that opens the queue.

  Either way, a queue nobody launches does nothing. Launching only works for the assignee (signed in)
  and needs the Sidekick browser extension — the launch surface prompts them to install it if missing.
- **Assigned to the caller only.** A run queue is always owned by and assigned to the authenticated
  user. There is no field to create one for, or hand one to, someone else — the server derives the
  assignee from the caller. To help a teammate, they run it themselves.
- **One active queue per user.** Each user has at most one active queue. Creating a new one
  **abandons** the caller's previous active queue (its status becomes `abandoned`). Confirm before
  replacing a queue the user may still be working.
- **Snapshot, not live.** Membership and order are frozen at creation from the boards' current
  contents. A record is included when it is **currently flagged** — triggering at least one of the
  board's rules. The queue is never re-membered; `items[].cleared` and `current_index` are progress
  **caches**, not sources of truth (real status is always recomputed live from the rules).
- **Team oversight** (`run_queues`) — a **paginated** list of the team's currently **active** queues,
  one per assignee with a queue in progress. Read `assignee` for who, and `current_index` vs
  `total_count` for how far along. Abandoned/completed/expired queues are excluded. Use it for "who's
  working a queue right now?" **Requires the `monitor_run_queues` permission** — it exposes the
  whole team's activity, so it's a manager view, unlike creating your own queue (which needs no
  permission). Pass `pagination.include_count` to learn the total across pages.

## Choosing the source (assembly_type)

Every create picks one `assembly_type` and provides the matching board field:

The `assembly_type` values are enum constants — pass them **UPPERCASE** (`BOARD`, `BOARDS`,
`COMPOSITE`), exactly as your tool schema lists them; lowercase is rejected.

| assembly_type | Board field | Use when |
| --- | --- | --- |
| `BOARD` | `process_board_id` (one ID) | Working a single process board. |
| `BOARDS` | `process_board_ids` (list of IDs) | Working several boards at once, ad-hoc — **no composite needed**. |
| `COMPOSITE` | `composite_process_board_id` (one ID) | Working a saved composite board (expands to all its member boards). |

Prefer `BOARDS` for a one-off "these three boards" request; reach for `COMPOSITE` only when a saved
composite board already exists or the user wants a reusable grouping. Records that appear on more
than one of the sourced boards are **deduped** into a single item that remembers every source board.

## Finding the IDs

- Boards: `process_boards` (paginate — see the page_size/include_count convention in
  **build-rules-and-boards**). Confirm the title with the user before queuing.
- Composites: `composite_process_boards`.
- Never guess a board ID; list and match by title.

## Optional filters

Narrow the snapshot with any combination (omit to include everything):

- `user_ids` — only records owned by these **Supered** user IDs.
- `group_ids` — only records owned by users in these Supered group IDs.
- `crm_owner_ids` — only records whose **CRM-side** owner ID is in this list.
- `rule_ids` — only records currently triggering these specific rules (a subset of the board's rules).

A common ask is "just my flagged records" — resolve the caller's own Supered user id via `whoami`
and pass it as `user_ids`.

## Size

The queue is capped at 1000 items. `total_available` on the result reports how many records matched
before the cap — if it exceeds `total_count`, tell the user the queue was truncated and suggest a
tighter filter (a specific rule, owner, or a single board).

## Workflow

1. Confirm a CRM is connected (`whoami`); if not, run **connect-integrations**.
2. Identify the source board(s): list with `process_boards` / `composite_process_boards` and confirm
   titles with the user.
3. If the user already has an active queue they're mid-way through, confirm it's OK to replace it.
4. Call `run_queue_create` with the chosen `assembly_type`, the matching board field, and any filters.
5. From the create response, report: how many records (`total_count`), whether it was truncated
   (`total_available` vs `total_count`).
6. **Let the user launch it** (see "Creating ≠ starting"): call `run_queue_start` if you have it
   (in-app → a button), otherwise hand them the `launch_url` — e.g. "Your queue of 40 records is
   ready — open this to start: <launch_url>". Without this step the queue just sits there unworked.

To see who on the team currently has a queue in progress, use `run_queues` (paginated; needs
`monitor_run_queues`).

## Guardrails

- Creating a queue **abandons** the caller's current active queue — confirm first if one may be in
  progress.
- Never stop at "created." Always give the user a way to launch — the `run_queue_start` button if you
  have that action, otherwise the `launch_url`. Creating without launching leaves nothing to click.
- A run queue can only be assigned to the current user; don't imply you can build one for a teammate.
- Report `total_count` from the queue, not `total_available`, as "records to work" — and flag
  truncation when the two differ.
- Don't treat `cleared` / `current_index` as authoritative completion; they're resume/progress
  caches. Whether a record is truly resolved is recomputed live from its rules.
- To analyze a board (counts, most-triggered rules, streaks) rather than work its records, use
  **analyze-boards** instead.
