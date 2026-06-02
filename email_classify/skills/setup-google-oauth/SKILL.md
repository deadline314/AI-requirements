---
name: setup-google-oauth
description: Guide a teammate through pasting the company-provided Google OAuth credentials into Claude Desktop's claude_desktop_config.json. Use when google-workspace MCP tool calls fail with auth errors, or when the user invokes /email-classify:setup.
---

# Setup Google OAuth — One-time Setup (Email Classify)

This skill is **near-identical** to the one in the `pto-system` plugin. Both plugins share the same `google-workspace` MCP server, so the OAuth setup only needs to be done once and works for both.

## If pto-system was already set up

If the user has already completed setup via `pto-system`, **stop here** and tell them:

> *You've already set up Google Workspace MCP through the PTO plugin. Email Classify uses the same credentials — no extra setup needed. Try `/email-classify:check-setup` to verify.*
>
> *Heads up: PTO's setup adds `"sheets", "gmail"` to `--tools`. Email Classify only needs `"gmail"`, so you don't need to change anything — `gmail` is already included. You're good.*

## If only email-classify is being installed

Follow the canonical guide at `${CLAUDE_PLUGIN_ROOT}/../pto_system/skills/setup-google-oauth/SKILL.md` — every rule, every step applies here verbatim, with **one difference** in Step 4: when this plugin is the only one being used, `--tools` should be `"gmail"` (not `"sheets", "gmail"`):

```json
{
  "mcpServers": {
    "google-workspace": {
      "command": "uvx",
      "args": ["workspace-mcp", "--tools", "gmail"],
      "env": {
        "GOOGLE_OAUTH_CLIENT_ID": "PASTE_YOUR_CLIENT_ID_HERE.apps.googleusercontent.com",
        "GOOGLE_OAUTH_CLIENT_SECRET": "PASTE_YOUR_CLIENT_SECRET_HERE",
        "USER_GOOGLE_EMAIL": "your.work.email@company.com"
      }
    }
  }
}
```

If the parent file is unavailable (the user installed only this plugin and not `pto-system`), embed the same guidance from memory:

1. **Same hard security rules** — never accept `GOOGLE_OAUTH_CLIENT_SECRET` in chat. Never write secrets to any file inside the plugin's repo. Never echo them back, even masked. The secret lives ONLY in the user's local `claude_desktop_config.json`, which must NEVER be committed to git or shared.
2. **Same 6 steps** — confirm credentials are in the company password manager → confirm `uvx` is installed → locate `claude_desktop_config.json` → add the `google-workspace` entry above (with `--tools gmail`) → optional client_id format check → first-run Google consent.
3. **Same post-setup** — if the user ever suspects a leak, the admin rotates at <https://console.cloud.google.com/apis/credentials> and re-shares via the password manager.

The canonical, full version of this guide lives in the `pto-system` plugin to avoid duplication. Always prefer reading that file when both plugins are installed.
