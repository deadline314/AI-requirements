---
name: setup-google-oauth
description: Guide a non-technical user through setting up Google OAuth credentials for the Google Workspace MCP, end to end. Use when the user says they haven't set up Google yet, when google-workspace MCP tool calls fail with auth errors, or when the user invokes /pto-system:setup or /email-classify:setup.
---

# Setup Google OAuth — One-time Setup

You are guiding a **non-technical user** through a one-time Google Cloud setup. They want to use this plugin in **Claude Desktop**. They will likely never edit a config file by hand. Be patient, explicit, and confirm each step before moving on.

## Hard security rules — MUST follow

These rules are non-negotiable. Read them before doing anything else.

1. **NEVER ask the user to paste their `GOOGLE_OAUTH_CLIENT_SECRET` into the chat.** The secret goes ONLY into the Claude Desktop Extensions GUI form, where it is stored in the OS secure store (Keychain on macOS, Credential Manager on Windows). If the user tries to paste it in chat, refuse and explain why.
2. **NEVER repeat, echo, mask-and-show, or summarize a secret you happen to see.** If the user pastes one anyway, respond ONLY with "I deleted that from my view. Please rotate it at <https://console.cloud.google.com/apis/credentials> immediately, then paste the NEW secret only into the Desktop Extensions form."
3. **NEVER write a secret to any file** — not `claude_desktop_config.json`, not `.env`, not anywhere. The DXT installer + Desktop GUI handles storage.
4. **NEVER call any tool with the secret as an argument.** Tools that accept arbitrary strings (including web search, file write, shell) are forbidden to receive the secret value.
5. The `client_id` is **less sensitive** but still PII-adjacent. You may show it masked (first 12 chars + `...` + last 4) to confirm correctness with the user, but never write it to git-tracked files.
6. If at any step the user expresses "I don't know what I'm doing", offer to pause. Never push them to continue past confusion — a confused user is how secrets leak.

## What you will help them do

The end result is: Google Workspace MCP installed in Claude Desktop with the user's OAuth credentials, ready for `pto-system` and `email-classify` plugins to use.

There are two halves:

- **Part A — Google Cloud Console** (~10 min, one-time, unavoidable)
- **Part B — Claude Desktop install** (~2 min, one-time)

After this, **the plugin works for life**. Future runs just open a browser tab once a year for token refresh.

---

## Part A — Google Cloud Console

Tell the user up front: *"I'll guide you click-by-click. You'll need a Google account. We'll create a Google Cloud project, enable three APIs, set up the consent screen, and download a credentials file. Total time: about 10 minutes. Ready?"*

Wait for confirmation before proceeding.

### A1. Create or pick a Google Cloud project

Send the user this clickable link: <https://console.cloud.google.com/projectcreate>

Tell them:
1. Project name: anything they like — suggest `claude-workspace`
2. Leave "Organization" alone if they don't see one
3. Click **Create**, wait ~30 seconds for the green tick, then continue

Ask: *"Are you in the project? What's the project ID showing in the top bar?"* Use this to confirm they're not still on a different project.

### A2. Enable the three APIs

Send these three links one at a time. For each, the user clicks **Enable**:

1. Gmail API: <https://console.cloud.google.com/apis/library/gmail.googleapis.com>
2. Google Sheets API: <https://console.cloud.google.com/apis/library/sheets.googleapis.com>
3. Google Drive API: <https://console.cloud.google.com/apis/library/drive.googleapis.com>

(Drive is needed because Sheets internally uses Drive metadata.)

Confirm each one before moving on: *"Do you see 'API enabled' in green?"*

### A3. Configure the OAuth consent screen

Link: <https://console.cloud.google.com/apis/credentials/consent>

