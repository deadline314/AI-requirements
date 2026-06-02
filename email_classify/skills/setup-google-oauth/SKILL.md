---
name: setup-google-oauth
description: Guide a non-technical user through setting up Google OAuth credentials for the Google Workspace MCP. Use when google-workspace MCP tool calls fail with auth errors, or when the user invokes /email-classify:setup.
---

# Setup Google OAuth — One-time Setup (Email Classify)

This skill is **identical** to the one in the `pto-system` plugin. Both plugins share the same `google-workspace` MCP server, so the OAuth setup only needs to be done once and works for both.

If the user has already completed setup via `pto-system`, **stop here** and tell them: *"You've already set up Google Workspace MCP through the PTO plugin. Email Classify uses the same credentials — no extra setup needed. Try `/email-classify:check-setup` to verify."*

Otherwise, run the full setup guide. Follow the instructions in `${CLAUDE_PLUGIN_ROOT}/../pto_system/skills/setup-google-oauth/SKILL.md` — every rule, every step, every security guarantee in that file applies here verbatim.

If the parent file is unavailable (the user installed only this plugin and not `pto-system`), embed the same guidance:

1. The same hard security rules apply: **never** ask the user to paste `GOOGLE_OAUTH_CLIENT_SECRET` into chat. Never write secrets to any file. Never echo them back even masked.
2. Walk them through Part A (Google Cloud Console — create project, enable Gmail/Sheets/Drive APIs, configure OAuth consent screen with themselves as test user, create OAuth client of type **Desktop app**).
3. Walk them through Part B (Claude Desktop install — recommend the `.dxt` one-click installer from <https://github.com/taylorwilsdon/google_workspace_mcp/releases>; the secret goes into the Desktop Extensions GUI form, never a JSON file).
4. Walk them through Part C (verify with `/email-classify:check-setup`, expect a Google consent browser tab on first run).
5. Walk them through Part D (post-setup security checklist: move credentials.json to password manager, never commit, rotate at <https://console.cloud.google.com/apis/credentials> if leaked).

The canonical, full version of this guide lives in the `pto-system` plugin to avoid duplication. Always prefer reading that file when both plugins are installed.
