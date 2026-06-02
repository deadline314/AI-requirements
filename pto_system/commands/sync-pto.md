---
description: Sync PTO requests from Slack to Google Sheet (incremental, approval-aware)
argument-hint: [days]
---

Run a full PTO sync now.

If the user provided an argument, treat it as the lookback window in days (override `PTO_LOOKBACK_DAYS`). Otherwise use the default from `pto-sync`.

Steps:

1. Verify the required environment variables are set: `PTO_SLACK_CHANNEL_ID`, `PTO_GOOGLE_SHEET_ID`, `PTO_GOOGLE_SHEET_TAB`. If any is missing, ask the user once and remember for this session.
2. Read and execute `${CLAUDE_PLUGIN_ROOT}/skills/pto-sync/SKILL.md` end to end.
3. Print the summary report.

User argument: $ARGUMENTS
