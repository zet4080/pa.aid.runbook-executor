### Story
ARC-1361 — Implement `preflight_check` OpenCode tool

### Goal
Implement a `preflight_check` tool that validates the presence and cleanliness of required artifacts before starting a runbook execution.

### What Was Built
- New tool file: `~/.config/opencode/tools/preflight_check.ts`
  - Exports `findIssueFile(repoPath, epic, issueKey): string | null`
  - Exports `checkFileExists(filePath, name): PreflightCheckItem`
  - Exports `checkNoPlaceholders(filePath): PreflightCheckItem`
  - Default export: tool with args `{ repo_path, issue_key, epic, lane }`, returns `{ passed, checks[] }`
  - 4 checks: `issue_file`, `implementation_plan`, `plan_no_placeholders`, `runbook`
- New test file: `~/.config/opencode/tools/tests/preflight_check.test.ts`
  - 9 tests, all passing
  - Run with: `node --experimental-strip-types --test ~/.config/opencode/tools/tests/preflight_check.test.ts`
- Added `preflight_check: true` to `build` and `senior-coder` agents in `~/.config/opencode/opencode.json`

### Acceptance Criteria
- AC1 (all artifacts present, no placeholders → `passed: true`) ✅ — verified by test 7
- AC2 (plan missing → `passed: false`, `implementation_plan` check fails) ✅ — verified by test 8
- AC3 (plan has TBD/TODO → `passed: false`, `plan_no_placeholders` check fails) ✅ — verified by test 9
- AC4 (runbook missing → `passed: false`, `runbook` check fails) ✅ — covered by tool logic (same pattern as implementation_plan check)

### Test Evidence
```
✔ 1. findIssueFile — finds matching file in issues/epic/ dir
✔ 2. findIssueFile — epic dir does not exist → null
✔ 3. findIssueFile — dir exists but no matching file → null
✔ 4. checkNoPlaceholders — clean file → passed: true, no details
✔ 5. checkNoPlaceholders — file with TBD line → passed: false, details with line ref
✔ 6. checkNoPlaceholders — file does not exist → passed: false, details with message
✔ 7. execute — all artifacts present and plan clean → passed: true, 4 checks all passed
✔ 8. execute — plan file missing → passed: false, implementation_plan check fails
✔ 9. execute — plan contains TBD → passed: false, plan_no_placeholders check fails
tests 9, pass 9, fail 0
```

### Files Changed
- `~/.config/opencode/tools/preflight_check.ts`
- `~/.config/opencode/tools/tests/preflight_check.test.ts`
- `~/.config/opencode/opencode.json`

### Config Repo Commit
`8a5d76b` on branch `ARC-1361` in `pa.aid.config.md`