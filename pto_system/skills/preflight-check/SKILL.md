---
name: preflight-check
description: Lightweight read-only check that the google-workspace MCP is configured and reachable. Use as the first step of any pto-* skill before doing real work. Does NOT call write tools, does NOT touch any secret.
---

# Preflight Check

A 3-second read-only check that the Google Workspace MCP is alive and authenticated. Run this at the very top of `pto-sync` (and any future write-flow) so the user gets a friendly setup nudge instead of a cryptic auth error mid-flow.

## What this skill does

1. Calls `list_gmail_labels` (read-only, smallest possible probe). It either:
   - Returns labels → MCP is healthy and OAuth token is valid → return `{"ready": true}`
   - Returns an auth error → return `{"ready": false, "reason": "auth"}`
   - Returns a not-found / tool-doesn't-exist error → return `{"ready": false, "reason": "not_installed"}`
   - Times out → return `{"ready": false, "reason": "unreachable"}`
2. **Never** calls any write tool. **Never** touches any secret. **Never** prints any value that could be a secret or a token.

## What to do based on the result

If `ready: true` → continue with the calling skill silently.

If `ready: false`:

- `reason: "not_installed"` → Tell the user:

  > *"It looks like the Google Workspace MCP isn't installed in Claude Desktop yet. I'll walk you through it — it's a one-time, ~10-minute setup. Want to start now?"*

  If they say yes, hand off to the `setup-google-oauth` skill.

- `reason: "auth"` → Tell the user:

  > *"The Google Workspace MCP is installed but the OAuth token is missing or expired. This usually means either (a) it's the first time you're using it on this computer, or (b) the token has expired (happens every ~7 days for unverified test apps). The fix is the same: I'll trigger the OAuth flow now and your browser will open. Continue?"*

  Then call any read-only Google tool (e.g. `list_gmail_labels` again) — the MCP itself opens the browser. The user clicks through Google's consent screen. Done.

- `reason: "unreachable"` → Tell the user:

  > *"The Google Workspace MCP server isn't responding. The most common fix is to fully quit Claude Desktop (Cmd/Ctrl+Q, not just close the window) and reopen it. If that doesn't help, check Settings → Extensions to see if the extension is enabled."*

## Hard rules

- This skill must complete in under 5 seconds. If `list_gmail_labels` doesn't return in 5 seconds, treat it as unreachable.
- Never log the actual response body — it contains user data (label IDs, names). Just check success vs error.
- Never call `manage_gmail_label`, `update_cells`, or any write tool from this skill. Read-only only.
- Never display the user's email, client_id, or any token value. The result is binary: ready or not ready.
