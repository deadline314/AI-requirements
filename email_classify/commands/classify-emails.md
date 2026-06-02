---
description: Classify recent Gmail messages and apply AI/* labels (contract, sales, support, finance, hr, newsletter, internal)
argument-hint: [days]
---

Run a full email classification pass now.

If the user provided an argument, treat it as the lookback window in days (override `EMAIL_LOOKBACK_DAYS` and `DEFAULT_LOOKBACK_DAYS`). Otherwise use the default (1 day).

Steps:

1. Verify `USER_GOOGLE_EMAIL` is set in the MCP env. If not, ask the user to complete OAuth setup first (see plugin README).
2. Read and execute `${CLAUDE_PLUGIN_ROOT}/skills/email-classify/SKILL.md` end to end.
3. Print the summary report including the unclassified count.

User argument: $ARGUMENTS
