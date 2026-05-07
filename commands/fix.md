---
description: Process one fix-unit through implementer -> test-engineer -> re-verifier.
argument-hint: [FND-id | next]
allowed-tools: Read Grep Glob Edit Write Bash
---

# /audit:fix

Apply one fix-unit and verify it independently. Dispatch `@audit-orchestrator`.

The user's target: `$ARGUMENTS` (a specific `FND-NNNN`, the word `next`, or empty which means `next`)

## Orchestrator, on receipt:

1. **Preflight.** Confirm `.claude/audit-state/triage.json` exists. If not, instruct the user to run `/audit:triage` first.

2. **Resolve the target fix-unit.**
   - If `$ARGUMENTS` is a valid `FND-NNNN`: find the fix-unit in `triage.json.plan` that contains it.
   - If `$ARGUMENTS` is empty or `"next"`: pick the first fix-unit in `plan` whose findings all have `status: "triaged"` (i.e., not yet started).
   - If no eligible unit exists: report "all planned fixes complete or in progress" and stop.

3. **Human-review gate.** If the fix-unit has `requires_human_review: true`:
   - Display the full fix-unit (findings, rationale, affected files)
   - Ask the user: "This fix-unit requires human review. Approve for auto-fix? (yes/no, or 'defer' to leave in queue)"
   - Do not proceed without explicit `yes`. On `no` or `defer`, mark status accordingly and stop.

4. **Route by type.**
   - If all findings in the unit are `type: "doc-drift"`: route to `@docs-reconciler`
   - Otherwise: route to `@fix-implementer`

5. **Dispatch fix:**

   > `@fix-implementer` -- apply fix for fix-unit covering findings `<FND-NNNN, FND-MMMM, ...>`. Work through them in the unit's declared order. Minimal diff, no scope creep.

   (or `@docs-reconciler` with the same payload for doc-drift)

6. **On implementer return:**
   - Verify the fix log was written
   - Verify every targeted finding's status is now `"fixed"`
   - Verify no unintended files were modified (compare `git diff` file list against the fix-unit's declared files)
   
   If any verification fails: report the gap, set findings back to `"triaged"`, and stop -- do not proceed to testing.

7. **Dispatch test-engineer** (skip this step for `docs-reconciler` fixes):

   > `@test-engineer` -- write regression test + run suite for fix at `log/fix-FND-NNNN.md`.

8. **On test-engineer return:**
   - Verify the fix log has a Tests section
   - If `test_status: "regression"`: stop. Report the regression, set findings back to `"in-progress"`, tell the user the implementer must re-do. Do **not** dispatch re-verifier.
   - If `test_status: "passing"`, `"manual"`, or `"not-applicable"`: proceed to step 9.

9. **Dispatch re-verifier:**

   > `@re-verifier` -- independently verify fix for `<FND-NNNN>`. Finding JSON: `findings/FND-NNNN.json`. Fix log: `log/fix-FND-NNNN.md`.

10. **On re-verifier return:**
    - If verdict is `CONFIRM`: findings should now be `"verified"`. Report success.
    - If verdict is `REJECT`: findings are back to `"in-progress"`. Report the rejection reason and the gap the implementer must address.

11. **Report to user:**

    ```
    Fix-unit #<order>: <CONFIRMED | REJECTED | REGRESSION>
      Findings: FND-NNNN, FND-MMMM
      Files changed: <n>
      Tests: <status>
      
      <If CONFIRMED>
      Next: /audit:fix to continue with the next unit, or /audit:status.
      
      <If REJECTED or REGRESSION>
      Gap to close: <description from re-verifier or test-engineer>
      Next: /audit:fix FND-NNNN to retry, or investigate manually.
    ```

## Hard rules

- One fix-unit per `/audit:fix` invocation. Do not loop into the next unit automatically.
- Never skip test-engineer. Even if the fix looks trivial. Even if there's no test framework -- in that case test-engineer produces `test_status: "manual"` and the fix still requires re-verifier approval.
- Never skip re-verifier. The implementer's own claim that the fix works is not sufficient.
- If any step is rejected, do not auto-retry. The user decides whether to retry, investigate, or mark `needs-human`.
