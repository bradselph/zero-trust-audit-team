---
description: Turn raw audit findings into a prioritized remediation plan.
allowed-tools: Read Grep Glob Edit Write
---

# /audit:triage

Convert findings into an ordered fix plan. Dispatch `@audit-orchestrator`.

## Orchestrator, on receipt:

1. **Preflight.** Confirm:
   - `.claude/audit-state/findings/` contains at least one `FND-*.json` file
   - Every manifest file has status `"complete"` in `coverage.json`
   
   If files remain unaudited, warn the user:
   
   ```
   Warning: <n> files not yet audited. Triage now will plan only against findings from completed files.
   Unfinished: <path1>, <path2>, ...
   
   Proceed anyway? (respond 'yes' to continue, or run /audit:run first)
   ```
   
   Default to waiting for user confirmation unless they explicitly passed a flag.

2. **Check for prior triage.** If `triage.json` exists:
   - If every finding listed in the existing plan still has `status: "triaged"` or earlier, this is a re-triage -- archive the old `triage.json` to `triage-<iso-date>.json` and build fresh.
   - If findings have moved to `fixed`/`verified`, merge: preserve completed items, re-triage only the open ones.

3. **Dispatch:**

   > `@triage-analyst` -- build the remediation plan from all findings in `.claude/audit-state/findings/`. Scope context: `.claude/audit-state/scope.json`.

4. **On analyst return:**
   - Verify `triage.json` was written and conforms to schema.
   - Verify every non-archived finding is accounted for (in `plan`, `deferred`, or `malformed`).
   - Verify findings now show `status: "triaged"` (except those explicitly deferred).
   
   Reject and redispatch if verifications fail.

5. **Report:**

   ```
   Triage complete.
   
   Plan: <n> fix-units covering <m> findings
     Auto-fix eligible: <a>
     Human review required: <b>
   Deferred: <c> findings (see triage.json.deferred)
   Malformed: <d> findings (see triage.json.malformed)
   
   Top 3 by score:
   #1  [<sev>x<conf>] <fix_unit>: FND-NNNN -- <one-line>
   #2  ...
   #3  ...
   
   Next:
     /audit:fix           -> process the top item
     /audit:fix FND-NNNN  -> jump to a specific finding
     /audit:status        -> see full open list
   ```

## Edge cases

- **Zero findings**: the audit produced nothing to triage. Report that with coverage stats, congratulate the user, and suggest `/audit:summary`.
- **All findings are `info` observations**: still produce a plan, but flag that auto-fix is inappropriate for observations -- they should be human-reviewed and optionally batched into a cleanup pass.
- **All findings are `unverified` or `low` confidence**: the plan will be empty with everything deferred. Report what's needed to un-defer each.
