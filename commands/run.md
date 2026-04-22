---
description: Run the zero-trust audit across the manifest. One file per turn, stops on STATUS: PARTIAL.
allowed-tools: Read Grep Glob Edit Write Bash
---

# /audit:run

Execute the chunked audit. Dispatch `@audit-orchestrator`.

## Orchestrator, on receipt:

1. **Preflight.** Confirm the following exist and are valid:
   - `.claude/audit-state/scope.json`
   - `.claude/audit-state/manifest.json`
   - `.claude/audit-state/coverage.json`
   
   If any are missing, stop and tell the user to run `/audit:init` first.

2. **Find the next file.** From `coverage.json`, pick the first file whose status is `"not-started"` or `"PARTIAL"`. Respect manifest ordering.
   - If none exists, go to step 5 (all files done).
   - If the file's status is `"PARTIAL"`, the resume line is in `coverage.json.files[file].resume_at`.

3. **Dispatch the auditor.** Pass the target file and resume line (if any):

   > `@code-auditor` — audit file `<path>`. Resume at line `<n>` if continuing. Prior findings directory: `.claude/audit-state/findings/`. Manifest declared lines: `<m>`.

4. **On auditor return:**
   - Verify the auditor output ends with a `STATUS: COMPLETE` or `STATUS: PARTIAL` marker.
   - Verify `coverage.json` was updated for this file.
   - Verify any claimed findings exist as `findings/FND-*.json` files with all required fields.
   - If any of those verifications fail, reject the auditor's output and redispatch with a note about what was missing.
   
   Then:
   - If `STATUS: COMPLETE`: report progress and **loop back to step 2** for the next file. Continue until token budget is tight or all files done.
   - If `STATUS: PARTIAL`: report the resume marker and **stop**. The user will run `/audit:continue` next turn.

5. **All files done.** Report:

   ```
   Audit complete.
   Coverage: 100% (<n>/<n> files, <m>/<m> lines)
   Findings: <C critical> <H high> <M medium> <L low> <I info>
   
   Next: /audit:triage to build the remediation plan.
   ```

## Loop discipline

The orchestrator is allowed to dispatch the auditor multiple times in a single `/audit:run` turn, one file at a time, as long as each invocation returns promptly. If the orchestrator's own output is getting long, it should stop after the current file and tell the user to run `/audit:run` again to continue.

Never dispatch the auditor on multiple files simultaneously. The auditor is designed to process exactly one file per invocation; parallel dispatch defeats the chunking protocol.

## Failure handling

- **Auditor returns without a STATUS marker**: redispatch with "your previous output lacked a STATUS marker; please emit one".
- **Auditor claims COMPLETE but coverage.json wasn't updated**: redispatch.
- **Same file fails twice**: mark it `status: "audit-failed"` in coverage.json with the failure reason, skip to next, and flag in the user-facing summary.
