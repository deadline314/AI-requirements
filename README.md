# AI Requirements — Claude Plugins

Two Claude plugins for Claude Desktop, designed to be used by **non-technical users**.
Once installed, you just chat with Claude — it walks you through the rest.

| Plugin | What it does |
| --- | --- |
| `pto-system` | Reads PTO/leave requests from a Slack channel, detects manager approval in thread replies, and writes the result into a Google Sheet incrementally. Asks you to confirm before any sheet write. |
| `email-classify` | Classifies recent Gmail messages into business categories (`contract`, `sales`, `support`, `finance`, `hr`, `newsletter`, `internal`) and applies `AI/*` Gmail labels. |

## How to install

1. **Open Claude Desktop** → left sidebar → **Customize** → **Plugins** tab.
2. Under **Personal plugins**, click `+` → **Add marketplace** → **Add from a repository**.
3. Paste this repo's GitHub URL and click **Add**.
4. From the marketplace, click **Install** on `pto-system` and on `email-classify`.
5. Add Slack: **Customize → Connectors → Slack → Connect**.
6. **Restart Claude Desktop** fully (Cmd/Ctrl+Q, then reopen).

## How to set up Google (one-time)

This plugin uses Google credentials your **admin has already provisioned**
for the team (one shared Google Cloud project, one shared OAuth Desktop
client). You don't need to touch Google Cloud Console — you just paste the
credentials your admin gave you into Claude Desktop.

In any Claude Desktop chat, type:

```
/pto-system:setup
```

Claude walks you through ~3 minutes of setup:

1. Confirm you can see the `client_id` and `client_secret` in the company
   password manager (1Password / Bitwarden / Dashlane / etc.)
2. Download the Google Workspace MCP `.dxt` installer and double-click it
3. Paste the credentials into **Settings → Extensions → Google Workspace
   MCP** (this is OS secure storage, not a config file, not chat)
4. Restart Claude Desktop and run `/pto-system:check-setup` — your browser
   opens once for Google consent, then you're done forever.

The setup is **shared with `email-classify`**, so you only do it once.

You will **never need to paste a secret into chat**. Claude is hard-coded
to refuse and instructs you where it actually goes (Claude Desktop's
Extensions form). See [Security](#security).

### For the team admin (one-time, before sharing the repo)

If you are the person provisioning credentials for the team, do this once:

1. **Create a Google Cloud project** in your company's Workspace org
   (must be Workspace, not personal Gmail, so you can use Internal mode).
2. **Enable** Gmail API, Google Sheets API, and Google Drive API.
3. **OAuth consent screen** → **Internal** user type → fill App name +
   support email + developer contact → don't add scopes here. Internal
   mode means tokens never expire and you skip Google's verification.
4. **Credentials** → Create OAuth client ID → **Desktop app** → name it
   `Claude Desktop`. Download the JSON.
5. **Move** `client_id` and `client_secret` into a shared password
   manager vault (1Password / Bitwarden / Dashlane). Delete the
   downloaded JSON from your machine.
6. **Share** that vault with teammates who should have the plugin.
7. Distribute this repo's GitHub URL to them. They run
   `/pto-system:setup` inside Claude Desktop and follow the guide.

Never share `client_secret` over Slack DM, email, Notion, SMS, or git.
To rotate: <https://console.cloud.google.com/apis/credentials> → Reset
Secret → update the password manager → notify everyone to repaste.

## How to use

Just chat with Claude in Claude Desktop:

| You say | What happens |
| --- | --- |
| `/pto-system:sync-pto` or *"sync PTO from Slack to the sheet"* | Plugin scans Slack, shows you a dry-run preview, and waits for you to choose `1` Apply / `2` Cancel / `3` Review one-by-one. |
| `/pto-system:check-setup` | Runs a read-only health check on the Google connection. Safe anytime. |
| `/email-classify:classify-emails` or *"tag the emails from the last day"* | Plugin scans Gmail, classifies each message, applies `AI/*` labels. Already-labeled messages are skipped. |
| `/email-classify:check-setup` | Read-only health check. |

The first time a plugin actually writes (to Sheet or Gmail labels), Claude
Desktop pops a per-tool permission dialog: `Allow once` / `Allow always` /
`Deny`. Pick `Allow always` after the first successful run if you trust the
flow — the in-chat dry-run preview (PTO) and the strict deny-list (Email)
are still your safety nets.

## Configuration the plugin will ask you for

The plugin asks for these the first time you use it. You don't need to set
them up in advance.

| Plugin | Asks you for |
| --- | --- |
| `pto-system` | Slack channel ID for PTO posts, Google Sheet ID, sheet tab name |
| `email-classify` | Your company email domain (optional; enables the `internal` category) |

## Security

This codebase follows strict no-leak rules:

- **Setup skill never accepts a secret in chat.** If you paste your
  `GOOGLE_OAUTH_CLIENT_SECRET` into a Claude conversation, Claude is
  instructed to refuse and tell you to rotate it.
- **Secrets live in OS secure storage**, not in any file in this repo. The
  recommended setup uses the `.dxt` one-click installer; secrets go into
  Claude Desktop's Extensions form (Keychain on macOS, Credential Manager
  on Windows).
