---
name: pto-detect-approval
description: Inspect the thread replies of a PTO request and decide whether a manager has approved or rejected it. Use as a sub-skill of pto-sync.
---

# Detect Approval

Scan the thread replies under a PTO request message and return the current decision state.

## Input

- `parent_message` — the original PTO request (with `user` field, the requester's Slack ID)
- `replies` — list of thread reply messages, ordered oldest → newest, each with `user`, `text`, `ts`, and `user_profile.display_name`
- `manager_user_ids` (optional) — explicit allow-list of Slack user IDs whose replies count as authoritative approval. If not provided, accept any reply that is **not** from the requester themselves.

## Output schema

```json
{
  "status": "pending" | "approved" | "rejected",
  "approved_by": "Manager Name" | "",
  "approved_at": "2026-06-02T10:30:00+08:00" | ""
}
```

`approved_by` and `approved_at` are populated only when status is `approved` or `rejected`.

## Decision logic

Walk replies in chronological order. For each reply NOT authored by the requester (and, if `manager_user_ids` is set, only by those IDs):

### Approval keywords (case-insensitive substring match)

- English: `approved`, `approve`, `ok`, `okay`, `lgtm`, `go ahead`, `confirmed`, `sounds good`, `granted`
- Chinese: `同意`, `批准`, `核准`, `准假`, `沒問題`, `沒問題`, `OK`, `好的`, `好`
- Standalone emoji-only message: `✅`, `👍`, `👌`

### Rejection keywords

- English: `rejected`, `reject`, `denied`, `deny`, `not approved`, `cannot`, `no`
- Chinese: `不同意`, `不准`, `駁回`, `拒絕`, `不行`
- Standalone emoji: `❌`, `👎`

### Resolution rules

1. The **last** reply that matches an approval or rejection keyword wins (managers may change their mind in-thread).
2. If there is at least one rejection and no later approval → `rejected`.
3. If there is at least one approval and no later rejection → `approved`.
4. If no reply matches either set → `pending`.
5. Plain "thanks" / "noted" / "received" do NOT count as approval — they are acknowledgment only.

## Time format

`approved_at` is the `ts` of the deciding reply, converted to ISO 8601 with the user's local timezone (default `+08:00` if unknown).

## Edge cases

- A bot reply (Slackbot, workflow) → ignore entirely.
- A reply with both an approval and a rejection word (`"approved but reject the second day"`) → flag as `pending` and let the human decide; do not guess.
- An emoji reaction on the parent message is **not** considered here (we agreed to use thread replies only). If you ever extend this, treat reactions as a separate signal.
