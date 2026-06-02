---
name: setup-google-oauth
description: Guide a user through setting up Google OAuth credentials for the Google Workspace MCP. Has two paths depending on whether the user's company has already provided credentials. Use when the user invokes /pto-system:setup or /email-classify:setup, or when google-workspace MCP tool calls fail with auth errors.
---

# Setup Google OAuth — One-time Setup

You are guiding a user through a one-time Google OAuth setup so they can use this plugin in **Claude Desktop**.

There are two paths. **Always start by asking which one applies** — the difference matters.

## Step 0 — Ask which path

Send the user this exact message:

> *Before we start: did your company / IT / team lead give you a Google `client_id` and `client_secret` for this plugin?*
>
> *1) Yes, I have credentials from my company → fast path (~3 minutes)*
> *2) No, I need to set everything up myself → full path (~10 minutes, requires a Google Cloud account)*

Wait for `1` or `2`. Branch accordingly.

If the user is unsure, ask: *"Have you been told to share a Google Cloud project, or is this your personal setup?"* — that usually clears it up. Default to path 2 only if there's clearly no company involvement.

---

## Hard security rules — apply to BOTH paths

These rules are non-negotiable. Read them before doing anything else.

1. **NEVER ask the user to paste their `GOOGLE_OAUTH_CLIENT_SECRET` into the chat.** The secret goes ONLY into the Claude Desktop Extensions GUI form, where it is stored in the OS secure store (Keychain on macOS, Credential Manager on Windows). If the user tries to paste it in chat, refuse and explain why.
2. **NEVER repeat, echo, mask-and-show, or summarize a secret you happen to see.** If the user pastes one anyway, respond ONLY with: *"I deleted that from my view. Please rotate that secret immediately at <https://console.cloud.google.com/apis/credentials> (or ask your IT admin to rotate it for you), then paste the NEW secret only into the Desktop Extensions form, never into a chat."*
3. **NEVER write a secret to any file** — not `claude_desktop_config.json`, not `.env`, not anywhere on disk that the user controls. The DXT installer + Desktop GUI handles storage.
4. **NEVER call any tool with the secret as an argument.** Tools that accept arbitrary strings (web search, file write, shell) are forbidden to receive the secret value.
5. The `client_id` is less sensitive but still PII-adjacent. You may show it masked (first 12 chars + `...` + last 30 chars, where the last 30 should be `.apps.googleusercontent.com`) to confirm correctness with the user. Never write the unmasked value to git-tracked files.
6. If the user expresses confusion at any step, offer to pause. A confused user is how secrets leak.

---

## Path 1 — Company-provided credentials (fast path, ~3 minutes)

The user already has `client_id` and `client_secret` from their admin. Skip all of Google Cloud Console — they're done.

### 1.1 Confirm what they have

Ask: *"Where did your admin send you the credentials? Common safe places are 1Password, Bitwarden, Dashlane, or a corporate password manager. **Don't paste the secret here** — I'll tell you exactly where it goes in a moment."*

If they say it came over Slack DM / email / SMS / a sticky note — pause and tell them:

> *"Heads up: that's not a safe channel for secrets. Once we finish setup, ask your admin to rotate the secret and re-share it via a password manager. For now we can still continue, just plan to refresh it within a day or two."*

### 1.2 Install the Google Workspace MCP via DXT

1. Open <https://github.com/taylorwilsdon/google_workspace_mcp/releases>
2. Find the latest release
3. Download the file ending in `.dxt`
4. **Double-click the file**. Claude Desktop opens automatically and shows an Install dialog.
5. Click **Install**

### 1.3 Paste credentials into the Extensions GUI

1. In Claude Desktop → **Settings → Extensions → Google Workspace MCP**
2. A form with three fields appears:
   - `GOOGLE_OAUTH_CLIENT_ID` → paste the Client ID from your password manager
   - `GOOGLE_OAUTH_CLIENT_SECRET` → **paste the secret HERE** (the safe place — OS secure store, not a config file)
   - `USER_GOOGLE_EMAIL` → your own work email address
3. Click **Save**
4. **Fully quit Claude Desktop** (Cmd/Ctrl+Q, not just close the window) and reopen

### 1.4 Verify the client ID format (optional, for sanity)

If the user wants to double-check they pasted the right thing, ask them to paste **just the client_id** (not the secret) into chat. Validate:

- Ends with `.apps.googleusercontent.com`
- Has a numeric prefix followed by `-`
- At least 60 chars total

Show only a masked version back: first 12 chars + `...` + `.apps.googleusercontent.com`. Never show the middle.

If anything looks off, tell them to double-check with their admin before proceeding.

### 1.5 First-run consent

Tell the user:

> *"Now run `/pto-system:check-setup` (or `/email-classify:check-setup`). The first time, your browser opens a Google sign-in page. Sign in with **your own work email**. You'll see a screen asking for permission to access Gmail / Sheets — click **Allow**. The page then closes itself, and you're done."*
>
> *Because your company set this up as an Internal app, this consent is permanent — you won't have to re-authorize ever (until your admin rotates the secret or you switch computers).*

