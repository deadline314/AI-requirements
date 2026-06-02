---
name: setup-google-oauth
description: Guide a teammate through pasting the company-provided Google OAuth credentials into Claude Desktop. Use when google-workspace MCP tool calls fail with auth errors, or when the user invokes /email-classify:setup.
---

# Setup Google OAuth — One-time Setup (Email Classify)

This skill is **identical** to the one in the `pto-system` plugin. Both plugins share the same `google-workspace` MCP server, so the OAuth setup only needs to be done once and works for both.

If the user has already completed setup via `pto-system`, **stop here** and tell them:

> *You've already set up Google Workspace MCP through the PTO plugin. Email Classify uses the same credentials — no extra setup needed. Try `/email-classify:check-setup` to verify.*

Otherwise, run the full setup guide. Follow the instructions in `${CLAUDE_PLUGIN_ROOT}/../pto_system/skills/setup-google-oauth/SKILL.md` — every rule, every step, every security guarantee in that file applies here verbatim.

If the parent file is unavailable (the user installed only this plugin and not `pto-system`), embed the same guidance:

1. **Same hard security rules** — never accept `GOOGLE_OAUTH_CLIENT_SECRET` in chat. Never write secrets to any file. Never echo them back even masked.
2. **Same 5 steps** — confirm credentials are in the company password manager → install Google Workspace MCP via DXT → paste into Extensions GUI (NOT chat) → optional client_id format check → first-run Google consent.
3. **Same post-setup** — if the user ever suspects a leak, the admin rotates at <https://console.cloud.google.com/apis/credentials> and re-shares via the password manager.

The canonical, full version of this guide lives in the `pto-system` plugin to avoid duplication. Always prefer reading that file when both plugins are installed.
