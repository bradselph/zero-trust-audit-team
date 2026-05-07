# Privacy Policy

**Zero-Trust Audit Team** is a local Claude Code plugin. It does not collect, transmit, or store any user data externally.

## What this plugin does

- Reads source files in your project directory to perform code analysis
- Writes audit state files to `.claude/audit-state/` within your project directory
- All data stays on your local machine

## What this plugin does NOT do

- No external network requests
- No telemetry or analytics
- No data sent to third-party services
- No user accounts or authentication required beyond your existing Claude Code session

## Data stored

All state is written locally to `.claude/audit-state/` in your project:

- `scope.json`, `manifest.json`, `coverage.json` -- audit configuration and progress
- `findings/FND-*.json` -- code findings from the auditor
- `triage.json` -- prioritized remediation plan
- `log/` -- per-file audit traces and fix records

This data never leaves your machine unless you explicitly commit it to version control. The `.gitignore` included with this plugin excludes all runtime state from git by default.

## Contact

For questions or concerns: https://github.com/bradselph/zero-trust-audit-team/issues