If their org actually used `External + Testing` mode (you can't tell from inside the chat), they'll see a "Google hasn't verified this app" warning. Tell them: *"That warning is normal because your company hasn't gone through Google's verification process. Click **Advanced → Continue**. Your data is safe — this is YOUR company's app, not a third party."*

That's the entire fast path. Stop here.

---

## Path 2 — Self-setup (full path, ~10 minutes)

The user has no company-provided credentials. Walk them through Google Cloud Console.

Tell them up front: *"I'll guide you click-by-click. You'll need a Google account. We'll create a Google Cloud project, enable three APIs, set up the consent screen, and create OAuth credentials. About 10 minutes. Ready?"*

Wait for confirmation.

### 2.1 Create or pick a Google Cloud project

Link: <https://console.cloud.google.com/projectcreate>

1. Project name: anything — suggest `claude-workspace`
2. Leave Organization alone if not visible
3. Click **Create**, wait ~30 seconds for the green tick

Confirm: *"Are you in the project? What's the project ID showing in the top bar?"*

### 2.2 Enable the three APIs

Send these one at a time. For each, click **Enable**:

1. Gmail API: <https://console.cloud.google.com/apis/library/gmail.googleapis.com>
2. Google Sheets API: <https://console.cloud.google.com/apis/library/sheets.googleapis.com>
3. Google Drive API: <https://console.cloud.google.com/apis/library/drive.googleapis.com>

(Drive is needed because Sheets internally uses Drive metadata.)

Confirm each: *"Do you see 'API enabled' in green?"*

### 2.3 Configure the OAuth consent screen

Link: <https://console.cloud.google.com/apis/credentials/consent>

1. **User Type**: choose **Internal** if their email is part of a Google Workspace org (no verification needed, tokens permanent). Otherwise **External** + Testing (note: tokens expire every 7 days).
2. App name: `Claude Workspace`
3. User support email + Developer contact: their own email
4. **Save and Continue** through Scopes (don't add scopes here)
5. If they chose External: **Test users** → **Add Users** → their own email → **Save**
6. Back to Dashboard

Tell them: *"Internal mode means only people in your Google Workspace org can authorize this app — and you don't need Google's verification. If you're using personal Gmail you had to pick External + Testing instead, and your token will expire every 7 days."*

### 2.4 Create the OAuth Client ID

Link: <https://console.cloud.google.com/apis/credentials>

1. **Create Credentials** → **OAuth client ID**
2. **Application type: `Desktop app`** ← critical
3. Name: `Claude Desktop`
4. Click **Create**
5. A dialog pops up with **Client ID** and **Client secret**

⚠️ **Sensitive moment.** Tell the user, exactly:

> *"You'll see two values: Client ID and Client secret. The Client ID is fine to share with me — paste it into the chat so I can verify the format. The **Client secret is private** — DO NOT paste it here. Just download the JSON file (button on the dialog) and keep it safe. I'll tell you exactly where to paste the secret in a moment."*

### 2.5 Verify the client ID format

When the user pastes the client ID, validate (same rules as Path 1.4):
- Ends with `.apps.googleusercontent.com`
- Numeric prefix + `-`
- At least 60 chars

Show masked confirmation only. If it looks wrong, ask them to re-copy.

If they paste something that looks like the secret instead (long random string without `.apps.googleusercontent.com`), apply security rule #2 immediately.

### 2.6 Install Google Workspace MCP via DXT

Same as Path 1.2:

1. Open <https://github.com/taylorwilsdon/google_workspace_mcp/releases>
2. Download the `.dxt` file
3. Double-click → **Install**

### 2.7 Paste credentials into the Extensions GUI

Same as Path 1.3:

1. **Settings → Extensions → Google Workspace MCP**
2. Paste Client ID, Client secret, and email into the form
3. Save
4. Quit and reopen Claude Desktop fully

### 2.8 First-run consent

Tell them: *"Type a `/` in chat. You should see `/pto-system:sync-pto` etc. Run `/pto-system:check-setup`. Browser opens, you sign in with your Google account, see 'Google hasn't verified this app' (expected — it's YOUR app), click **Advanced → Continue → Allow**. Done."*

### 2.9 Post-setup security

Paste this for them:

> **Keep your credentials safe**
> - Move the `credentials.json` from Downloads to a password manager (1Password, Bitwarden, etc.). Delete the original.
> - Never paste the secret into any chat (even with me — I never need it again).
> - If you suspect a leak: <https://console.cloud.google.com/apis/credentials> → click your OAuth client → **Reset Secret** → update the new value via Claude Desktop → Settings → Extensions → Google Workspace MCP.

---

## Common pitfalls (both paths)

- **Tokens expire after 7 days** → user is on `External + Testing`. Switch to Internal (Workspace org) or accept weekly re-auth.
- **"This app isn't verified" page** → expected for self-setup or unverified company apps. Click **Advanced → Continue**.
- **DXT install fails with "couldn't open"** → Claude Desktop too old. Update it.
- **`/...:check-setup` says ready: false / reason: "auth"** → user hasn't completed the Google consent flow yet. Trigger it by retrying.
- **Path 1 user pastes secret in chat anyway** → security rule #2 verbatim. Do not soften the message. The cost of a leaked secret to the company is much higher than a moment of awkwardness.
