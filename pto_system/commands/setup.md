---
description: One-time guided setup for Google Workspace MCP (Gmail + Sheets). Add your company-provided Google credentials into Claude Desktop's claude_desktop_config.json. ~5 minutes.
---

Run the one-time Google OAuth setup.

Read `${CLAUDE_PLUGIN_ROOT}/skills/setup-google-oauth/SKILL.md` and follow it from the top.

Critical: **never** ask the user to paste `GOOGLE_OAUTH_CLIENT_SECRET` into the chat. The skill explains exactly where it goes (the user's local `claude_desktop_config.json`, never the chat, never any file inside the plugin repo). Refuse to accept the secret in chat — if the user pastes it anyway, follow the skill's incident-response wording (acknowledge, ask them to rotate it, never repeat the value).
