---
name: analyze-boards
description: >
  Answer analytics, streak, and compliance questions about Supered process boards — record
  counts, most-triggered rules, owners with the most flagged records, trends over time, and
  clean-streak / compliance rates. Use when the user asks "how many records are on this board",
  "which rules fire most", "who has the most flagged records", "how's our compliance trending",
  or "what's my streak".
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

`board_streak_rollup` returns a `{date, value}` time series where `value` is the summed
`min_violation_count` across the board on that date — use it to plot trends or spot when
compliance improved or regressed.

**Date filters:** all four streak tools accept `filters` with optional `start_date` / `end_date`
(ISO 8601). With no dates, the server defaults to the last 30 days — state the window you used
when you report a rate.

## Guardrails

- Don't report a count or "no records" from an empty/partial response without checking you queried
  the right board and period.
- Always translate IDs to human names before presenting breakdowns.
