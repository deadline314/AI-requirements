---
name: pto-sync
description: Sync PTO requests from a Slack channel into a Google Sheet incrementally. Use when the user wants to import, sync, refresh, or update PTO/leave/請假 records from Slack to the tracking spreadsheet, or whenever the user invokes /pto-system:sync-pto.
---

# PTO Sync

You orchestrate the end-to-end PTO sync flow: read recent Slack messages from the PTO channel, parse them into structured requests, detect manager approval in thread replies, and append/update rows in the Google Sheet — all incrementally so the same request is never written twice.

## Required configuration

Read these from environment variables or ask the user once if any are missing:

- `PTO_SLACK_CHANNEL_ID` — the Slack channel where employees post PTO requests
- `PTO_GOOGLE_SHEET_ID` — the spreadsheet ID of the PTO tracking sheet
- `PTO_GOOGLE_SHEET_TAB` — the tab/worksheet name (default: `PTO`)
- `PTO_LOOKBACK_DAYS` — how many days to scan in Slack (default: `14`)

## Expected sheet schema

The plugin uses the **first row** of the sheet as headers and matches columns by name. Expected columns (create them if missing):

| Column | Meaning |
| --- | --- |
| `slack_ts` | Slack message timestamp (unique key, used as dedup ID) |
| `requester` | Slack user display name |
| `requester_id` | Slack user ID |
| `start_date` | Leave start (YYYY-MM-DD) |
| `end_date` | Leave end (YYYY-MM-DD) |
| `leave_type` | annual / sick / personal / other |
| `reason` | Free-text reason |
| `status` | `pending` / `approved` / `rejected` |
| `approved_by` | Slack user who approved (display name) |
| `approved_at` | ISO timestamp of approval |
| `slack_permalink` | Link back to the Slack message |
| `last_synced_at` | ISO timestamp of last sync touch |

## Workflow

Follow these steps in order. Stop and report to the user immediately if any step fails.

The flow has a mandatory **dry-run preview + user confirmation** between detection and writing. Nothing touches the Sheet until the user explicitly approves.

### Step 0 — Preflight check

Before anything else, run the `preflight-check` skill (read `${CLAUDE_PLUGIN_ROOT}/skills/preflight-check/SKILL.md`). If it returns `ready: false`, hand off to the appropriate setup flow it recommends and stop. Do not proceed to Step 1 until preflight passes.

### Step 1 — Read existing sheet state

Call `list_sheets` (or equivalent) to confirm the tab exists. If not, create it with the headers above. Then read the entire data range, e.g. `${PTO_GOOGLE_SHEET_TAB}!A:L`. Build an in-memory map keyed by `slack_ts` → row number + current row values. This is the source of truth for "already processed" — **do not** create any local cache file.

### Step 2 — Fetch recent Slack messages

Use the Slack connector to list messages from `PTO_SLACK_CHANNEL_ID` within the last `PTO_LOOKBACK_DAYS` days (oldest=now - days). Skip bot messages and messages without text.

### Step 3 — Parse each message into a PTO request

For each parent message, invoke the `pto-parse-message` skill (read `${CLAUDE_PLUGIN_ROOT}/skills/pto-parse-message/SKILL.md`). It returns a structured object or `null` if the message is not a PTO request — silently skip non-PTO chatter.

### Step 4 — Detect approval in the thread

For each parsed request, fetch the message's thread replies. Pass the replies to the `pto-detect-approval` skill (read `${CLAUDE_PLUGIN_ROOT}/skills/pto-detect-approval/SKILL.md`). It returns `{ status, approved_by, approved_at }` based on thread reply keywords.

### Step 5 — Build a dry-run plan (NO writes yet)

Compare against the in-memory map from Step 1 and build an in-memory `plan` list. Each item has shape:

```
{
  "action": "insert" | "update" | "skip",
  "slack_ts": "...",
  "requester": "...",
  "summary": "Stan Liu  2026-06-10~12  annual  status: pending → approved",
  "row_payload": { ... full data to write if action != skip ... }
}
```

