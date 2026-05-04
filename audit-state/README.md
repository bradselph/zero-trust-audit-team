# Audit State — Schema Reference

All agents read from and write to this directory. Every file is plain JSON or Markdown — diffable, reviewable, and persistent across `/clear`.

> **Schema policy.** The templates below are the *only* legal shapes. Field names, casing, and enums are normative. Agents must copy these templates verbatim and fill in values — they must not invent fields, rename fields, or change casing. Two real-world deployments produced 12 schema-drift bugs between them; this document is the response.

---

## `scope.json`

```json
{
  "created_at": "<iso8601>",
  "in_scope": ["src/", "internal/"],
  "out_of_scope": ["vendor/", "node_modules/", "dist/", "*.min.js"],
  "languages": ["go", "typescript"],
  "sensitive_paths": ["**/auth/**", "**/crypto/**", "**/payment*"],
  "notes": "optional — exclusion rationale, edge cases"
}
```

---

## `manifest.json`

```json
{
  "built_at": "<iso8601>",
  "notes": "audit order rationale",
  "files": [
    {
      "order": 1,
      "path": "src/main.ts",
      "bytes": 4321,
      "lines": 142,
      "reason": "entry point"
    }
  ]
}
```

---

## `coverage.json`

```json
{
  "files_total": 12,
  "files_reviewed": 4,
  "lines_total": 2143,
  "lines_reviewed": 587,
  "coverage_pct": 27.4,
  "files": {
    "src/main.ts": {
      "status": "complete",
      "declared_lines": 142,
      "inspected_lines": 142,
      "functions_analyzed": 8,
      "resume_at": null,
      "finding_ids": ["FND-0001", "FND-0002"]
    },
    "src/auth.ts": {
      "status": "partial",
      "declared_lines": 430,
      "inspected_lines": 210,
      "functions_analyzed": 5,
      "resume_at": 211,
      "finding_ids": ["FND-0003"],
      "resume_reason": "token-limit"
    },
    "src/utils.ts": {
      "status": "not-started",
      "declared_lines": 98,
      "inspected_lines": 0,
      "functions_analyzed": 0,
      "resume_at": null,
      "finding_ids": []
    }
  }
}
```

**Valid `status` values** (lowercase): `not-started` · `partial` · `complete` · `audit-failed`

**Top-level shape is an object, not an array.** Per-file entries are keyed by path under `files`. Do not flatten to `[ {path, status}, ... ]`.

---

## `findings/FND-NNNN.json`

All fields are required (use `null` for absent values, not omission). IDs are zero-padded four-digit integers (`FND-0001`), assigned sequentially.

```json
{
  "id": "FND-0001",
  "file": "src/auth.ts",
  "line": 87,
  "end_line": 94,
  "anchor": "if (token == null) return",
  "type": "silent-failure",
  "severity": "high",
  "confidence": "high",
  "title": "validateToken returns undefined on expired tokens",
  "description": "validateToken silently returns undefined on expired tokens instead of returning false or throwing. Callers that check truthiness are fooled into treating the failure as success.",
  "impact": "Authentication bypass on any route that calls validateToken() and checks return value with truthy/falsy test rather than strict equality.",
  "snippet": "<verbatim code, ≥5 lines of context>",
  "trace": "validateToken() called with expired token → line 87 branch taken → function returns undefined instead of false → caller at routes.ts:42 checks truthiness, treats undefined as falsy, proceeds with unauthenticated request",
  "status": "open",
  "linked_fix_id": null,
  "related_findings": [],
  "detected_by": "code-auditor",
  "fix_implementer_dissent": null
}
```

**Field notes:**
- `line` is the start line (single-line findings: `end_line` equals `line`). Do not use `line_start`/`line_end` — those names were the v1.0 spec but no real run ever used them.
- `title` is a one-sentence, human-scannable summary. `description` is the full explanation. `impact` is the concrete consequence. Three distinct fields, each with one job.
- The v1.0 `kind` field (`bug` / `observation`) is removed. `severity: "info"` encodes observations; everything else is a bug.

**Valid `type` values** (preferred — kebab-case, prefer reusing values seen in prior findings):
- Behavior bugs: `silent-failure` · `logic-error` · `unreachable-code` · `dead-code` · `redundancy`
- Resources & concurrency: `resource-leak` · `concurrency-hazard` · `race-condition` · `toctou`
- Security primitives: `input-validation` · `insecure-pattern` · `injection` · `buffer-overflow` · `null-deref` · `crypto` · `path-traversal` · `directory-hijack`
- Cross-cutting: `cross-file-mismatch` · `doc-drift` · `spec-violation` · `incomplete-feature`

