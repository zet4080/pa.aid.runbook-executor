# ARC-1357: Implement `get_branch_diff` OpenCode tool

## Story
ARC-1357: Implement `get_branch_diff` OpenCode tool

## What was implemented
- Created `~/.config/opencode/tools/get_branch_diff.ts` — returns unified diff of current branch vs merge-base with main/master
- Created `~/.config/opencode/tools/tests/get_branch_diff.test.ts` — 13 tests, all passing
- Added `get_branch_diff: true` to `build` and `senior-coder` agent tool allowlists in `~/.config/opencode/opencode.json`
- Updated `opencode-sync-config.sh` in `pa.aid.wsl-setup.sh` with same allowlist entries

## Acceptance criteria coverage
1. **Given** a feature branch with commits since branching from `main`, **When** `get_branch_diff({ directory })` is called, **Then** the tool returns the unified diff from merge-base to HEAD — ✅ covered by tests 10, 13
2. **Given** the base branch is `master`, **When** called, **Then** auto-detects `master` — ✅ covered by tests 7, git detection logic
3. **Given** no changes since merge-base, **When** called, **Then** returns `{ diff: "", changed_files: [] }` — ✅ covered by test 11
4. **Given** no `main` or `master` branch, **When** called, **Then** returns error "Cannot determine base branch — neither main nor master found" — ✅ covered by tests 8, 12

## Test results
All 13 tests pass:
- Tests 1-5: pure helper unit tests (parseChangedFiles, buildErrorResult)
- Tests 6-8: git show-ref detection (main, master, develop/neither)
- Test 9: git merge-base returns hex hash
- Tests 10-11: git diff with changes and without changes
- Tests 12-13: end-to-end simulation (error path, success path)

## Artifacts
| Artifact | Path |
|----------|------|
| Tool implementation | `~/.config/opencode/tools/get_branch_diff.ts` |
| Tests | `~/.config/opencode/tools/tests/get_branch_diff.test.ts` |
| Config allowlist (generated) | `~/.config/opencode/opencode.json` |
| Config sync script | `components/opencode/opencode-sync-config.sh` in `pa.aid.wsl-setup.sh` |
| Config repo commit | `ec7d1c1` on branch `ARC-1357` in `pa.aid.config.md` |
| Setup script commit | `320b9e6` on `main` in `pa.aid.wsl-setup.sh` |

## Key design decisions
- `Bun.$` calls kept only in `execute()` — not exported as helpers (Bun not available in Node test runner)
- Exported pure helpers: `parseChangedFiles()` and `buildErrorResult()` — fully testable via Node
- Git command correctness tested end-to-end using `execSync` on real temp repos
- `main` preferred over `master` per issue spec
- Empty diff is valid (no error) — returns `{ diff: '', changed_files: [], base_branch }`