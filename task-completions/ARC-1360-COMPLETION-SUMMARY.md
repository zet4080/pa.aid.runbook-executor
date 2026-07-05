```markdown
# ARC-1360 Completion Summary — `archive_issue` OpenCode Tool

| Field | Value |
|-------|-------|
| Story | ARC-1360 |
| Title | Implement `archive_issue` OpenCode tool |
| Status | Completed |
| Completed | 2026-07-05 |
| Lane | agent-tools |

## Files Created

| File | Description |
|------|-------------|
| `~/.config/opencode/tools/archive_issue.ts` | Tool implementation — helpers + execute() + tool export |
| `~/.config/opencode/tools/tests/archive_issue.test.ts` | 10 unit tests covering all helpers and execute() paths |

## Config Repo

| Item | Value |
|------|-------|
| Branch | `ARC-1360` |
| Commit | `3f28947` |
| Repo | `pa.aid.config.md` |

## Test Results

All 10 tests pass under `node --experimental-strip-types --test`:

```
✔ 1. findArtifact — matching file found
✔ 2. findArtifact — directory does not exist → null
✔ 3. findArtifact — directory exists but no matching file → null
✔ 4. extractTitle — extracts first # heading from content
✔ 5. extractTitle — no # heading → "(no title)"
✔ 6. ensureReadmeTable — creates table header when file does not exist
✔ 7. ensureReadmeTable — no-op when table already present
✔ 8. execute — issue file missing → error returned, no moves
✔ 9. execute — plan file missing → skipped silently, issue and summary moved
✔ 10. execute — all 3 artifacts present → all moved, README updated
ℹ pass 10 / fail 0
```

## Allowlist Status

`archive_issue` was already present in `opencode-sync-config.sh` and `~/.config/opencode/opencode.json` for `build` and `senior-coder` agents (added during prior setup session).

## Acceptance Criteria

| AC | Description | Evidence |
|----|-------------|----------|
| AC1 | All 3 artifacts moved + README updated + commit+push | Test 10 verifies file moves + README row; execute() commits+pushes |
| AC2 | Missing plan skipped silently | Test 9 verifies issue+summary moved, plan absent from moved[] |
| AC3 | README table created if missing | Tests 6+7 verify ensureReadmeTable() creates header or no-ops |
| AC4 | Missing required file → error before any changes | Test 8 verifies error returned, moved[] is empty |
| AC5 | Push failure → preserve commit, return `pushed: false` | Separate try/catch in execute(); push result isolated from commit |
```