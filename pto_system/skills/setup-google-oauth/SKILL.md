---
name: setup-google-oauth
description: Guide a teammate through pasting the company-provided Google OAuth credentials into Claude Desktop's Google Workspace MCP. The Google Cloud project + OAuth client are already provisioned by the admin. Use when the user invokes /pto-system:setup or /email-classify:setup, or when google-workspace MCP tool calls fail with auth errors.
---

# Setup Google OAuth — One-time Setup

You are guiding a teammate through a one-time, ~3-minute setup so they can use this plugin in **Claude Desktop**.

The company admin has already done the heavy lifting:

- Created the Google Cloud project
- Enabled Gmail / Sheets / Drive APIs
- Configured the OAuth consent screen as **Internal** (so tokens never expire)
- Created a Desktop OAuth client and obtained `client_id` + `client_secret`
- Shared those two values via the company password manager (1Password / Bitwarden / Dashlane / etc.)

Your job is to walk the user through:

1. Installing the Google Workspace MCP (`.dxt` one-click installer)
2. Pasting the credentials into Claude Desktop's Extensions form (NOT into chat, NOT into a file)
3. Doing the first-run Google consent flow

---

## Hard security rules — read first, never break

These rules are non-negotiable.

1. **NEVER ask the user to paste their `GOOGLE_OAUTH_CLIENT_SECRET` into the chat.** The secret goes ONLY into the Claude Desktop Extensions GUI form, where it is stored in the OS secure store (Keychain on macOS, Credential Manager on Windows). If the user tries to paste it in chat, refuse and explain why.
2. **NEVER repeat, echo, mask-and-show, or summarize a secret you happen to see.** If the user pastes one anyway, respond ONLY with: *"I deleted that from my view. Please rotate that secret immediately at <https://console.cloud.google.com/apis/credentials> (or ask your IT admin to rotate it for you), then paste the NEW secret only into the Desktop Extensions form, never into a chat."*
3. **NEVER write a secret to any file** — not `claude_desktop_config.json`, not `.env`, not anywhere on disk that the user controls. The DXT installer + Desktop GUI handles storage.
4. **NEVER call any tool with the secret as an argument.** Tools that accept arbitrary strings (web search, file write, shell) are forbidden to receive the secret value.
5. The `client_id` is less sensitive but still PII-adjacent. You may show it masked (first 12 chars + `...` + last 30 chars, where the last 30 should be `.apps.googleusercontent.com`) to confirm correctness with the user. Never write the unmasked value to git-tracked files.
6. If the user expresses confusion at any step, offer to pause. A confused user is how secrets leak.

---

## Step 1 — Confirm where their credentials came from

Send the user:

> *Quick check before we start: your admin should have shared two values with you — `GOOGLE_OAUTH_CLIENT_ID` and `GOOGLE_OAUTH_CLIENT_SECRET` — through the company password manager (1Password, Bitwarden, Dashlane, or similar).*
>
> *Do you have access to those? **Don't paste them here** — I'll tell you exactly where they go in a moment.*

If the user says they got the credentials over Slack DM / email / SMS / a sticky note, pause and tell them:

> *Heads up: that's not a safe channel for secrets. Once we finish setup, ask your admin to rotate the secret and re-share it via the password manager. We can still continue for now, just plan to refresh it within a day or two.*

If the user says they have no credentials at all, stop and tell them:

> *You'll need to ask your admin (or whoever set this plugin up internally) for the Google credentials before we can continue. They should share them via the company password manager.*

---

## Step 2 — Install the Google Workspace MCP via DXT

Send:

> *Open <https://github.com/taylorwilsdon/google_workspace_mcp/releases> and find the latest release. Download the file ending in `.dxt`. Double-click it — Claude Desktop will open and show an Install dialog. Click **Install**.*

Wait for confirmation that the install succeeded.

If the install fails ("couldn't open" or similar) the user's Claude Desktop is too old. Tell them to update Claude Desktop and try again.

---

## Step 3 — Paste credentials into the Extensions GUI

This is the **only** safe place for the secret. Send the user:

> *In Claude Desktop, go to **Settings → Extensions → Google Workspace MCP**. You'll see a form with three fields:*
>
> - *`GOOGLE_OAUTH_CLIENT_ID` → paste the Client ID from your password manager*
> - *`GOOGLE_OAUTH_CLIENT_SECRET` → paste the secret HERE (this is the safe place — OS secure store, not a config file, not chat)*
> - *`USER_GOOGLE_EMAIL` → your own work email address*
>
> *Click **Save**. Then **fully quit Claude Desktop** (Cmd+Q on macOS, Ctrl+Q on Windows — not just close the window) and reopen it.*

Wait for confirmation that they have saved and restarted.

---

## Step 4 — (Optional) Verify the client_id format

If the user wants to double-check they pasted the right thing, ask them to paste **just the client_id** (NOT the secret) into chat. Validate:

- Ends with `.apps.googleusercontent.com`
- Has a numeric prefix followed by `-`
- At least 60 chars total

Show only a masked version back: first 12 chars + `...` + `.apps.googleusercontent.com`. Never show the middle.

If anything looks off, tell them to double-check with the admin before proceeding.

If they paste something that looks like the secret instead (long random string without `.apps.googleusercontent.com`), apply security rule #2 immediately and **do not proceed** until they rotate the secret.

---

## Step 5 — First-run Google consent

Send:

> *Now run `/pto-system:check-setup` (or `/email-classify:check-setup`). The first time, your browser opens a Google sign-in page. Sign in with **your own work email**. You'll see a screen asking for permission to access Gmail / Sheets — click **Allow**. The page closes itself, and you're done.*
>
> *Because the admin set this up as an **Internal** app inside our Google Workspace, this consent is permanent — you won't have to re-authorize unless the admin rotates the secret or you switch computers.*

If the user reports a "Google hasn't verified this app" warning, that means the admin set it up as `External + Testing` instead of `Internal`. Tell them:

> *That warning is normal because the company hasn't gone through Google's verification process. Click **Advanced → Continue**. Your data is safe — this is YOUR company's app, not a third party. (Tokens will expire every 7 days in this mode — flag that to the admin so they can switch to Internal.)*

---

## Done

Once `/pto-system:check-setup` returns `ready: true`, the user is fully set up. Tell them:

> *You're all set. The same Google connection works for both `pto-system` and `email-classify`, so you don't need to repeat this for the other plugin. Try `/pto-system:sync-pto` or `/email-classify:classify-emails` whenever you're ready.*

---

## Common pitfalls

- **Tokens expire after 7 days** → admin set up `External + Testing`, not `Internal`. Tell the user to ping the admin; meanwhile they re-auth weekly.
- **"This app isn't verified" page** → expected if the company hasn't verified the app with Google. Click **Advanced → Continue**.
- **DXT install fails with "couldn't open"** → Claude Desktop too old. Update it.
- **`/...:check-setup` says `ready: false` / `reason: "auth"`** → the user hasn't completed the Google consent flow yet (Step 5). Trigger it by retrying.
- **User pastes the secret in chat anyway** → security rule #2 verbatim. Do not soften the message. The cost of a leaked secret is much higher than a moment of awkwardness.
- **User has no credentials** → they need to ask the admin via the company password manager. Do not walk them through Google Cloud Console — that's the admin's job, not theirs.
