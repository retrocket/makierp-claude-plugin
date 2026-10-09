# Scheduled Analytics Mail

Use this workflow when the user wants a saved question delivered repeatedly by email.

## Safe workflow

1. Inspect the question with `analytics_questions_show`, including its current computed parameter surface. A schedule always runs the question's current saved definition; it does not copy the query or inherited declarations.
2. Clarify which recipient sources are enabled: collection members, selected members, organizational units, and/or external addresses.
3. Translate the user's calendar wording into the structured `recurrence` object and provide an IANA timezone.
4. Give every required field still visible on that computed surface an explicit fixed or occurrence-relative definition, even when the question declares a default.
5. Choose the saved-view attachment format and any additional tabular data files.
6. Call `analytics_mail_schedules_preview` with `question_id`, recurrence, timezone, and validity dates. It is read-only and must not create a schedule.
7. Create or update only after the returned `preview.summary` and five `preview.occurrences` match the requested calendar. Read those values back to the user.
8. Call `analytics_mail_schedules_run` only when the user explicitly asks for an immediate or test delivery. It sends real email with real file attachments.
9. Re-read with `analytics_mail_schedules_show` after a run and report its terminal history state.

Management tools are `analytics_mail_schedules_list`, `analytics_mail_schedules_show`, `analytics_mail_schedules_create`, `analytics_mail_schedules_update`, `analytics_mail_schedules_pause`, `analytics_mail_schedules_resume`, `analytics_mail_schedules_delete`, and `analytics_mail_schedules_run`. Prefer pause over delete when the user may resume later.

For `show`, `update`, `pause`, `resume`, `delete`, and `run`, pass the schedule key as `id`. Do not rename it to `schedule_id`; that field appears in run output but is not a management-tool input.

## Recurrence contract

Always send the complete version 1 shape:

```json
{
  "version": 1,
  "frequency": "day",
  "interval": 1,
  "times": ["09:00", "18:00"],
  "weekdays": [],
  "month_days": [],
  "ordinal_weekdays": [],
  "months": [],
  "day_match": "all",
  "window": null
}
```

- `frequency`: `hour`, `day`, `week`, `month`, or `year`.
- `interval`: repeat every N frequency units, from 1 through 99. `week` plus `interval: 2` is every two weeks anchored to `starts_on`.
- `times`: whole-hour local delivery times such as `09:00` for day, week, month, and year rules.
- `window`: inclusive whole-hour local daily `{starts_at, ends_at}` window required for hour rules.
- `weekdays`: ISO weekdays, Monday `1` through Sunday `7`.
- `month_days`: calendar days `1` through `31`; `-1` means the last day of the month.
- `ordinal_weekdays`: entries such as `{"ordinal":1,"weekday":1}` for first Monday or `{"ordinal":-1,"weekday":5}` for last Friday.
- `months`: January `1` through December `12`.
- `day_match`: when both `weekdays` and month-day rules exist, `all` requires both kinds to match and `any` accepts either kind.

Examples:

- Every weekday at 09:00: `frequency: week`, `interval: 1`, `weekdays: [1,2,3,4,5]`, `times: ["09:00"]`.
- Every 2 hours during working hours: `frequency: hour`, `interval: 2`, `times: []`, `window: {"starts_at":"08:00","ends_at":"18:00"}`.
- First and last day of every month: `frequency: month`, `month_days: [1,-1]`, `times: ["09:00"]`.
- First Monday of each quarter: `frequency: year`, `months: [1,4,7,10]`, `ordinal_weekdays: [{"ordinal":1,"weekday":1}]`, `times: ["09:00"]`.

Always provide `starts_on` as `YYYY-MM-DD` and a canonical IANA `timezone` such as `Asia/Nicosia`. Use `ends_on` only when delivery must stop automatically. Scheduling resolution is one hour; minute-level times are rejected. The recurrence follows local daylight-saving changes; nonexistent local times are skipped.

## Parameters

Every required parameter on the question's current computed surface needs its
own fixed or relative definition. A question default does not replace this
schedule requirement. Parameters hidden by valid pin/map composition do not get
separate public schedule fields, but they must still resolve inward through the
saved-question contract.

```json
{
  "warehouse_id": {"type": "fixed", "value": 42},
  "from": {"type": "relative", "value": "start_of_previous_month"},
  "to": {"type": "relative", "value": "end_of_previous_month"}
}
```

Relative tokens: `today`, `yesterday`, `start_of_week`, `end_of_week`, `start_of_previous_week`, `end_of_previous_week`, `start_of_month`, `end_of_month`, `start_of_previous_month`, `end_of_previous_month`. They resolve from the intended occurrence in the schedule timezone.

For an optional parameter, omitting its name allows the question default. A
fixed definition with native JSON `null` explicitly clears that default; a
required parameter cannot be satisfied by `null`. Numeric `0` and boolean
`false` are fixed values, not omission.

Each occurrence resolves relative values in the schedule timezone, then runs
the current saved question through the ordinary surface, pin/rename/map,
default, requiredness, and column/relation-effect pipeline. Later question
changes affect later occurrences. If a changed surface makes the schedule
invalid, the run must fail explicitly rather than use a stale copied contract.

## File attachments

Every schedule sends files directly; scheduled delivery is not link-only. Provide one primary file that preserves the saved question view and up to four additional files generated from the underlying rows:

```json
{
  "attachments": {
    "question": {"format": "pdf"},
    "data": [{"format": "xlsx"}, {"format": "csv"}]
  }
}
```

- Table question: primary `pdf`, `xlsx`, `csv`, or `json`.
- Pivot question: primary `pdf`, `xlsx`, or `csv`.
- Chart or number question: primary `pdf`, `png`, or `svg`.
- Additional tabular data: `pdf`, `xlsx`, `csv`, or `json`; each format may appear once.

The question file always comes first. Additional files use the current raw result rows as a plain table and do not apply a print template. A run fails instead of silently switching to a link when the combined files exceed the configured mail attachment limit.

## Audience rules

Every source has an independent switch. A configured selection is retained when its switch is off but does not participate in delivery.

- `include_collection_members`: current active collection owner and members.
- `include_selected_members` + `member_ids`: explicitly selected active tenant members.
- `include_organizational_unit_members` + `organizational_unit_ids`: current active members assigned to selected organizational units or their descendant units.
- `include_external_emails` + `emails`: direct external addresses with optional names.

At least one enabled source must resolve from its corresponding configuration. Sources are unioned and deduplicated by normalized email at send time. External addresses require separate permission. Each recipient receives a private message; addresses are not exposed to one another.

## Lifecycle

- Pause when delivery should stop temporarily; delete only when definition and history should be removed.
- `send_empty_results: false` records an empty run as skipped and sends nothing.
- Scheduler catch-up is bounded to one occurrence after downtime.
- A manual run does not change recurrence timing.
- History states are `pending`, `processing`, `completed`, `skipped`, and `failed`. Never claim delivery succeeded until history says `completed`.
- Per-recipient delivery states are `pending`, `queued`, `sent`, and `failed`; a run reaches `completed` only after every recipient reaches `sent`.
