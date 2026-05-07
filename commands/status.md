---
description: Show current audit progress, open findings by severity, and next recommended action.
allowed-tools: Read Grep Glob
---

# /audit:status

Read-only snapshot. Dispatch `@audit-orchestrator` but instruct it to dispatch **no specialists** -- pure state read.

## Orchestrator, on receipt:

1. **Existence check.** For each file, note whether it exists:
   - `.claude/audit-state/scope.json`
   - `.claude/audit-state/manifest.json`
   - `.claude/audit-state/coverage.json`
   - `.claude/audit-state/triage.json`
   - `.claude/audit-state/findings/` (count files)
   - `.claude/audit-state/log/` (count files)
   
   If the state directory is entirely missing or empty, report "No audit initialized" and tell the user to run `/audit:init`.

2. **Coverage snapshot.** From `coverage.json`:
   - `files_reviewed / files_total`
   - `lines_reviewed / lines_total`
   - `coverage_pct`
   - Count of files in each status: `not-started`, `partial`, `complete`, `audit-failed`
   - List any files with `status: "partial"` and their resume lines

3. **Findings snapshot.** Enumerate `findings/FND-*.json`:
   - Count by severity: `critical`, `high`, `medium`, `low`, `info`
   - Count by confidence: `high`, `medium`, `low`
   - Count by status: `open`, `triaged`, `in-progress`, `fixed`, `verified`, `needs-human`, `wontfix`, `deferred`, `unverified`
   - Count of `doc-drift` findings (they route differently)

4. **Triage snapshot** (if `triage.json` exists):
   - Total fix-units in plan
   - Fix-units requiring human review
   - Fix-units completed (all findings `verified`)
   - Fix-units in progress
   - Deferred count
   - Malformed count

5. **Determine next recommended action** using this decision tree:

   ```
   No scope.json / manifest.json?           -> /audit:init
   Any file with status = "partial"?        -> /audit:continue
   Any file with status = "not-started"?    -> /audit:run
   Files complete but no triage.json?       -> /audit:triage
   Triage exists with unverified fix-units? -> /audit:fix
   Everything verified / deferred?          -> /audit:summary
   ```

6. **Output format:**

   ```
   +- Audit Status ----------------------------------------
   |
   |  Coverage: <Y>/<X> files | <Z>/<total> lines | <pct>%
   |    +- complete:    <n>
   |    +- partial:     <n>  (resume: <path>:<line>)
   |    +- not-started: <n>
   |    +- failed:      <n>
   |
   |  Findings: <total>
   |    Severity:    <c critical>  <h high>  <m medium>  <l low>  <i info>
   |    Confidence:  <h high>  <m medium>  <l low>
   |    Status:
   |      open:         <n>
   |      triaged:      <n>
   |      in-progress:  <n>
   |      fixed:        <n>  (awaiting re-verifier)
   |      verified:     <n>  OK
   |      needs-human:  <n>  !
   |      wontfix:      <n>
   |      deferred:     <n>
   |      unverified:   <n>
   |
   |  Triage: <n> fix-units
   |    Auto-fix eligible: <a>
   |    Human review:      <b>
   |    Complete:          <c>
   |    Deferred:          <d>
   |
   |  Next: <recommended command>
   |
   +-------------------------------------------------------
   ```

   If any `needs-human` findings exist, list them explicitly below the box with their dissent notes. These are the items blocked on the user's judgment and should not get buried.

## Hard rules

- This command never modifies state. `allowed-tools` excludes `Edit`, `Write`, `Bash`.
- This command never dispatches specialist agents. The orchestrator reads state directly.
- If state files are malformed (invalid JSON, missing required fields), report the specific file and field -- do not silently skip. Corrupt state is itself a status worth surfacing.
- If finding counts across `coverage.json`, `findings/`, and `triage.json` disagree, report the discrepancy. Do not pick a winner -- let the user investigate.
