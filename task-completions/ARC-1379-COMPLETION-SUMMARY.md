# ARC-1379: Fix worktree-provisioning silent failure — route to escalation checkpoint — Completion Summary

**Issue:** `issues/story-lifecycle/ARC-1379-fix-worktree-provisioning-silent-failure.md`
**Implementation Plan:** `implementation_plans/story-lifecycle/ARC-1379-implementation-plan.md`
**Completed:** 2026-07-12
**Branch:** `ARC-1379` (pa.aid.conductor.ts, branched off `main`, pushed to origin, PR intentionally not opened)
**Commit:** `6258e08` — "fix(story-lifecycle): route worktree-provisioning failure to escalation checkpoint"

## Acceptance Criteria Verification

| # | Criterion | Status | Evidence |
|---|-----------|--------|----------|
| 1 | Given worktree provisioning throws an error, when the error occurs, then the story transitions to `awaiting_escalation_checkpoint` instead of failing silently | Passed | `packages/server/src/runner/laneRunner.ts` — the `provisionWorktree(storyKey)` call inside `runLane`'s `isHighStory(step)` branch is wrapped in try/catch. On catch: `setStoryLifecycleState(getState(), storyKey, 'awaiting_escalation_checkpoint')` is called, a `CheckpointQueueEntry` (`checkpointLevel: 'high'`) is pushed onto `checkpointQueue`, state is persisted via `writeSidecar`, and the lane is set to `'paused'`. `runLane` then returns normally (no unhandled rejection). Verified by two new/rewritten tests in `packages/server/src/__tests__/laneRunner.test.ts`: `'pauses the lane and pushes a checkpoint entry when provisionWorktree throws (ARC-1379)'` (asserts `runLane` resolves, `executeStep` not called, lane state `'paused'`, one queued `checkpointQueue` entry with `checkpointLevel: 'high'`) and `'records awaiting_escalation_checkpoint lifecycle state when provisionWorktree throws (ARC-1379)'` (asserts `getStoryLifecycleState(getState(), 'ARC-1368')` equals `'awaiting_escalation_checkpoint'`). Both pass — `npx vitest run` full suite: 394/394 tests passed, 22 files. |
| 2 | Given the story is in `awaiting_escalation_checkpoint` due to a worktree failure, when the supervisor resumes after fixing the issue, then the exact same worktree-provisioning operation is re-attempted (uniform resume behavior, no special-case routing needed) | Passed | No changes made to `packages/server/src/routes/checkpoints.ts` — confirmed via `git diff HEAD --stat` showing only `laneRunner.ts` and `laneRunner.test.ts` changed. The pushed `CheckpointQueueEntry` deliberately has no `prUrl` and no `planningArtifact` field, so it flows through the pre-existing generic "dequeue and re-run `runLane` from disk" branch in `checkpoints.ts`'s resume route with zero new code. Because the failed step's `checked` field is never set to `true` (the catch returns before `markStepCheckedInMarkdown`), re-parsing and re-invoking `runLane` naturally re-enters the same `isHighStory(step)` branch and re-calls `provisionWorktree(storyKey)`. Verified by new test `'resume after worktree-provisioning failure re-attempts provisionWorktree with the same storyKey (ARC-1379)'`: first `runLane` call throws (mocked via `mockImplementationOnce`) and pauses the lane; a direct second `runLane` call with the same unchanged step object asserts `provisionWorktreeMock` was called a second time with `'ARC-1368'` and `executeStepMock` was then called once (provisioning succeeded on retry, agent dispatch proceeded) — proving no special-case routing exists. |

## Implementation Summary

