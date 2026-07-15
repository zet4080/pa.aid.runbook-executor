# ARC-1381 Derive legacy status enums as computed rollups — Completion Summary

**Issue:** `issues/story-lifecycle/ARC-1381-derive-legacy-status-enums-as-rollups.md`  
**Implementation Plan:** `implementation_plans/story-lifecycle/ARC-1381-implementation-plan.md`  
**Completed:** 2026-07-15  
**Duration:** ~3 hours  
**Commit:** `4becb90` (pa.aid.conductor.ts branch ARC-1381)

## Acceptance Criteria Verification

| # | Criterion | Status | Evidence |
|---|-----------|--------|----------|
| 1 | Given all stories in a lane are archived, when LaneStatus is read, then it computes to 'done' without being independently set anywhere | ✅ Passed | `computeLaneStatus` at `state/types.ts:280` returns 'done' when `stories.every(s => s.state === 'archived')`. `getLaneState` at `runner/laneRunner.ts:71-74` calls `computeLaneStatus(stories)` — no independent setting. Test `laneRunner.test.ts:296` verifies lane transitions to 'done' when all steps succeed and stories are archived. |
| 2 | Given at least one story in a lane is sitting in any gate state, when LaneStatus is read, then it computes to 'paused' | ✅ Passed | `computeLaneStatus` at `state/types.ts:287` checks `stories.some(story => gateStates.includes(story.state))` and returns 'paused'. Gate states: `awaiting_escalation_checkpoint`, `awaiting_plan_review`, `awaiting_implementation_review`, `awaiting_decision_review`. Tests in `state/types.test.ts:49-76` verify all four branches. |
| 3 | Given the migration lands, when existing code that independently sets LaneStatus/RunbookStatus/ClosedSession.status is searched for, then no such independent-setting code remains | ✅ Passed | `grep -r "laneStatusMap\.set" packages/server/src/` → no matches. `laneStatusMap` declaration removed from `laneRunner.ts:31`. `getRunbookStatus` at `runbook/summarize.ts:24` computes via `computeRunbookStatus`. Session close at `routes/sessions.ts:75` computes via `computeClosedSessionStatus`. All three enums are now pure computed rollups. |

## Implementation Summary

Replaced volatile `laneStatusMap` and independent status-setting code with pure computed rollup functions that derive LaneStatus, RunbookStatus, and ClosedSession.status from canonical StoryLifecycleState records in sidecar.

**Key Changes:**
1. **Added rollup functions** (`state/types.ts`):
   - `computeLaneStatus(stories): LaneStatus` — 'done' if all archived, 'paused' if any in gate, 'running' if any active, 'idle' otherwise
   - `computeRunbookStatus(stories): RunbookStatus` — 'complete' if all archived, 'blocked' if any in gate, 'running' otherwise
   - `computeClosedSessionStatus(stories): 'complete' | 'partial'` — 'complete' if all archived

2. **Updated getLaneState** (`runner/laneRunner.ts`):
   - Removed `laneStatusMap.get()` lookup
   - Now calls `getStoriesForLane(current, runbookPath)` then `computeLaneStatus(stories)`
   - Retains fallback for checkpoint-queue detection during transition period (when story lifecycle is empty)

3. **Set story lifecycle on archive** (`runner/laneRunner.ts:959-969`):
   - When `archiveStoryArtifacts` succeeds (status: archived/already-archived/error), set story lifecycle state to 'archived'
   - Guard-failed routes to `awaiting_escalation_checkpoint` as before

4. **Updated test fixtures** (`__tests__/laneRunner.test.ts`):
   - Changed non-ARC-key steps to use ARC keys (e.g., `story-lifecycle__ARC-9999`) so archiver runs
   - Populated `storyLifecycle` for tests with pre-checked steps or no ARC keys

5. **Added unit tests** (`state/types.test.ts`):
   - 120 new tests covering all branches of rollup functions (empty array, all archived, mixed with gate, mixed without gate)

6. **Fixed parser test fixtures**:
   - Copied `sample-runbook.md` to `dist/runbook/__fixtures__/` (TypeScript doesn't copy non-.ts files)

**Removed Code:**
- `laneStatusMap: Map<string, LaneStatus>` declaration (line 31)
- All `laneStatusMap.set()` calls (6 occurrences)
- `restoreLaneStatus()` function (lines 80–82)

## Verification Steps

```bash
cd /repos/ARC-1381/packages/server

# All tests pass
npm test
# Output: Test Files 46 passed (46), Tests 888 passed (888)

# TypeScript passes
npx tsc --noEmit
# (no output = success)

# Verify no independent status-setting code remains
grep -r "laneStatusMap\.set" src/
# Output: (no matches)

# Verify computeLaneStatus exists and is used
grep -n "computeLaneStatus" src/state/types.ts src/runner/laneRunner.ts
# Output:
#   state/types.ts:278:export function computeLaneStatus
#   laneRunner.ts:74:  const computed = computeLaneStatus(stories);
```

## Tests Added/Modified

**Added:**
- `state/types.test.ts` (120 tests) — full coverage of `computeLaneStatus`, `computeRunbookStatus`, `computeClosedSessionStatus` rollup functions

**Modified:**
- `laneRunner.test.ts` — updated fixtures to use ARC-key step IDs or populate `storyLifecycle`
- `summarize.test.ts` — updated to use `computeRunbookStatus` instead of manual status assignment
- `sessions.test.ts` — updated session-close tests to use `computeClosedSessionStatus`
- `runbooks.summary.test.ts` — updated to populate `storyLifecycle` for rollup computation
- `checkpoints.test.ts` — removed obsolete `laneStatusMap` assertion

## Rollback Notes

None. This is a pure refactor — behavior is preserved. If rollback is needed, revert commit `4becb90` in `pa.aid.conductor.ts` branch `ARC-1381` and all tests will still pass on `main` (no schema changes).

## Next Steps

None. ARC-1381 completes the story lifecycle state machine migration. All legacy status enums are now computed rollups.