Walk them through:
1. **User Type**: choose **External** (unless they're inside a Google Workspace org and prefer Internal — only then pick Internal)
2. **App name**: `Claude Workspace` (anything)
3. **User support email**: their own email
4. **Developer contact**: their own email again
5. **Save and Continue** through the Scopes screen (don't add scopes here — the MCP requests them at runtime)
6. **Test users** screen → click **Add Users** → add their own Google email → **Save**
7. **Back to Dashboard**

Tell them: *"This screen is what people see when they click 'Allow' during OAuth. Since you added yourself as a test user, only your account will be allowed to authorize — that's exactly what we want."*

### A4. Create the OAuth Client ID

Link: <https://console.cloud.google.com/apis/credentials>

Walk them:
1. Top bar → **Create Credentials** → **OAuth client ID**
2. **Application type: `Desktop app`** ← critical, must be Desktop (not Web)
3. Name: `Claude Desktop` (anything)
4. Click **Create**
5. A dialog pops up with **Client ID** and **Client secret**

⚠️ **HERE IS THE SENSITIVE MOMENT.** Tell the user, in this exact wording:

> *"You'll see two values: Client ID and Client secret. The Client ID is fine to share with me — paste it into the chat so I can verify the format. The **Client secret is private** — DO NOT paste it here. Just download the JSON file (button on the dialog) and keep it safe. I'll tell you exactly where to paste the secret in Part B."*

### A5. Verify the client ID format

When the user pastes the client ID, validate:

- It ends with `.apps.googleusercontent.com`
- It contains a numeric prefix followed by a `-` (e.g. `123456789-abc...apps.googleusercontent.com`)
- It is at least 60 chars long

Show only a masked version back to the user for confirmation: first 12 chars + `...` + last 30 chars (which is the `.apps.googleusercontent.com` ending). Example: `123456789012...xyz.apps.googleusercontent.com`. Never show the middle.

If it doesn't match, ask them to double-check they copied the right field — they may have grabbed the project number or service account ID by mistake.

If they paste anything that looks like the secret instead (long random string without `.apps.googleusercontent.com`), STOP IMMEDIATELY and refer to security rule #2 above.

### A6. They should now have

- ✅ A Google Cloud project with Gmail/Sheets/Drive APIs enabled
- ✅ An OAuth consent screen configured (External, themselves as test user)
- ✅ A `credentials.json` file downloaded (or at minimum, the Client ID and Client secret on screen)

Confirm all three before moving to Part B.

---

## Part B — Claude Desktop install

There are two ways to install the Google Workspace MCP into Claude Desktop. Recommend Option 1 to non-technical users.

### Option 1 — DXT one-click (recommended)

1. Open this URL: <https://github.com/taylorwilsdon/google_workspace_mcp/releases>
2. Find the latest release
3. Download the file ending in `.dxt` (e.g. `google_workspace_mcp.dxt`)
4. **Double-click the file**. Claude Desktop opens automatically and shows an Install dialog.
5. Click **Install**
6. Claude Desktop now has a new entry under **Settings → Extensions → Google Workspace MCP**
7. Click that entry. A form appears with three fields:
   - `GOOGLE_OAUTH_CLIENT_ID` → paste the Client ID
   - `GOOGLE_OAUTH_CLIENT_SECRET` → **paste the Client secret HERE** (this is the safe place — it goes into the OS secure store, not a config file)
   - `USER_GOOGLE_EMAIL` → paste their own Gmail address
8. Click **Save**
9. **Fully quit Claude Desktop** (Cmd/Ctrl+Q, not just close the window) and reopen

### Option 2 — `claude_desktop_config.json` (only if Option 1 doesn't work)

Tell the user: *"If the .dxt file didn't work for you, we can fall back to manual config. This is more error-prone and the secret goes into a plain text file on your computer — try Option 1 first."*

If they insist:
1. In Claude Desktop → **Settings → Developer → Edit Config**
2. They will paste the block from `examples/claude_desktop_config.template.json`, replacing the three placeholders with their values
3. **Crucial**: tell them to make sure that file is NOT in any cloud sync folder (Dropbox, OneDrive, iCloud, Google Drive). The default location (`%APPDATA%\Claude\` on Windows, `~/Library/Application Support/Claude/` on macOS) is safe.
4. Save and restart Claude Desktop

In either option, **never paste secrets into the chat with you**.

---

## Part C — Verify it works

After Claude Desktop restarts:

1. Tell the user: *"Type a `/` in the chat box. You should see `/pto-system:sync-pto` and `/email-classify:classify-emails` in the list. Do you?"*
2. If yes: *"Great — try `/pto-system:check-setup` (or `/email-classify:check-setup`). It will do a read-only test call. The first time, your browser will open with a Google sign-in page. Click your account, see the warning that says 'Google hasn't verified this app' — that's expected because you're using YOUR OWN test app. Click **Advanced → Go to Claude Workspace (unsafe)** → **Continue**. Done."*
3. After consent, the test call should succeed. If it returns labels (Gmail) or sheet info (Sheets), the setup is good.

If the test fails:
- Most common: user picked **Web app** instead of **Desktop app** in step A4. They have to re-create the OAuth client.
- Second most common: forgot to add themselves as test user in step A3.
- Third: typo in the email field. Compare carefully.

Walk them through the fix, do NOT just say "check your settings" — be specific.

---

## Part D — Remembering credentials safely

After setup is done, leave them with this short security checklist (paste it in chat):

> **Keep your credentials safe**
> - The `credentials.json` file you downloaded contains the secret. **Move it to a password manager** (1Password, Bitwarden, LastPass) and delete the original from Downloads.
> - Never commit it to git. Never paste it in any chat with anyone (including me — I do not need it again).
> - If you suspect it leaked, go to <https://console.cloud.google.com/apis/credentials>, find the OAuth client, click **Reset Secret**, then update the value in Claude Desktop → Settings → Extensions → Google Workspace MCP.
> - Token (the file Claude uses to call Google) is stored under your home directory by the MCP server. It is automatically refreshed; you don't need to touch it.

That's it. From this point on, plugins just work.

---

## Common pitfalls to flag proactively

- **"My Google org doesn't allow me to create OAuth apps"** → they're inside a managed Workspace, need IT to allow it or use a personal Google account
- **"Verification needed" pop-up after months of use** → the OAuth consent screen is in Testing mode; tokens expire after 7 days unless they publish the app or use Internal user type
- **"This app isn't verified" page** → expected, click Advanced → Continue
- **DXT install fails with "couldn't open"** → Desktop version too old; ask them to update Claude Desktop
