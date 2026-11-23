Reschedule upcoming workout routines in Hevy based on the Tuesday–Friday weekly slots.

## Inputs

- **User instructions** (optional, passed as `$ARGUMENTS`): e.g. "Tomorrow I will do Lower A", "Skipping Thursday this week", "Doing Upper B on Friday". If empty, infer automatically from the last workout.
- **Time zone:** Australia/Sydney. Hevy returns `start_time` in UTC — convert to Sydney time before working out which day a workout happened on. Treat "today" as today's date in Sydney.
- **Weekly slots** (fixed, the week restarts every Tuesday):

| Day       | Session |
| --------- | ------- |
| Tuesday   | Upper A |
| Wednesday | Lower A |
| Thursday  | Upper B |
| Friday    | Lower B |

Skipped sessions are dropped, not carried over into the next week — the next week always starts with Upper A on Tuesday.

## Steps

### 1. Fetch data (two calls)

- `get-routines` (page 1, page_size 4) — find the active routines: titles starting with "Upper A", "Upper B", "Lower A" or "Lower B" that do **not** contain "OLD" (case-insensitive). If fewer than four are found and `has_next_page` is true, fetch page 2.
- `get-workouts` (page 1, page_size 1) — get the most recent workout to see what was last done and when.

### 2. Determine the starting point

Convert the last workout's `start_time` to Sydney time and note its day type and date.

- If `$ARGUMENTS` contains an override, apply it. Resolve relative words like "tomorrow" or "Thursday" against today's Sydney date.
- If there's no override, the next session is the next slot (from the table) that falls **after the last workout and on or after today**. A slot already passed this week without a workout is treated as skipped.

### 3. Build the schedule

Give each active routine the date of its next slot from that starting point onwards:
- Slots still ahead this week get this week's dates.
- Slots already done or skipped this week get next week's dates.

Example — today is Wednesday 25 November and the last workout was Upper A on Tuesday 24 November:
- Lower A → Wed 25/11
- Upper B → Thu 26/11
- Lower B → Fri 27/11
- Upper A → Tue 1/12

If an override moves a session (e.g. "Doing Upper B on Friday"), use that date for that routine and keep the rest on their slots, unless two sessions would land on the same day — then ask which one I'm dropping.

### 4. Rename the active routines

Each active routine gets the date of its next scheduled session:

- Title format: `<title without date> - DD/MM` (e.g. `Upper A (Chest/Tricep focus) - 1/12`). Replace any existing ` - DD/MM` suffix rather than adding a second one.
- Only the title changes — do **not** modify exercises, sets, weights, notes, or anything else.

### 5. Show the plan and confirm BEFORE doing anything

Print the proposed plan, e.g.:

```
Upcoming schedule (from Wednesday 25 November):
  Wed 25/11 → Lower A
  Thu 26/11 → Upper B
  Fri 27/11 → Lower B
  Tue 1/12  → Upper A

Routine renames:
  "Lower A (Quad focus)"                  →  "Lower A (Quad focus) - 25/11"
  "Upper B (Tricep/lat focus) - 20/11"    →  "Upper B (Tricep/lat focus) - 26/11"
  ...
```

Then ask: **"Should I go ahead?"**

Only rename after confirmation.

### 6. Apply changes

Call `update-routine` for each active routine, changing only the `title` field. Pass all exercises through unchanged.

## Rules

- Only touch routines whose titles start with Upper A, Upper B, Lower A or Lower B and do NOT contain "OLD".
- Never rename or modify `-OLD` routines.
- Do not change exercises, sets, weights, or notes — only the title.
- Use `page_size: 4` for routines, `page_size: 1` for the last workout (per CLAUDE.md).