Decision rules:

- **Not in sheet** → `insert` with all parsed fields + status from Step 4.
- **In sheet, status `pending`, new status `approved` or `rejected`** → `update` (only `status`, `approved_by`, `approved_at`, `last_synced_at`).
- **In sheet, status is `approved` or `rejected`** → `skip` (final state, never overwrite).
- **In sheet, both still `pending`** → `skip` (nothing changed).

**Do not call any write tool in this step.** Build the plan in memory only.

### Step 6 — Show the dry-run preview and ask the user to choose

Print the preview in this exact format so the UI can render the choices as clickable shortcuts:

```
PTO sync — dry-run preview (last {N} days)

  Will INSERT ({insert_count}):
    1. Stan Liu     2026-06-10 → 2026-06-12   annual    status: approved      (approved by Mgr Wang)
    2. Alice Chen   2026-06-15                sick      status: pending
    ...

  Will UPDATE ({update_count}):
    1. Bob Lin      2026-05-28 → 2026-05-30   annual    pending → approved    (approved by Mgr Wang)
    ...

  Will SKIP ({skip_count}): unchanged or already final

How do you want to proceed? Reply with the number:
  1) Apply ALL changes to the Sheet
  2) Cancel — write nothing
  3) Review one-by-one (I'll ask for each insert/update individually)
```

Then **stop and wait for the user's reply**. Do not call any tool until you receive `1`, `2`, or `3`.

- If user replies `1` → go to Step 7 (apply all).
- If user replies `2` → print `Cancelled. Sheet unchanged.` and exit.
- If user replies `3` → enter the per-item loop: for each non-skip item in `plan`, print one line with the change and ask:

  ```
  [3 of 5] Alice Chen 2026-06-15 sick  status: pending
    1) Apply
    2) Skip this one
    3) Cancel the rest
  ```

  Collect each answer one at a time. `3` exits the loop and applies whatever was already approved.

If the user types something other than `1`/`2`/`3`, treat it as a question and answer it, then re-show the same prompt.

### Step 7 — Apply approved changes in batch

Now (and only now) call the write tools. Group inserts and updates and prefer batched writes (`batch_update_cells`) over per-row calls.

When the MCP triggers Claude Code's permission dialog ("Allow `update_cells` to write to spreadsheet `…`?"), that's the OS-level second confirmation — the user clicks Allow / Allow always / Deny. Do NOT try to suppress or auto-answer this; it's the user's safety net.

After each tool call, also update `last_synced_at` for every row touched.

### Step 8 — Summary report

Print a concise summary to the user:

```
PTO sync complete
  Scanned: 27 Slack messages (last 14 days)
  Plan was: 3 inserts, 2 updates, 22 skips
  Applied:  3 inserts, 2 updates
  Cancelled by user: 0
  Sheet: https://docs.google.com/spreadsheets/d/.../edit#gid=...
```

Include the sheet URL so the user can click through.

If the user cancelled at Step 6, end with `Cancelled. Sheet unchanged.` and skip the apply counts.

## Edge cases

- If the same person posts the same date range twice, the `slack_ts` differs so both rows exist. Do not try to merge — the human in the loop decides.
- If a message is edited after first sync, do **not** re-parse — the row is keyed by `slack_ts` and editing history is out of scope for v1.
- If the sheet is missing entirely, ask the user to create it first and provide the ID — do not auto-create spreadsheets, only tabs within an existing one.
- If Slack rate-limits, back off and resume — never retry tightly in a loop.

## Constants used by this skill

```
DEFAULT_LOOKBACK_DAYS = 14
DEFAULT_SHEET_TAB = "PTO"
SHEET_HEADERS = [
  "slack_ts", "requester", "requester_id",
  "start_date", "end_date", "leave_type", "reason",
  "status", "approved_by", "approved_at",
  "slack_permalink", "last_synced_at"
]
TERMINAL_STATUSES = {"approved", "rejected"}
```
