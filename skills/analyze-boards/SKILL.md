---
name: analyze-boards
description: >
  Answer analytics, streak, and compliance questions about Supered process boards — record
  counts, most-triggered rules, owners with the most flagged records, trends over time, and
  clean-streak / compliance rates. Use when the user asks "how many records are on this board",
  "which rules fire most", "who has the most flagged records", "how's our compliance trending",
  or "what's my streak". Also use when a user has no streak data or their home page shows no
  boards to clear.
---

# Analyze boards

## Which tool

- **`process_board_analytics`** — questions about a single process board.
- **`process_boards_analytics`** — a composite board, or an aggregated view across several boards.
- **Streak tools** — `my_board_streaks` (the caller's own boards), `process_board_streak_stats`
  (all users on one board), `composite_process_board_streak_stats` (all users on a composite),
  `board_streak_rollup` (team compliance trend over time).

## Analytics: which field to read

| Question | Field |
| --- | --- |
| How many records are on this board now? | Latest entry in `records_rollup` |
| How has the record/violation count changed over time? | Full `records_rollup` series |
| Which rules are triggered most? | `rule_counts_rollup` — sort by most recent `value` |
| Which owners have the most flagged records? | `user_counts_rollup` — sort by most recent `value` |
| Which board in a composite has the most issues? | `counts_per_board` (`process_boards_analytics`) |

Present **names, not raw IDs**: resolve rule IDs to names via `process_board`; owner names are
already in the `name` field of `user_counts_rollup`.

## Streaks & compliance

Key fields:

- **`streak`** — current consecutive clean-period run (clean days or weeks). Zero means the board
  wasn't clean in the most recent scoreable period.
- **`clean_periods`** — periods in the window that ended with zero violations.
- **`eligible_periods`** — total scoreable periods in the window (weekdays for daily boards, ISO
  weeks for weekly). The denominator.
- **Compliance rate** = `clean_periods / eligible_periods` (e.g. 8/10 = 80%).

`board_streak_rollup` takes the board's `id` and its `board_type` (`PROCESS_BOARD` or
`COMPOSITE_PROCESS_BOARD`), and optionally `user_ids` to narrow it to specific people. It returns
a time series of `{label, value, exempt}`: `label` is the period (e.g. "May 14"), `value` is the
total violation count across the selected users for that period, and `exempt` marks periods the
team excludes from streaks. Use it to plot trends or spot when compliance improved or regressed.

**Date filters:** all four streak tools accept `filters` with optional `date_start` / `date_end`
(ISO 8601 dates) and `timezone` (IANA, defaults to UTC). With no dates, the server uses the last
14 days — state the window you used when you report a rate.

## When a user has no streak data

`my_board_streaks` returns nothing when no streak rows exist for the user. Rows are written
hourly, per process board, for each user linked to a CRM owner. Check every cause before
replying so the user can fix everything in one go:

1. **No CRM connected** (`whoami` team connections). Fix: an admin connects the CRM in
   Settings > Connections.
2. **No process boards** (`process_boards`, `composite_process_boards`). Fix: create one now,
   unless cause 1 applies.
3. **Boards exist but the user has no rows.** The user isn't linked to a CRM owner, or a
   manual-only board doesn't list them. You can't see either, so name both. Fix: an admin links
   them to their CRM owner in the CRM's page under Settings > Connections, or adds them to the
   board.
4. **A board is under an hour old.** Fix: none, it fills in at the next hourly run.

Do every fix you can yourself in the same turn. Then reply with only the steps left for the
user, as a short numbered checklist with links, one line each. If nothing is left, say when the
data will appear. No preamble, no explanation of how streaks work, no causes that don't apply.

## Guardrails

- Don't report a count or "no records" from an empty/partial response without checking you queried
  the right board and period.
- Always translate IDs to human names before presenting breakdowns.
