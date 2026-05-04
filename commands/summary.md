---
description: Produce the final audit report. Only valid when every manifest file is complete and every finding is in a terminal state.
allowed-tools: Read Grep Glob Write
---

# /audit:summary

Final report generation. Dispatch `@audit-orchestrator`.

## Orchestrator, on receipt:

### 1. Preflight — strict

The summary is premature if **any** of these are true:

- Any file in `coverage.json` has `status` ≠ `"complete"` and ≠ `"audit-failed"` (with `audit-failed` recorded as residual risk)
- Any finding has `status` in: `open`, `triaged`, `in-progress`, `fixed`
  - (`fixed` means awaiting re-verifier — not a terminal state)
- `triage.json` does not exist (nothing has been planned)

If any of these are true, **refuse** to produce the summary. Output:

```
Audit summary is premature. Blocking items:
  - <n> files not yet complete: <list>
  - <n> findings in non-terminal status:
      open:         FND-NNNN, FND-MMMM, ...
      triaged:      ...
      in-progress:  ...
      fixed:        ... (awaiting re-verifier)
  
Next: <appropriate command to close each gap>
```

Do not write a partial report. Do not proceed.

### 2. Write the report

Terminal states are: `verified`, `wontfix`, `needs-human`, `deferred`, `unverified`, plus `audit-failed` files. Every finding must be in one of these. Every file must be `complete` or `audit-failed`.

Write `.claude/audit-state/FINAL_REPORT.md` with the following sections.

#### Section A — Coverage Table

Table format, one row per file, sourced from `coverage.json`:

| File | Declared Lines | Inspected Lines | Functions Analyzed | critical | high | medium | low | info |
|------|----------------|-----------------|--------------------|----------|------|--------|-----|------|
| ...  | ...            | ...             | ...                | ...      | ...  | ...    | ... | ...  |

Totals row at the bottom. Any file with `status: "audit-failed"` gets its row with lines marked `N/A (audit-failed: <reason>)` and no finding counts.

#### Section B — Top Findings

Top 10 findings ranked by: severity (critical > high > ...) → confidence (high > medium > low) → impact (from the finding's `impact` field, judged by blast radius).

For each, emit:

```
### #N — [sev:<level>] [conf:<level>] FND-NNNN

File: <path>:<line>
Anchor: <unique substring>
Status: <verified | wontfix | needs-human | deferred | unverified>

Title: <from finding JSON>
Description: <from finding JSON>
Impact: <from finding JSON>
Resolution: <reference to log/fix-FND-NNNN.md if verified, or the dissent note if needs-human>
```

Only include `verified` findings if the user explicitly requested a fix log; otherwise focus on the unresolved state that future audits will inherit (`needs-human`, `deferred`, `unverified`, `wontfix`).

#### Section C — Cross-Cutting Themes

Draw from `triage.json.plan` entries with `fix_unit: "pattern-cluster"` or `"cross-cutting"`. For each recurring pattern, report:

- The pattern (in the triage-analyst's rationale language)
- Number of instances
- Files affected
- Whether the pattern was resolved (all cluster findings verified) or partially resolved

This section is what makes the audit valuable for organizational learning — single findings fade, patterns persist.

#### Section D — `unverified` / `deferred` Inventory

Every finding with `status: "unverified"` or `status: "deferred"`. For each:

- The finding (file, line, anchor, snippet)
- Exactly what is needed to verify it (from the finding's `needed_to_verify` field, populated by the triage-analyst)
- Who should verify (human with access to: runtime config, production logs, external service, etc.)

Do not try to reclassify these. They are unverifiable under the zero-trust rule; the report says so plainly.

#### Section E — `needs-human` Inventory

Findings awaiting human judgment. For each:

- The finding
- The dissent note from whoever escalated (fix-implementer, re-verifier, triage-analyst, or docs-reconciler)
- The recommended human action

#### Section F — Residual Risk

Items that affected the audit but were out of scope:

- `scope.json.out_of_scope` paths with notes on why each was excluded
- External dependencies flagged during the audit as unverified third-party risk
- Files with `status: "audit-failed"` — what blocked them
- Known limitations of the audit itself (e.g., "dynamic dispatch in <module> could not be fully traced")

#### Section G — Metadata

- Audit started: `scope.json.created_at`
- Audit completed: `<iso8601 now>`
- Tool versions (if known from environment)
- Agent set used (read from `.claude/agents/` — list each agent file + its last-modified date)
- Total findings: <n>, resolved: <r>, unresolved: <u>

This metadata is what makes the report reproducible. A future audit can reference it to know what baseline to compare against.

### 3. Output to chat

```
Final report written: .claude/audit-state/FINAL_REPORT.md
  Coverage: 100% (<n>/<n> files, <m>/<m> lines)
  Resolved: <r> findings  (verified: <v>, wontfix: <w>)
  Unresolved: <u> findings  (needs-human: <h>, deferred: <d>, unverified: <x>)
  Cross-cutting themes: <t>
  
The audit is complete.
```

Do not append summarizing commentary. The report itself is the artifact; the chat message points to it.

## Hard rules

- Never produce a partial summary. If preflight fails, refuse.
- Never fabricate entries. Every row in the coverage table comes from `coverage.json`. Every finding entry comes from a `findings/FND-*.json` file that exists on disk.
- Never reclassify findings in the summary. If a finding is `unverified`, it stays `unverified` in the report — the summary is a factual reconstruction, not a re-triage.
- Never omit `needs-human`, `deferred`, or `unverified` items to make the report look cleaner. Those are the whole point — they tell the human what work remains.
- Do not write the report to `/mnt/user-data/outputs/` or anywhere outside `.claude/audit-state/`. The report belongs to the audit state, versioned with the rest of it.
