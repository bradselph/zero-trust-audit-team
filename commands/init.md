---
description: Bootstrap the zero-trust audit. Collect scope, build the FILE_MANIFEST, initialize state.
argument-hint: [in-scope-paths-or-globs]
allowed-tools: Read Grep Glob Edit Write Bash
---

# /audit:init

You are starting (or resetting) a zero-trust audit. Dispatch `@audit-orchestrator` to handle this properly.

The user's scope hint: `$ARGUMENTS`

## Orchestrator, on receipt:

1. Check whether `.claude/audit-state/` already contains a manifest. If yes, ask the user whether this is:
   - a **fresh audit** (archive existing state to `.claude/audit-state/archive/<iso-date>/` and start over), or
   - a **re-scope** (update manifest against the existing state, preserving findings where file paths still match).

2. Normalize the scope:
   - If `$ARGUMENTS` is non-empty, treat it as IN_SCOPE. Ask the user for OUT_OF_SCOPE and LANGUAGES.
   - If `$ARGUMENTS` is empty, ask the user for all three: IN_SCOPE, OUT_OF_SCOPE, LANGUAGES.
   - Propose sensible defaults for OUT_OF_SCOPE (node_modules, vendor, dist, build, generated files) and confirm.

3. Once scope is confirmed, dispatch:

   > `@audit-planner` — please build the FILE_MANIFEST for scope <paste confirmed scope>.

4. When the planner returns, verify:
   - `scope.json` exists and matches the confirmed scope
   - `manifest.json` exists with non-zero file count
   - `coverage.json` has been initialized with all files in state `"not-started"`

5. Report to the user:

   ```
   Audit initialized.
   Scope: <languages>, <n> files, <total> lines
   Excluded: <n> files (<reason summary>)
   Order: entry-points → security-sensitive → core → utils → config
   
   Next: run /audit:run to begin.
   ```

6. If any step fails (planner can't run, scope ambiguous, files missing), stop and report exactly what's blocking. Do not proceed with partial initialization.
