---
name: pto-parse-message
description: Parse a single Slack message into a structured PTO request. Use as a sub-skill of pto-sync — do not invoke standalone. Returns null when the message is not a leave request.
---

# Parse PTO Message

Convert one raw Slack message into a normalized PTO request, or return `null` if it is not a leave request at all.

## Input

A Slack message object with at least:

- `ts` — Slack timestamp
- `user` — Slack user ID
- `text` — message body
- `permalink` — link to the message
- `user_profile.display_name` (or fallback to `real_name`)

## Decision: is this a PTO request?

Treat the message as a PTO request only if it clearly expresses an intent to take leave. Strong signals (any one is enough):

- Mentions of `PTO`, `leave`, `vacation`, `out of office`, `OOO`, `sick`, `time off`
- Chinese: `請假`, `休假`, `病假`, `特休`, `事假`, `年假`, `不在`
- Contains a date range or specific date(s) plus an absence verb

Weak / ambiguous cases (return `null`):

- Asking *about* PTO policy
- Approving someone else's request
- Reminders, jokes, links to articles

When in doubt, prefer `null`. False negatives are recoverable on the next sync; false positives pollute the sheet.

## Output schema

When it IS a PTO request, return:

```json
{
  "slack_ts": "1717200000.123456",
  "requester": "Stan Liu",
  "requester_id": "U01ABCDEFG",
  "start_date": "2026-06-10",
  "end_date": "2026-06-12",
  "leave_type": "annual",
  "reason": "family trip",
  "slack_permalink": "https://workspace.slack.com/archives/CXXX/pYYY"
}
```

Otherwise return `null`.

## Field rules

- `start_date` / `end_date`: always `YYYY-MM-DD`. If only one date is given, set both to the same value. If a range like `6/10 ~ 6/12` is given, parse both. Resolve relative dates (`tomorrow`, `下週一`) against the message's own `ts` (NOT today).
- `leave_type`: normalize to one of `annual`, `sick`, `personal`, `other`. Default to `other` if unclear. Map common terms: `特休/年假/vacation` → `annual`; `病假/sick` → `sick`; `事假/personal` → `personal`.
- `reason`: keep short, strip mentions and emojis. Empty string is fine.
- Do not invent dates. If you cannot extract any date, return `null`.

## Examples

Input text: `"請假 6/10 ~ 6/12，家裡有事"` → returns annual/personal leave with dates `2026-06-10` to `2026-06-12`, `reason: "家裡有事"`.

Input text: `"請問 PTO 政策是怎樣？"` → returns `null` (asking about policy, not requesting leave).

Input text: `"OOO tomorrow, sick"` → resolve `tomorrow` from message ts, `leave_type: "sick"`.
