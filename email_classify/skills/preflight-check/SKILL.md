---
name: preflight-check
description: Lightweight read-only check that the google-workspace MCP is configured and reachable. Use as the first step of email-classify before doing real work. Does NOT call write tools, does NOT touch any secret.
---

# Preflight Check (Email Classify)

A 3-second read-only check that the Google Workspace MCP is alive and authenticated.

## What this skill does

1. Calls `list_gmail_labels` (read-only, smallest possible probe). It either:
   - Returns labels → MCP is healthy and OAuth token is valid → return `{"ready": true}`
   - Returns an auth error → return `{"ready": false, "reason": "auth"}`
   - Returns a not-found error → return `{"ready": false, "reason": "not_installed"}`
   - Times out → return `{"ready": false, "reason": "unreachable"}`
2. **Never** calls any write tool. **Never** touches any secret. **Never** prints any value that could be a secret or a token.

## What to do based on the result

If `ready: true` → continue with `email-classify` silently.

If `ready: false`:

- `reason: "not_installed"` → Hand off to `setup-google-oauth` skill. Tell the user setup is one-time and takes ~10 minutes.
- `reason: "auth"` → Tell the user the OAuth token has expired or is missing; trigger the consent flow by retrying a read-only call and instruct them to complete the browser consent.
- `reason: "unreachable"` → Tell the user to fully quit Claude Desktop (Cmd/Ctrl+Q) and reopen.

## Hard rules

- Must complete in under 5 seconds.
- Never log the response body.
- Never call any write tool from this skill.
- Never display the user's email, client_id, or any token value.
