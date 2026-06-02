---
name: email-classify
description: Classify recent Gmail messages into business categories and apply Gmail labels. Use when the user asks to tag, label, categorize, classify, or organize their inbox / recent emails / 最近的信件, or invokes /email-classify:classify-emails.
---

# Email Classify

Scan the user's Gmail for messages received in the last N days, decide a business category for each, and apply the matching Gmail label. Built on top of the `google-workspace` MCP because Claude's built-in Google connector is read-only and cannot create/apply labels.

## Constants

```
DEFAULT_LOOKBACK_DAYS = 1
LABEL_PREFIX = "AI/"
CATEGORIES = [
  "contract",
  "sales",
  "support",
  "finance",
  "hr",
  "newsletter",
  "internal"
]
TERMINAL_LABEL_NAMES = ["AI/contract", "AI/sales", "AI/support",
                       "AI/finance", "AI/hr", "AI/newsletter",
                       "AI/internal"]
BATCH_SIZE = 25
GMAIL_QUERY_TEMPLATE = "newer_than:{days}d -in:chats -in:trash -in:spam"
```

`LABEL_PREFIX` keeps everything this plugin creates under one nested label group in Gmail (e.g. `AI/contract`), so the user can collapse or remove them as a unit.

## Required configuration

- `USER_GOOGLE_EMAIL` — set when the MCP is configured
- `EMAIL_INTERNAL_DOMAIN` (optional) — the user's company domain (e.g. `acme.com`); used by classification rules to decide `internal`. If unset, `internal` will only fire on very strong signals.
- `EMAIL_LOOKBACK_DAYS` (optional) — overrides `DEFAULT_LOOKBACK_DAYS`

## Workflow

### Step 0 — Preflight check

Before anything else, run the `preflight-check` skill (read `${CLAUDE_PLUGIN_ROOT}/skills/preflight-check/SKILL.md`). If it returns `ready: false`, hand off to the appropriate setup flow and stop. Do not proceed until preflight passes.

### Step 1 — Ensure labels exist

Call `list_gmail_labels`. For each name in `TERMINAL_LABEL_NAMES`, if it does not already exist, call `manage_gmail_label` (action=create) to create it. Cache `{name → label_id}` in memory for this run. Skip creation if it already exists — never duplicate.

### Step 2 — Search recent messages

Use `search_gmail_messages` with the query produced from `GMAIL_QUERY_TEMPLATE` and the lookback days. Page through results until done.

For each message ID, also fetch enough header context to classify: `From`, `To`, `Subject`, `List-Id` header (newsletter signal), and a snippet of the body. Prefer `get_gmail_messages_content_batch` over per-message calls.

### Step 3 — Skip already-classified messages

For each message, check its current labels. **If the message already carries any label starting with `AI/`, skip it** — the user (or a previous run) has already classified it, and we treat human edits as authoritative. This is the only "incremental" mechanism needed; no local cache.

### Step 4 — Classify

For each remaining message, read `${CLAUDE_PLUGIN_ROOT}/skills/email-categorize-rules/SKILL.md` once at the top of the run, then apply those rules to assign exactly **one** category from `CATEGORIES`. If nothing matches confidently, leave the message untouched (do NOT label as `internal` by default — `internal` is a positive signal, not a fallback).

### Step 5 — Apply labels in batch

Group classified messages by target label. For each group, call `batch_modify_gmail_message_labels` with `addLabelIds: [label_id]`. Process in chunks of `BATCH_SIZE`.

Read `${CLAUDE_PLUGIN_ROOT}/skills/email-label-apply/SKILL.md` for batching, retry, and quota guidance.

### Step 6 — Summary

Print:

```
Email classification complete (last 1 day)
  Scanned: 84 messages
  Already classified (skipped): 12
  Newly labeled:
    AI/contract:   3
    AI/sales:      9
    AI/support:    5
    AI/finance:    2
    AI/hr:         1
    AI/newsletter: 11
    AI/internal:   8
  Unclassified (no confident match): 33
```

Always include the unclassified count — it tells the user when their rules need refinement.

## Hard rules

- **Never delete or remove labels.** Only add `AI/*` labels.
- **Never read message bodies the user did not authorize.** The MCP's OAuth scope already governs this; do not try to bypass.
- **One category per message.** If a message looks like both `sales` and `contract`, prefer `contract` (it is the more downstream, higher-stakes state).
- **Do not process the Spam or Trash folders** (already excluded in the query, but double-check before applying labels).