If none of the above fit, you may introduce a new kebab-case type — but check `findings/` first for an existing match. The triage-analyst clusters by `type`, so synonym proliferation hurts.

**Valid `severity` values** (lowercase): `critical` · `high` · `medium` · `low` · `info`

**Valid `confidence` values** (lowercase): `high` · `medium` · `low`

**Valid `status` values** (lifecycle): `open` → `triaged` → `in-progress` → `fixed` → `verified`
**Terminal:** `verified` · `wontfix` · `needs-human` · `deferred` · `unverified`

- `deferred`: triage decided this finding cannot be acted on now (low confidence, blocked on external info). Distinct from `unverified`, which means the auditor itself could not finish verifying.
- `wontfix`: only humans set this, except for one case — `fix-implementer` may set `wontfix` if the file no longer exists (`wontfix_reason: "file deleted since audit"`).
- `needs-human`: the agents disagree, the fix is risky, or the finding is in a `sensitive_paths` location.

**`detected_by` values**: `code-auditor` · `fix-implementer` · `re-verifier`

---

## `triage.json`

```json
{
  "built_at": "<iso8601>",
  "total_findings": 18,
  "plan": [
    {
      "order": 1,
      "fix_unit": "single-file",
      "finding_ids": ["FND-0001"],
      "rationale": "Authentication bypass with high confidence. Isolated to validateToken(). Single line change (return false instead of return).",
      "score": {
        "severity": 12,
        "confidence": 6,
        "blast_radius": 4,
        "simplicity": 3,
        "total": 25
      },
      "estimated_risk": "low",
      "requires_human_review": false,
      "suggested_agent": "fix-implementer"
    },
    {
      "order": 2,
      "fix_unit": "pattern-cluster",
      "finding_ids": ["FND-0007", "FND-0012", "FND-0019"],
      "rationale": "Same swallowed-exception pattern across three HTTP handlers. Fix together to prevent divergence.",
      "score": { "severity": 9, "confidence": 6, "blast_radius": 3, "simplicity": 3, "total": 21 },
      "estimated_risk": "medium",
      "requires_human_review": false,
      "suggested_agent": "fix-implementer"
    }
  ],
  "deferred": [
    {
      "finding_ids": ["FND-0005"],
      "reason": "low confidence — external dependency behavior not verifiable from source",
      "needed_to_verify": "Access to redis-client source or integration test that exercises the timeout path"
    }
  ],
  "malformed": [
    {
      "finding_id": "FND-0011",
      "missing_fields": ["anchor", "trace"]
    }
  ]
}
```

**Valid `fix_unit` values**: `single-file` · `pattern-cluster` · `cascade` · `cross-cutting`

---

## `log/audit-<sanitized-path>.md`

Written by `code-auditor` per file. File path sanitized: `/` → `-`, `.` → `-`.

Example path: `log/audit-src-auth-ts.md`

Content: execution traces, finding summaries, STATUS marker. See `code-auditor` agent for the exact format.

---

## `log/fix-FND-NNNN.md`

Written by `fix-implementer`, then appended to by `test-engineer` and `re-verifier`.

Sections (in order of authorship):
1. **Fix Record** — original finding summary, change plan, edits applied, sanity checks (`fix-implementer`)
2. **Tests** — regression test code, Phase A/B results (`test-engineer`)
3. **Re-Verifier Verdict** — CONFIRM/REJECT with execution trace (`re-verifier`)

---

## `FINAL_REPORT.md`

Written by the orchestrator on `/audit:summary`. Only exists when the audit is complete. Do not write to this file manually — its contents are a factual reconstruction from the state files above.

---

## State transitions (summary)

```
File:     not-started → partial → complete
                                └→ audit-failed

Finding:  open → triaged → in-progress → fixed → verified
                                      └→ needs-human
                         └→ needs-human
                         └→ deferred       (triage cannot act now)
               └→ unverified                (auditor could not finish)
                                            wontfix (humans, or file-deleted)
```

---

## Migration from v1.0 schema

If you are resuming an audit that was started before v1.1, the state files may have these legacy shapes. The orchestrator should normalize on first read:

| v1.0 | v1.1 |
|---|---|
| `line_start`, `line_end` | `line`, `end_line` |
| `kind: "bug"` / `kind: "observation"` | drop field; severity already encodes it |
| `explanation` | merge into `description` |
| `severity: "HIGH"` | `severity: "high"` |
| `confidence: "HIGH"` | `confidence: "high"` |
| coverage `status: "COMPLETE"` | `status: "complete"` |
| coverage `status: "PARTIAL"` | `status: "partial"` |
| finding `status: "UNVERIFIED"` | `status: "unverified"` |
