# Audit State — Schema Reference

All agents read from and write to this directory. Every file is plain JSON or Markdown — diffable, reviewable, and persistent across `/clear`.

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
      "status": "COMPLETE",
      "declared_lines": 142,
      "inspected_lines": 142,
      "functions_analyzed": 8,
      "resume_at": null,
      "finding_ids": ["FND-0001", "FND-0002"]
    },
    "src/auth.ts": {
      "status": "PARTIAL",
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

**Valid `status` values**: `not-started` · `PARTIAL` · `COMPLETE` · `audit-failed`

---

## `findings/FND-NNNN.json`

All fields are required. IDs are zero-padded four-digit integers (`FND-0001`), assigned sequentially.

```json
{
  "id": "FND-0001",
  "file": "src/auth.ts",
  "line_start": 87,
  "line_end": 94,
  "anchor": "if (token == null) return",
  "kind": "bug",
  "type": "silent-failure",
  "severity": "HIGH",
  "confidence": "HIGH",
  "snippet": "<verbatim code, ≥5 lines of context>",
  "trace": "validateToken() called with expired token → line 87 branch taken → function returns undefined instead of false → caller at routes.ts:42 checks truthiness, treats undefined as falsy, proceeds with unauthenticated request",
  "explanation": "validateToken silently returns undefined on expired tokens instead of returning false or throwing. Callers that check truthiness are fooled into treating the failure as success.",
  "impact": "Authentication bypass on any route that calls validateToken() and checks return value with truthy/falsy test rather than strict equality.",
  "status": "open",
  "linked_fix_id": null,
  "related_findings": [],
  "detected_by": "code-auditor",
  "fix_implementer_dissent": null
}
```

**Valid `kind` values**: `bug` · `observation`

**Valid `type` values**: `silent-failure` · `resource-leak` · `concurrency-hazard` · `input-validation` · `insecure-pattern` · `unreachable-code` · `cross-file-mismatch` · `doc-drift` · `redundancy` · `dead-code` · `spec-violation`

**Valid `severity` values**: `CRITICAL` · `HIGH` · `MEDIUM` · `LOW` · `INFO`

**Valid `confidence` values**: `HIGH` · `MEDIUM` · `LOW`

**Valid `status` values**: `open` → `triaged` → `in-progress` → `fixed` → `verified`  
Terminal statuses: `verified` · `wontfix` · `needs-human` · `UNVERIFIED`

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
      "reason": "LOW confidence — external dependency behavior not verifiable from source",
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
File:     not-started → PARTIAL → COMPLETE
                                └→ audit-failed

Finding:  open → triaged → in-progress → fixed → verified
                                      └→ needs-human
                         └→ needs-human
               └→ UNVERIFIED (deferred)
                           └→ wontfix (file deleted only, or human decision)
```