- Wrapped the `provisionWorktree(storyKey)` call in `runLane` (`packages/server/src/runner/laneRunner.ts`) in a try/catch, following the exact `getState()` → mutate → `setState()` → `writeSidecar()` pattern already used at every other checkpoint-push site in the file (e.g. the retries-exhausted block, the `triggerPlanningArtifactPR` catch block).
- On catch: logs via `console.error` with an `[ARC-1379]` prefix (matching the `[ARC-1371]`/`[ARC-1302]` convention), calls the already-merged (ARC-1374) `setStoryLifecycleState` helper to record `'awaiting_escalation_checkpoint'` for the story, pushes a generic `CheckpointQueueEntry` (`checkpointLevel: 'high'`, no `prUrl`/`planningArtifact`) onto `checkpointQueue`, persists state, sets the lane status to `'paused'`, and returns — so `runLane` resolves normally instead of rejecting.
- Extended the existing `import { writeSidecar } from '../state/sidecar.js'` to also import `setStoryLifecycleState`.
- No changes to `worktreeGit.ts`'s own provisioning logic (unchanged per issue's Out of Scope) and no changes to `checkpoints.ts`'s resume route (unchanged per the plan's Step 3 analysis — the generic resume path already handles this entry shape).
- Rewrote the pre-existing test that asserted `runLane` rejects on a `provisionWorktree` throw (that assertion was the exact silent-failure behavior being fixed) to assert the new paused-and-queued behavior. Added two new tests covering the lifecycle-state write and the resume re-attempt path.

## Verification Steps

```bash
# Typecheck
cd /repos/ARC-1379/packages/server && npx tsc --noEmit
# → clean, 0 errors

# Full test suite
cd /repos/ARC-1379/packages/server && npx vitest run
# → Test Files 22 passed (22); Tests 394 passed (394)

# Targeted test file
cd /repos/ARC-1379/packages/server && npx vitest run src/__tests__/laneRunner.test.ts
# → 92 tests passed

# Diff scope confirmation
cd /repos/ARC-1379 && git diff main...HEAD --stat
# → 2 files changed: packages/server/src/runner/laneRunner.ts,
#   packages/server/src/__tests__/laneRunner.test.ts
```

## Tests Added/Modified

- `packages/server/src/__tests__/laneRunner.test.ts`
  - **Rewrote** `'pauses the lane and pushes a checkpoint entry when provisionWorktree throws (ARC-1379)'` — now asserts `runLane` resolves (not rejects), `executeStep` was not called, lane state is `'paused'`, and one `CheckpointQueueEntry` (`checkpointLevel: 'high'`, string `hitAt`) is queued.
  - **Added** `'records awaiting_escalation_checkpoint lifecycle state when provisionWorktree throws (ARC-1379)'` — asserts `getStoryLifecycleState` returns `'awaiting_escalation_checkpoint'` for the story key after the throw.
  - **Added** `'resume after worktree-provisioning failure re-attempts provisionWorktree with the same storyKey (ARC-1379)'` — asserts a second direct `runLane` invocation with the same unchanged step re-attempts `provisionWorktree` with the same story key and proceeds to `executeStep` on success.
  - Also updated the `vi.mock('../state/sidecar.js', ...)` mock factory to add pure re-implementations of `getStoryLifecycleState`/`setStoryLifecycleState` matching production logic (the module mock previously only stubbed `writeSidecar`/`readSidecar`/`createFreshState`).

## Local Code Review

Ran the `local-code-review` skill loop — clean on iteration 1/3 (no BLOCKER, no ISSUE, no SUGGESTION, no NIT findings). Full test suite (394/394), typecheck (`tsc --noEmit`, 0 errors) both pass. No lint tool is configured for `packages/server` in this repo (no `lint` script, no ESLint config) — skipped per the skill's "skip checks with no corresponding tool" rule.

## Rollback Notes

None. The change is additive error-handling around an existing call site; reverting the commit restores the prior (silent-failure) behavior with no data migration required. `storyLifecycle` sidecar entries written by this code path are informational state, not consumed by any other code yet (per ARC-1374's stated scope — types + persistence plumbing only, no readers yet).

## Next Steps

- ARC-1380 (sibling story, executed in parallel in a separate worktree) applies the same try/catch + `setStoryLifecycleState` + generic `CheckpointQueueEntry` pattern to `archiveStoryArtifacts` failures — no coordination was needed since each branch is isolated off `main`; any merge conflicts in `laneRunner.ts` will be resolved at PR/merge time.
- No new `checkpoints.ts` read-side consumer of `storyLifecycle` state was added — a future story may want to surface the `awaiting_escalation_checkpoint` reason distinctly in the UI (would require a `failureReason` field on `CheckpointQueueEntry`), but that is out of this story's scope per its Acceptance Criteria (state transition + uniform resume only).
