# ARC-1292 Run multiple lanes concurrently up to concurrency limit — Completion Summary

**Issue:** `issues/parallel-lane-execution/ARC-1292-concurrent-lanes.md`
**Implementation Plan:** `implementation_plans/parallel-lane-execution/ARC-1292-implementation-plan.md`
**Completed:** 2026-06-30
**Duration:** ~25 minutes
**Cost:** not tracked

---

## Acceptance Criteria Verification

| # | Criterion | Status | Evidence |
|---|-----------|--------|----------|
| 1 | Given concurrency N and M>N runbooks selected, when session starts, then exactly N lanes run and rest queue. | Passed | `laneScheduler.test.ts` — describe `"startSession — AC 1: concurrency cap at session start"`: (1) `"starts exactly maxConcurrent lanes immediately when M > N"` asserts `runLaneMock` called exactly 2 times for 4 lanes at `maxConcurrent=2`; (2) `"marks deferred lanes as queued"` asserts `setLaneQueued` called for lanes 3+4 and NOT called for lanes 1+2. `vitest run`: 161 passed, 0 failed. |
| 2 | Given running lane completes, when capacity frees, then next queued lane starts automatically. | Passed | `laneScheduler.test.ts` — describe `"startSession — AC 2: dequeue triggers automatically on lane completion"`: (1) `"starts next queued lane when first lane resolves (maxConcurrent = 1)"` manually resolves the first lane's Promise and asserts `runLaneMock` subsequently called for the second lane without external trigger; (2) `"starts all subsequent queued lanes in order as capacity frees"` verifies a 3-lane chain dequeues sequentially. `vitest run`: 161 passed, 0 failed. |

---

## Implementation Summary

### What was built

**`packages/server/src/runner/laneScheduler.ts`** (new, 77 lines)

A concurrency-bounded lane scheduler. Module-level `active: Set<string>` tracks runbook paths whose `runLane` Promise has not yet resolved; `queue: LaneEntry[]` holds lanes waiting to start.

- `startSession(lanes, maxConcurrent)`: launches the first `min(maxConcurrent, lanes.length)` entries immediately via fire-and-forget `void launchLane(...)`. The remaining lanes are marked `"queued"` via `setLaneQueued` and pushed to `queue`.
- `launchLane(entry)` (private): adds to `active`, awaits `runLane`, removes from `active` in `finally`, then shifts and recursively launches the next queued entry. All errors are caught and logged — unhandled rejections do not propagate.
- `_resetForTesting()`: clears `active` and `queue` for test isolation.

**`packages/server/src/runner/laneRunner.ts`** (modified)

- `LaneStatus` union extended: `'idle' | 'queued' | 'running' | 'paused' | 'done'`
- `setLaneQueued(runbookPath: string): void` exported — writes `"queued"` into `laneStatusMap` for lanes not yet started.

**`packages/server/src/routes/sessions.ts`** (modified)

`POST /api/sessions/start` handler: replaced the uncapped `for` loop (`void runLane(...).catch(...)`) with:
1. Build `lanesToSchedule: Array<{ runbookPath, steps }>` by mapping `selectedRunbooks` through `getRunbook` (preserving cache-miss warn+skip).
2. `startSession(lanesToSchedule, current.config.maxConcurrentLanes)`.

Response shape (`{ ok: true, sessionStartedAt }`) is unchanged. `maxConcurrentLanes` (previously declared in `SessionConfig` but never read) is now the live concurrency cap.

### Key decisions

- **Encapsulated scheduler module** (not inline in route): isolates concurrency state, enables independent unit testing by mocking `runLane`.
- **`finally`-block dequeue**: guarantees the next lane starts whether the previous lane succeeded, failed, or threw — matching the goal statement ("when a lane finishes or blocks").
- **Session route resolves steps**: `getRunbook` stays in the route layer; the scheduler only needs `{ runbookPath, steps }` pairs and `maxConcurrent`.

---

## Verification Steps

```bash
# From the ARC-1292 worktree
cd /repos/ARC-1292/runbook-executor/packages/server
../../node_modules/.bin/vitest run
../../node_modules/.bin/tsc --noEmit
```

Expected: `161 passed, 0 failed` (vitest); no output (tsc).

---

## Tests Added / Modified

| File | Change | Tests |
|---|---|---|
| `src/__tests__/laneScheduler.test.ts` | New — 10 tests for `startSession` | AC 1 cap enforced (2), AC 2 auto-dequeue (2), `maxConcurrent ≥ lane count` (2), empty lane list (2), error resilience (1), `maxConcurrent = 0` (1) |
| `src/__tests__/laneRunner.test.ts` | Modified — local `LaneStatus` type alias updated to include `"queued"` | Existing 16 tests continue to pass |

---

## Rollback Notes

None. The change is self-contained to the scheduler module and the sessions route handler. Rolling back requires reverting `laneScheduler.ts` (delete), `laneRunner.ts` (`"queued"` union and `setLaneQueued` removal), and `sessions.ts` (restore `for` loop). No database migrations or sidecar schema changes.

---

## Next Steps

- ARC-1293: Stream live agent output per lane in the UI — depends on ARC-1291 (already merged); can begin now.
- ARC-1294: Update runbook checkbox state in real time — batch with ARC-1293 or standalone; depends on ARC-1291.
