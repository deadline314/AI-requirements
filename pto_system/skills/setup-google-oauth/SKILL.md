---
name: setup-google-oauth
description: Guide a teammate through pasting the company-provided Google OAuth credentials into Claude Desktop's claude_desktop_config.json so the google-workspace MCP can run. Use when the user invokes /pto-system:setup or /email-classify:setup, or when google-workspace MCP tool calls fail with auth errors.
---

# Setup Google OAuth — One-time Setup

You are guiding a teammate through a one-time, ~5-minute setup so they can use this plugin in **Claude Desktop**.

The company admin has already done the heavy lifting:

- Created the Google Cloud project
- Enabled Gmail / Sheets / Drive APIs
- Configured the OAuth consent screen as **Internal** (so tokens never expire)
- Created a Desktop OAuth client and obtained `client_id` + `client_secret`
- Shared those two values via the company password manager (1Password / Bitwarden / Dashlane / etc.)

Your job is to walk the user through:

1. Locating their `claude_desktop_config.json`
2. Adding the `google-workspace` MCP server entry with credentials in the `env` block
3. Restarting Claude Desktop and completing the first-run Google consent flow

---

## Hard security rules — read first, never break

These rules are non-negotiable. The setup writes credentials into a plain JSON file on disk, so the rules around handling that file are stricter than a GUI-based flow.

1. **NEVER ask the user to paste their `GOOGLE_OAUTH_CLIENT_SECRET` into the chat.** The secret goes ONLY into `claude_desktop_config.json` on the user's local machine. If the user tries to paste it into chat, refuse and explain why.
2. **NEVER repeat, echo, mask-and-show, or summarize a secret you happen to see.** If the user pastes one anyway, respond ONLY with: *"I deleted that from my view. Please rotate that secret immediately at <https://console.cloud.google.com/apis/credentials> (or ask your IT admin to rotate it for you), then paste the NEW secret only into your local `claude_desktop_config.json`, never into a chat."*
3. **`claude_desktop_config.json` MUST stay on the user's local machine.** It contains the secret in plaintext. It must NEVER be:
   - Committed to git (any repo, public or private)
   - Pasted into chat, Slack, Discord, email, PR, issue, screenshot, or screen share
   - Synced to a public cloud folder (public Dropbox link, public Google Drive, etc.)
   - Copied to a shared machine
4. **NEVER write the secret into any file inside the plugin's repo.** No `.env`, no `config.local.json`, no commented-out example with the real value. The only file that holds the secret is the user's personal `claude_desktop_config.json`.
5. **NEVER call any tool with the secret as an argument.** Tools that accept arbitrary strings (web search, file write, shell with the secret inlined, etc.) are forbidden to receive the secret value. If you must edit `claude_desktop_config.json` for the user, instruct them to do it themselves rather than running an automated edit that puts the secret into your tool input.
6. The `client_id` is less sensitive but still PII-adjacent. You may show it masked (first 12 chars + `...` + `.apps.googleusercontent.com`) to confirm correctness with the user. Never show the unmasked middle, never write the unmasked value to git-tracked files.
7. If the user expresses confusion at any step, offer to pause. A confused user is how secrets leak.

---

## Step 1 — Confirm the user has credentials from a safe channel

Send the user:

> *Quick check before we start: your admin should have shared two values with you — `GOOGLE_OAUTH_CLIENT_ID` and `GOOGLE_OAUTH_CLIENT_SECRET` — through the company password manager (1Password, Bitwarden, Dashlane, or similar).*
>
> *Do you have access to those? **Don't paste them here** — I'll tell you exactly where they go in a moment.*

If the user says they got the credentials over Slack DM / email / SMS / a sticky note, pause and tell them:

> *Heads up: that's not a safe channel for secrets. Once we finish setup, ask your admin to rotate the secret and re-share it via the password manager. We can still continue for now, just plan to refresh it within a day or two.*

If the user says they have no credentials at all, stop and tell them:

> *You'll need to ask your admin (or whoever set this plugin up internally) for the Google credentials before we can continue. They should share them via the company password manager.*

---

## Step 2 — Confirm `uv` / `uvx` is installed

The MCP server runs through `uvx`. If it's missing, the MCP will fail to start with a "command not found" error.

Send the user:

> *Open a terminal and run `uvx --version`.*
>
> *If it prints a version number, you're good. If it says "command not found" or "not recognized", install `uv` first:*
>
> - *macOS / Linux: `curl -LsSf https://astral.sh/uv/install.sh | sh`*
> - *Windows (PowerShell): `powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"`*
>
> *Then close and reopen the terminal so `uvx` is on PATH.*

Wait for confirmation before continuing.

---

## Step 3 — Locate `claude_desktop_config.json`

Send the user the path for their OS:

