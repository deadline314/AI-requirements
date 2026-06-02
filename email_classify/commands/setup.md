---
description: One-time guided setup for Google Workspace MCP (Gmail). Walks you through Google Cloud Console and Claude Desktop install. ~10 minutes. Shared with pto-system — only need to run once.
---

Run the one-time Google OAuth setup.

Read `${CLAUDE_PLUGIN_ROOT}/skills/setup-google-oauth/SKILL.md` and follow it from the top.

Critical security rule: **never** ask the user to paste `GOOGLE_OAUTH_CLIENT_SECRET` into the chat. The skill explains exactly where it goes. If the user pastes it anyway, follow the skill's incident-response wording (acknowledge, ask them to rotate, never repeat the value).
