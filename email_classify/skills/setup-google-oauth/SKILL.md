---
name: setup-google-oauth
description: Guide a user through setting up Google OAuth credentials for the Google Workspace MCP. Use when google-workspace MCP tool calls fail with auth errors, or when the user invokes /email-classify:setup.
---

# Setup Google OAuth — One-time Setup (Email Classify)

This skill is **identical** to the one in the `pto-system` plugin. Both plugins share the same `google-workspace` MCP server, so the OAuth setup only needs to be done once and works for both.

If the user has already completed setup via `pto-system`, **stop here** and tell them: *"You've already set up Google Workspace MCP through the PTO plugin. Email Classify uses the same credentials — no extra setup needed. Try `/email-classify:check-setup` to verify."*

Otherwise, run the full setup guide. Follow the instructions in `${CLAUDE_PLUGIN_ROOT}/../pto_system/skills/setup-google-oauth/SKILL.md` — every rule, every step, every security guarantee in that file applies here verbatim. **Including the two-path branch (company-provided vs self-setup) at Step 0.**

If the parent file is unavailable (the user installed only this plugin and not `pto-system`), embed the same guidance:

1. **Same Step 0 branch**: ask whether the user's company has provided `client_id` / `client_secret`. If yes → fast path (DXT install + paste into Extensions form, skip GCP Console entirely). If no → full path (GCP project + OAuth consent + create OAuth client + DXT install).
2. **Same hard security rules**: never accept `GOOGLE_OAUTH_CLIENT_SECRET` in chat. Never write secrets to any file. Never echo them back even masked.
3. **Same post-setup**: rotate at <https://console.cloud.google.com/apis/credentials> if leaked. Move credentials to a password manager.

The canonical, full version of this guide lives in the `pto-system` plugin to avoid duplication. Always prefer reading that file when both plugins are installed.