- **`.gitignore` blocks the obvious leakers**: `credentials.json`,
  `token.json`, `claude_desktop_config.json`, `.env*`, `*.key`, etc.
- **Write tools require explicit consent every time** until you click
  `Allow always`. Read tools are pre-allowlisted. Dangerous tools
  (`delete_sheet`, `share_spreadsheet`, `send_gmail_message`,
  `manage_gmail_filter`) are pre-denied — Claude can't call them even if
  asked.
- **The PTO flow has a mandatory dry-run preview** before any write — you
  always see what's about to change and confirm with `1`/`2`/`3`.
- **Email classifier is additive only** — it adds `AI/*` labels, never
  removes existing labels, never reads spam/trash, never sends mail.

If you suspect a credential leak: <https://console.cloud.google.com/apis/credentials>
→ click your OAuth client → **Reset Secret** → update via Claude Desktop's
Extensions form.

## Where each component runs

This is a Claude **plugin**, which means the same source works in three
clients with slightly different feature levels:

| Component | Claude Desktop | Claude Cowork | Claude Code |
| --- | :---: | :---: | :---: |
| Skills (`pto-sync`, `email-classify`, …) | ✅ | ✅ | ✅ |
| Slash commands | ✅ | ✅ | ✅ |
| MCP server (`google-workspace`) | ✅ | ✅ | ✅ |
| Sub-agents (`pto-coordinator`, `email-classifier`) | ❌ greyed out | ✅ | ✅ |
| Hooks | ❌ | ✅ | ✅ |

In Claude Desktop, the **skills carry the entire workflow**, so even without
sub-agents these two plugins work end-to-end.

## Layout

```
AI-requirements/
├── .claude-plugin/marketplace.json
├── README.md
├── examples/
│   ├── README.md
│   └── claude_desktop_config.example.json    # fallback only; use DXT instead
│
├── pto_system/
│   ├── .claude-plugin/plugin.json
│   ├── .mcp.json                              # for Claude Code; Desktop uses DXT
│   ├── settings.json                          # allow / ask / deny per tool
│   ├── skills/
│   │   ├── setup-google-oauth/SKILL.md        # one-time setup wizard
│   │   ├── preflight-check/SKILL.md           # health check
│   │   ├── pto-sync/SKILL.md                  # main flow
│   │   ├── pto-parse-message/SKILL.md
│   │   └── pto-detect-approval/SKILL.md
│   ├── agents/pto-coordinator.md              # Cowork / Code only
│   └── commands/
│       ├── setup.md
│       ├── check-setup.md
│       └── sync-pto.md
│
└── email_classify/
    ├── .claude-plugin/plugin.json
    ├── .mcp.json
    ├── settings.json
    ├── skills/
    │   ├── setup-google-oauth/SKILL.md        # references pto-system's copy
    │   ├── preflight-check/SKILL.md
    │   ├── email-classify/SKILL.md
    │   ├── email-categorize-rules/SKILL.md
    │   └── email-label-apply/SKILL.md
    ├── agents/email-classifier.md
    └── commands/
        ├── setup.md
        ├── check-setup.md
        └── classify-emails.md
```

## Why a custom Google MCP?

Claude's built-in Google connector is **read-only**. It cannot create labels
or write to a sheet. Both of these are core to what these plugins do, so we
use [`taylorwilsdon/google_workspace_mcp`](https://github.com/taylorwilsdon/google_workspace_mcp)
— a community MCP that exposes the full Gmail and Sheets API surface.

The setup skill recommends installing it via the `.dxt` one-click installer
because that flow puts your secret into OS secure storage instead of a
plain-text JSON file.
