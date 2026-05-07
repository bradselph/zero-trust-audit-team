# Zero-Trust Audit Team

A Claude Code plugin that turns your existing codebase into a fully-tracked improvement pipeline: **plan -> audit -> triage -> fix -> test -> re-verify**, with persistent state that survives `/clear` and context resets.

## Install

```
/plugin install https://github.com/bradselph/zero-trust-audit-team
```

Or load locally for testing:

```bash
git clone https://github.com/bradselph/zero-trust-audit-team
claude --plugin-dir ./zero-trust-audit-team
```

## Quickstart

```
/audit:init src/
/audit:run
/audit:triage
/audit:fix
/audit:summary
```

## Commands

| Command | Description |
|---|---|
| `/audit:init [paths]` | Define scope, build the file manifest |
| `/audit:run` | Audit files one-by-one (stops at `STATUS: PARTIAL`) |
| `/audit:continue` | Resume a paused audit at the last resume marker |
| `/audit:triage` | Convert raw findings into a prioritized fix plan |
| `/audit:fix [FND-id]` | Run one finding through fix -> test -> re-verify |
| `/audit:status` | Snapshot: coverage %, open findings by severity, next action |
| `/audit:summary` | Final report (only valid after full manifest coverage) |

## The team

Eight specialized agents coordinate through a shared state ledger in `.claude/audit-state/`. Each has a narrow, evidence-enforced role:

| Agent | Role | Writes code? |
|---|---|---|
| `audit-orchestrator` | Coordinator -- dispatches all other agents, owns state | State files only |
| `audit-planner` | Builds the file manifest, defines scope | No |
| `code-auditor` | Zero-trust per-file verifier -- one file per invocation, explicit evidence required | No |
| `triage-analyst` | Dedupes findings, scores by severity x confidence x blast radius | No |
| `fix-implementer` | Applies one triaged fix at a time -- minimal change, no scope creep | Yes |
| `test-engineer` | Runs existing tests; writes a regression test per fix | Tests only |
| `re-verifier` | Independent re-audit of the changed region -- confirms or rejects with evidence | No |
| `docs-reconciler` | Resolves code/doc contradictions flagged by the auditor | Docs only |

## The flow

```
/audit:init   ->  define scope, build manifest
/audit:run    ->  code-auditor audits files one-by-one, writes findings/FND-*.json
/audit:triage ->  triage-analyst ranks and groups findings into a fix plan
/audit:fix    ->  fix-implementer -> test-engineer -> re-verifier (one finding per run)
/audit:summary -> final report: coverage table, themes, unresolved inventory
```

Each phase is a separate command. Nothing happens automatically between phases -- you stay in control of when to move forward.

## State layout

All state lives in `.claude/audit-state/` in your project as plain JSON and Markdown -- diffable, reviewable, readable without tooling.

```
.claude/audit-state/
+-- scope.json           IN_SCOPE, OUT_OF_SCOPE, LANGUAGES, sensitive_paths
+-- manifest.json        Every in-scope file: path, size, line count, audit order
+-- coverage.json        Per-file status (not-started / PARTIAL / COMPLETE) + rollups
+-- findings/            FND-0001.json, FND-0002.json, ... -- one file per finding
+-- triage.json          Ordered remediation plan with scores and fix-unit groupings
+-- log/
|   +-- audit-<file>.md  Per-file execution traces from code-auditor
|   +-- fix-FND-NNNN.md  Fix record: before/after + test results + re-verifier verdict
+-- README.md            Schema reference for every state file
```

The `audit-state/README.md` schema reference is included in this plugin for reference. The orchestrator writes it to your project during `/audit:init`.

## Design choices

**Why one file per audit invocation.** The `code-auditor` audits exactly one file per call (or emits `STATUS: PARTIAL` with a resume line when the file exceeds the context window). This makes `/audit:run` resumable across multiple sessions on any repo size.

**Why findings are files, not chat history.** Chat transcripts don't survive `/clear`. `findings/FND-0042.json` does. Every agent reads from this ledger days later with full fidelity.

**Why one finding per fix invocation.** Batching unrelated fixes is where regressions hide. Each fix-unit gets its own complete trace: locate -> edit -> lint/typecheck -> regression test -> independent re-verification.

**Why the auditor cannot write.** The `code-auditor` has read-only tools. It literally cannot fix what it finds -- no incentive to downplay findings for a cleaner diff.

**Why re-verification is a separate agent.** The agent that applied the fix is the wrong agent to certify it. The `re-verifier` starts from a clean context and must produce an independent execution trace.

## Customizing

**Severity thresholds** -- edit `agents/triage-analyst.md`, step 5 (auto-fix eligibility rules).

**Per-language hazard checklists** -- create `.claude/audit-state/appendix/<language>.md` in your project. The `code-auditor` picks it up during the hazard check pass.

**Sensitive path overrides** -- add `sensitive_paths` to `scope.json`. The triage-analyst routes anything matching those paths to human review.

**Pre-existing findings** -- drop JSON into `.claude/audit-state/findings/` matching the schema in `audit-state/README.md`. The triage-analyst folds them into the plan.

## When not to use this

- **Single-file fixes**: overkill -- use plain Claude Code.
- **Greenfield projects**: the auditor's value is on accumulated code with real issues.
- **If you want instant results**: the chunking protocol is intentional. Each phase is checkpointed so nothing is lost if context resets.
