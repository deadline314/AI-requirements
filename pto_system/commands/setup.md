---
description: One-time guided setup for Google Workspace MCP (Gmail + Sheets). Paste your company-provided Google credentials into Claude Desktop. ~3 minutes.
---

Run the one-time Google OAuth setup.

Read `${CLAUDE_PLUGIN_ROOT}/skills/setup-google-oauth/SKILL.md` and follow it from the top.

Critical: **never** ask the user to paste `GOOGLE_OAUTH_CLIENT_SECRET` into the chat. The skill explains exactly where it goes (the Claude Desktop Extensions form, stored in OS secure storage). Refuse to accept the secret in chat — if the user pastes it anyway, follow the skill's incident-response wording (acknowledge, ask them to rotate it, never repeat the value).
