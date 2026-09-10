---
name: manage-announcements
description: >
  Create, edit, schedule, and stop Supered Announcements (aka Updates) — broadcast messages
  that surface to the team in the Supered extension as a modal or banner. Use when the user
  wants to "send an announcement", "post an update to the team", "tell everyone about X",
  "announce a process change", schedule or retarget an announcement, or stop one that's live.
---

# Manage announcements

## Concepts

- **Announcement** (surfaced in-app as an **Update**) — a message broadcast to users on the
  team. Read one with `announcement`; list/search with `announcements`; create with
  `announcement_create`; edit/reschedule/stop with `announcement_update`.
- **Surface (`ui_type`)** — `modal` (a dialog the user must dismiss; the default) or `banner`
  (a less intrusive strip shown before the modal). Use `banner` for low-urgency notes, `modal`
  when you want the user to actually read it.
- **Audience (`assigned_to_type`)** — `all` (every licensed user) or `targeted` (specific
  `assigned_user_ids` / `assigned_group_ids`). The id fields are only used when targeted.
- **Body** — authored as `markdown` (same subset as content cards). When reading back to edit,
  use `markdown_body` (canonical for agents) or `plain_text_body` to keep context short.
- **Link** — attach either a Card via `linked_content_id` **or** an external `linked_url`
  (mutually exclusive), not both.
- **In-app URL** — an announcement's own page is `app_url` (full, login-required) or `path`
  (`/updates/<id>`). Always use these fields when linking a user to an announcement; never hand-
  build the URL (the route is `/updates/<id>`, **not** `/announcements/<id>`).

## Scheduling & lifecycle

- **`assignment_begins_at`** — ISO8601 launch time; defaults to now. An announcement only
  reaches users once this time has passed, so leave it blank to send immediately or set a future
  time to schedule.
- **`assignment_cutoff_days`** — **required on create**, must be a positive number. Stops
  assigning the announcement this many days after launch. Open-ended announcements are not
  allowed through this API — pick a sensible window (e.g. 7, 14, 30) and confirm it.
- **`stopped_at`** — set (to now or an ISO8601 time) via `announcement_update` to stop showing
  it to new users. This is the "take it down" action; the announcement is not deleted.

## Workflow

1. **Before creating**, confirm with the user: title, body, surface (`modal`/`banner`),
   audience (all vs. targeted — and resolve `team_members` / groups to ids if targeted), launch
   time, and cutoff days. Show the drafted markdown and these settings for approval first.
2. Create with `announcement_create` (`title` + `markdown` + `assignment_cutoff_days` are
   required). Report back the new title and `app_url`.
3. **To change a live announcement, edit it — don't recreate.** Fetch `markdown_body`, edit the
   full string, and send it back through `announcement_update.markdown` (it replaces the whole
   body; there is no partial patch, and an empty string is rejected).
4. To retarget, reschedule, or stop, use `announcement_update` with only the fields that change
   (`assigned_to_type`/ids, `assignment_begins_at`, `stopped_at`).
5. When the user asks for "the link", return `app_url` from the tool response.

## Finding an announcement to edit

Use `announcements` to locate the id first. Filter with `search` (title/body), `tags`,
`ui_type`, `assigned_to_type`, or the `launched_*` / `stopped_*` date bounds. Pagination is
opt-in — pass a larger `pagination__page_size` (e.g. 50) with `pagination__include_count` when
scanning the full list.

## Guardrails

- Creating and editing announcements is a **broadcast to real users** — always confirm content
  and audience before sending, and prefer a future `assignment_begins_at` if the user is still
  reviewing.
- Requires the `updates_manage` permission; if the create/update tools aren't available, the
  team or user isn't entitled to manage announcements — say so rather than retrying.
- Prefer `announcement_update` over stop-and-recreate. To take one down, set `stopped_at`, don't
  ask the user to delete it.
- Never hand-construct the announcement URL — read `app_url` / `path` from the response.