- **macOS:** `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Windows:** `%APPDATA%\Claude\claude_desktop_config.json` (typically `C:\Users\<you>\AppData\Roaming\Claude\claude_desktop_config.json`)
- **Linux:** `~/.config/Claude/claude_desktop_config.json`

Tell them:

> *Open that file in any text editor. If the file doesn't exist, create it with the content `{}` first.*
>
> *Heads up: this file is private to your machine. After we finish, please confirm it's NOT inside any git repo and NOT inside a synced public folder.*

---

## Step 4 — Add the `google-workspace` MCP server entry

Tell the user to add (or merge into the existing `mcpServers` block) the following entry. Pick the right `--tools` value based on which plugin(s) they use:

- **PTO sync only** → `"sheets", "gmail"`
- **Email classify only** → `"gmail"`
- **Both plugins** → `"sheets", "gmail"` (the union)

Send them this template (they replace the three placeholder values directly in the file — NOT in chat):

```json
{
  "mcpServers": {
    "google-workspace": {
      "command": "uvx",
      "args": ["workspace-mcp", "--tools", "sheets", "gmail"],
      "env": {
        "GOOGLE_OAUTH_CLIENT_ID": "PASTE_YOUR_CLIENT_ID_HERE.apps.googleusercontent.com",
        "GOOGLE_OAUTH_CLIENT_SECRET": "PASTE_YOUR_CLIENT_SECRET_HERE",
        "USER_GOOGLE_EMAIL": "your.work.email@company.com"
      }
    }
  }
}
```

Walk them through it:

> *Replace the three `PASTE_...` placeholders with the values from your password manager + your own work email. Keep the rest as-is.*
>
> *Keeping `--tools` to the minimum your plugin needs is a security best practice — it limits what scopes Google asks you to consent to.*
>
> *If you already have other MCP servers in this file, merge `google-workspace` into the existing `mcpServers` object — don't overwrite the whole file.*
>
> *Save the file. Then **fully quit Claude Desktop** (Cmd+Q on macOS, Ctrl+Q or right-click tray → Quit on Windows — closing the window is not enough) and reopen it.*

Wait for confirmation that they have saved and restarted.

---

## Step 5 — (Optional) Verify the client_id format

If the user wants to double-check they pasted the right thing into the file, ask them to paste **just the client_id** (NOT the secret) into chat. Validate:

- Ends with `.apps.googleusercontent.com`
- Has a numeric prefix followed by `-`
- At least 60 chars total

Show only a masked version back: first 12 chars + `...` + `.apps.googleusercontent.com`. Never show the middle.

If anything looks off, tell them to double-check with the admin before proceeding.

If they paste something that looks like the secret instead (long random string without `.apps.googleusercontent.com`), apply security rule #2 immediately and **do not proceed** until they rotate the secret.

---

## Step 6 — First-run Google consent

Send:

> *Now run `/pto-system:check-setup` (or `/email-classify:check-setup`). The first time, your browser opens a Google sign-in page. Sign in with **your own work email**. You'll see a screen asking for permission to access Gmail / Sheets — click **Allow**. The page closes itself, and you're done.*
>
> *Because the admin set this up as an **Internal** app inside our Google Workspace, this consent is permanent — you won't have to re-authorize unless the admin rotates the secret or you switch computers.*

If the user reports a "Google hasn't verified this app" warning, that means the admin set it up as `External + Testing` instead of `Internal`. Tell them:

> *That warning is normal because the company hasn't gone through Google's verification process. Click **Advanced → Continue**. Your data is safe — this is YOUR company's app, not a third party. (Tokens will expire every 7 days in this mode — flag that to the admin so they can switch to Internal.)*

---

## Done

Once `/pto-system:check-setup` returns `ready: true`, the user is fully set up. Tell them:

> *You're all set. The same Google connection works for both `pto-system` and `email-classify`, so you don't need to repeat this for the other plugin (though if you only added `"gmail"` to `--tools`, you'd need to add `"sheets"` before using PTO sync). Try `/pto-system:sync-pto` or `/email-classify:classify-emails` whenever you're ready.*

Final reminder to send:

> *One last thing: please double-check that `claude_desktop_config.json` is NOT inside any git repo and NOT inside a synced public folder. The secret is in plaintext in that file — it stays only on your machine.*

---

## Common pitfalls

- **`uvx: command not found`** → Step 2 was skipped. Install `uv`, reopen terminal, restart Claude Desktop.
- **Tokens expire after 7 days** → admin set up `External + Testing`, not `Internal`. Tell the user to ping the admin; meanwhile they re-auth weekly.
- **"This app isn't verified" page** → expected if the company hasn't verified the app with Google. Click **Advanced → Continue**.
- **`/...:check-setup` says `ready: false` / `reason: "auth"`** → the user hasn't completed the Google consent flow yet (Step 6). Trigger it by retrying.
- **MCP tool calls return "missing scope" errors** → user only added `"gmail"` to `--tools` but is now using PTO (which needs `sheets`). Update `--tools` to include both, restart Claude Desktop, re-run consent.
- **JSON parse error in Claude Desktop logs** → user pasted the template but broke the existing `mcpServers` block. Tell them to validate the file at <https://jsonlint.com/> (paste-only-once, no save) or just open it in an editor with JSON syntax highlighting.
- **User pastes the secret in chat anyway** → security rule #2 verbatim. Do not soften the message. The cost of a leaked secret is much higher than a moment of awkwardness.
- **User has no credentials** → they need to ask the admin via the company password manager. Do not walk them through Google Cloud Console — that's the admin's job, not theirs.
- **User asks if they can commit `claude_desktop_config.json` to a private repo** → no. Even private repos get cloned, forked, screenshot, and shared. The file stays on the local machine, full stop.
