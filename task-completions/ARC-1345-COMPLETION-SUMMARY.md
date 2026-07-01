### ARC-1345 Fix checkpoint workflow — runbook file changes must not appear in checkpoint PR — Completion Summary

**Issue:** `issues/agent-workflow/ARC-1345-fix-runbook-commit-workflow.md`
**Implementation Plan:** `implementation_plans/agent-workflow/ARC-1345-implementation-plan.md`
**Completed:** 2026-07-01
**Duration:** ~30 minutes
**Cost:** —

## Acceptance Criteria Verification

| # | Criterion | Status | Evidence |
|---|-----------|--------|----------|
| 1 | Given agent has modified runbook files AND planning artifacts (both uncommitted), when checkpoint step reached, then runbook file changes committed directly to base branch and do NOT appear in checkpoint PR | Passed | New test `commits runbook files to base branch before creating checkpoint branch` verifies git add + git commit called before git checkout -b; `npm run test --workspace=packages/server` — 273 passed |
| 2 | Given agent has created planning artifacts (uncommitted), when checkpoint step reached, then planning artifacts appear in checkpoint branch and PR as expected | Passed | Pre-branch commit is scoped to `docs/plans/runbook-*.md` only; all other working-tree changes (planning artifacts) are carried to the checkpoint branch by git checkout -b as before; confirmed by code inspection of modified checkpointGit.ts |
| 3 | Given checkpoint branch has been created and pushed, when PR opened, then diff contains only planning artifacts — no runbook file changes | Passed | Direct consequence of AC1: runbook files committed to base branch before branch creation; verified by test `commits runbook files to base branch before creating checkpoint branch` |
| 4 | If no runbook changes are present, pre-branch commit step is a no-op | Passed | New test `skips runbook commit when no runbook files are modified` verifies git add and git commit for runbook-*.md are NOT called when git status --porcelain returns empty string; `npm run test --workspace=packages/server` — 273 passed |

## Implementation Summary

Single surgical change to `createCheckpointBranchAndPR` in `packages/server/src/git/checkpointGit.ts`:

- Before the `branchExistsLocally` check, inserted a pre-branch runbook commit block:
  1. Runs `git status --porcelain -- docs/plans/runbook-*.md` to detect uncommitted runbook changes
  2. If output is non-empty: runs `git add -- docs/plans/runbook-*.md` then `git commit -m "docs(runbook): auto-commit runbook checkoffs before checkpoint branch"` — directly on the current base branch
  3. If output is empty: no-op (skip add+commit)
- The subsequent branch creation and push logic is unchanged — only non-runbook working-tree changes (planning artifacts) follow to the checkpoint branch

Test file `packages/server/src/git/checkpointGit.test.ts`:
- Fixed 3 pre-existing test failures introduced by earlier commits (rev-parse guard, ls-remote skip-push optimization, auto-derive remote URL from git remote) that were never reflected in the test suite
- Replaced `execSyncMock.mockReturnValue('')` with `setupStandardSuccessMock()` helper using per-command mockImplementation
- Updated call-count assertion from 3 to 7 (reflecting full current call sequence)
- Added `FAKE_REMOTE_URL` constant and `setupStandardSuccessMock` helper
- Added 2 new tests: `commits runbook files to base branch before creating checkpoint branch` and `skips runbook commit when no runbook files are modified`

## Verification Steps

```bash
# Run server tests (from worktree)
cd /repos/ARC-1345 && npm run test --workspace=packages/server
# Expected: Test Files 19 passed (19), Tests 273 passed (273)

# Type-check
cd /repos/ARC-1345/packages/server && npx tsc --noEmit
# Expected: no output (clean)
```

## Tests Added/Modified

- `packages/server/src/git/checkpointGit.test.ts`
  - Fixed 3 pre-existing failures for success-path tests
  - Added `setupStandardSuccessMock()` helper
  - Added `FAKE_REMOTE_URL` constant
  - Added: `commits runbook files to base branch before creating checkpoint branch`
  - Added: `skips runbook file commit when no runbook files are modified`
  - Updated call-count assertion from 3 → 7

## Rollback Notes

None — additive change only. To revert: remove the 13-line pre-branch commit block from `createCheckpointBranchAndPR` in `checkpointGit.ts` and revert the test updates.

## Next Steps

None.
---
