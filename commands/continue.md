---
description: "Resume a paused audit from the last STATUS: PARTIAL resume marker."
allowed-tools: Read Grep Glob Edit Write Bash
---

# /audit:continue

Resume the audit where it stopped. Dispatch `@audit-orchestrator`.

## Orchestrator, on receipt:

1. Read `coverage.json` and find the file with status `"partial"` (there should be at most one; if more than one, something went wrong — report it).

2. If no `partial` file exists, this command is a no-op. Check whether there are `"not-started"` files:
   - If yes, the user should run `/audit:run` instead. Tell them.
   - If no, the audit is complete. Tell them to run `/audit:triage` or `/audit:summary`.

3. Otherwise, dispatch:

   > `@code-auditor` — resume audit of `<path>` at line `<n>`. Reason for previous pause: `<reason>`. Prior findings directory: `.claude/audit-state/findings/`.

4. On auditor return, same verification as `/audit:run` step 4. After that, this command hands control back — it does not loop into `/audit:run`. The user can chain commands explicitly if they want to.
