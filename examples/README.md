# Claude Desktop config — fallback example only

⚠️ **Most users should NOT use this file.** The recommended path is the
MCPB one-click installer (`.mcpb`, formerly `.dxt`) for Google Workspace
MCP — secrets go into Claude Desktop's Extensions form (OS secure
storage), not into a plain text config file.

This fallback example exists only for advanced users whose machines block
MCPB installs. The skill `setup-google-oauth` will guide you to the MCPB
path first.

## If you must use this fallback

1. Copy `claude_desktop_config.example.json` → your actual config location:
   - Windows: `%APPDATA%\Claude\claude_desktop_config.json`
   - macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
2. Replace the three `REPLACE_WITH_*` placeholders with your real values.
3. **Verify the destination directory is NOT in any cloud sync folder**
   (Dropbox, OneDrive, iCloud Drive, Google Drive). The default OS
   locations above are safe — confirm yours is the same.
4. Save and **fully quit + reopen Claude Desktop** (not just close the
   window).
5. **Never commit your real config file to git.** This repo's
   `.gitignore` already excludes `claude_desktop_config.json` and
   `*.local.json`. The example file in this folder is named
   `*.example.json` and contains only placeholders.

## Rotating credentials

If the secret leaks (committed, sent in chat, screenshotted, etc.):

1. Go to <https://console.cloud.google.com/apis/credentials>
2. Click your OAuth client → **Reset Secret**
3. Update the new value via the same path as above (MCPB Extensions form
   or this config file)
4. Restart Claude Desktop
