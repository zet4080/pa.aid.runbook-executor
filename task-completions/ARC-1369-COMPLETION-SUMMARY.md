# ARC-1369 Conductor: auto-commit implementation plans & completion summaries — Completion Summary

**Issue:** `issues/parallel-lane-execution/ARC-1369-conductor-auto-commit-planning-artifacts.md`
**Implementation Plan:** `implementation_plans/parallel-lane-execution/ARC-1369-implementation-plan.md`
**Completed:** 2026-07-07
**Duration:** ~1 session
**Cost:** not tracked

## Acceptance Criteria Verification

| # | Criterion | Status | Evidence |
|---|-----------|--------|----------|
| AC1 | `implementation_plans/` new file → committed + pushed with `docs({lane}): add {filename}` | Passed | `commitPlanningArtifacts` in `checkpointGit.ts`; test "one implementation_plans/ file dirty → commit with single-file message" passes |
| AC2 | `task-completions/` new file → committed + pushed | Passed | `commitPlanningArtifacts` in `checkpointGit.ts`; test "one task-completions/ file dirty → commit with single-file message" passes |
| AC3 | No dirty files → idempotent no-op | Passed | `if (statusOutput.trim() === '') return;` guard; test "no dirty files → execSync called once" passes |
| AC4 | Skills updated to remove manual git ceremony | Passed | `write-implementation-plan/SKILL.md` Phase 2 section updated; `write-completion-summary/SKILL.md` checklist updated |

## Implementation Summary

Added `commitPlanningArtifacts(repoRoot, lane)` to `checkpointGit.ts`. The function uses `execSync` with the same shape as existing git operations (10-second timeout, piped stdio, `cwd: repoRoot`). It calls `git status --porcelain -- implementation_plans/ task-completions/`, returns immediately if clean (AC3), otherwise parses filenames, stages, commits with `docs({lane}): add {filename}` (single file) or `docs({lane}): add planning artifacts` (multiple), and pushes. Push failure is caught and logged as a warning; any other error is caught in an outer try/catch — the function never throws.

Wired the call in `laneRunner.ts` immediately after `markStepCheckedInMarkdown()` in the no-pending-gate success path. Not called on gate-pause or step failure.

Skill files updated: `write-implementation-plan/SKILL.md` Phase 2 "Commit and Push" section replaced with auto-commit note; `write-completion-summary/SKILL.md` checklist line updated.

## Verification Steps

```bash
npm test --workspace packages/server
# 302/304 pass (2 pre-existing failures on main, unrelated to ARC-1369)
npm run build --workspace packages/server
# compiles with no errors
```

## Tests Added/Modified

- `packages/server/src/git/checkpointGit.test.ts` — 6 new tests in `describe('commitPlanningArtifacts', ...)` covering: no-op, single implementation_plans/ file, single task-completions/ file, multiple files, push failure (warn only), git status throws (outer catch)
- `packages/server/src/__tests__/laneRunner.test.ts` — 4 new tests in `describe('runLane — auto-commit planning artifacts (ARC-1369)', ...)` covering: called on step success, not called on failure, not called on gate-pause, called once per step; mock infrastructure extended

## Rollback Notes

None — additive change. Removing would require deleting `commitPlanningArtifacts` from `checkpointGit.ts` and removing the call + import from `laneRunner.ts`.

## Next Steps

- ARC-1370 — auto-archive story artifacts on completion (depends on this story)
