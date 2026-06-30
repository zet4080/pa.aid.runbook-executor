# ARC-1292 Run multiple lanes concurrently up to concurrency limit — Completion Summary

**Issue:** `issues/parallel-lane-execution/ARC-1292-concurrent-lanes.md`
**Implementation Plan:** `implementation_plans/parallel-lane-execution/ARC-1292-implementation-plan.md`
**Completed:** 2026-06-30
**Duration:** ~25 minutes (core) + ~60 minutes (post-completion bug fixes)
**Cost:** not tracked

---

## Acceptance Criteria Verification

| # | Criterion | Status | Evidence |
|---|-----------|--------|----------|
| 1 | Given concurrency N and M>N runbooks selected, when session starts, then exactly N lanes run and rest queue. | Passed | `laneScheduler.test.ts` — describe `"startSession — AC 1: concurrency cap at session start"`: (1) `"starts exactly maxConcurrent lanes immediately when M > N"` asserts `runLaneMock` called exactly 2 times for 4 lanes at `maxConcurrent=2`; (2) `"marks deferred lanes as queued"` asserts `setLaneQueued` called for lanes 3+4 and NOT called for lanes 1+2. `vitest run`: 187 passed, 0 failed. |
| 2 | Given running lane completes, when capacity frees, then next queued lane starts automatically. | Passed | `laneScheduler.test.ts` — describe `"startSession — AC 2: dequeue triggers automatically on lane completion"`: (1) `"starts next queued lane when first lane resolves (maxConcurrent = 1)"` manually resolves the first lane's Promise and asserts `runLaneMock` subsequently called for the second lane without external trigger; (2) `"starts all subsequent queued lanes in order as capacity frees"` verifies a 3-lane chain dequeues sequentially. `vitest run`: 187 passed, 0 failed. |

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

## Post-Completion Bug Fixes

Three bugs discovered during integration testing were fixed on the ARC-1292 branch before merge:

### Fix 1 — `astWalker.ts` checkpoint classification (commits `f5254b8`, `cc7d2c4`)

**Bug:** `deriveStepTypeAndLevel` classified every list item under a `🔴 HIGH` H3 heading as `type: 'checkpoint'` — including story title checkboxes, action items, and all sub-items — because it inspected the H3 heading text rather than the individual item label.

**Fix:** Renamed to `deriveStepTypeFromLabel(itemLabel)`. A step is now `type: 'checkpoint'` if and only if its own label contains `CHECKPOINT` or `GATE` (case-insensitive). H3 heading text no longer affects classification. Checkpoint level derived from item-level keywords: `BATCH` → `medium`, `WAVE` → `low`, fallback → `high`.

**Impact:** Prevented story-title items (`**ARC-1293** — ...`) from being pushed to `checkpointQueue` immediately on lane start, which was causing both lanes to show "Blocked" in the UI.

### Fix 2 — `buildPromptFromStep` non-executable sub-step filtering (commits `f35791d`, `15440d7`)

**Bug:** `buildPromptFromStep` sent all unchecked sub-steps to opencode, including bookkeeping markers (`🔒 Claimed:`, `_(fill in...)_`) and pause gates (`INDIVIDUAL PLAN CHECKPOINT`).

**Fix:** Added `isNonExecutableSubStep(label)` filter and ARC-key context line injection:
- Sub-steps with `CHECKPOINT`, `GATE`, `Claimed:`, or `_(fill in` in their label are excluded from the numbered task list.
- When the step label contains an ARC key, a context line is prepended: `Context: See issues/{lane}/{ARC-KEY}-*.md for full requirements and acceptance criteria.`
- `buildPromptFromStep` signature updated to accept `runbookPath` for lane derivation.

### Fix 3 — Prompt segmentation at CHECKPOINT sub-step boundaries (commits `f3f0c73`, `dede690`)

**Bug:** `buildPromptFromStep` sent all unchecked executable sub-steps at once. For a HIGH story with sub-steps [Read issue, Write plan, CHECKPOINT, Execute plan, ...], opencode received all tasks including post-checkpoint tasks in the first call.

**Fix:** Sub-steps are segmented at the first unchecked CHECKPOINT/GATE sub-step:
- **Segment 1 (first call):** unchecked executable sub-steps before the gate only.
- **Segment 2 (resume):** if all pre-gate sub-steps are checked, unchecked executable sub-steps after the gate.

Additionally, `runLane` now detects an unchecked CHECKPOINT/GATE sub-step after each successful `executeStep` call and pushes a `CheckpointQueueEntry` with composite ID `${step.id}__checkpoint`, pausing the lane. A module-level `pushedSubStepCheckpoints: Set<string>` guards against duplicate queue entries on re-entry.

---

## Verification Steps

```bash
# From the merged runbook-executor branch
cd /repos/pa.aid.wsl-setup.sh/runbook-executor/packages/server
npm test
npx tsc --noEmit
```

Expected: `187 passed, 0 failed` (vitest); no output (tsc).

---

## Tests Added / Modified

| File | Change | Tests |
|---|---|---|
| `src/__tests__/laneScheduler.test.ts` | New — 10 tests for `startSession` | AC 1 cap enforced (2), AC 2 auto-dequeue (2), `maxConcurrent ≥ lane count` (2), empty lane list (2), error resilience (1), `maxConcurrent = 0` (1) |
| `src/runbook/parser.test.ts` | Modified + extended | Old wrong H3-heading assertions corrected; 9 new per-item classification tests added |
| `src/__tests__/laneRunner.test.ts` | Modified + extended | `LaneStatus` type alias updated; 7 new filtering tests; 7 new segmentation + sub-step checkpoint detection tests |

Total test count: **187 passing** (was 161 before bug fixes).

---

## Commits on ARC-1292 branch

| Hash | Description |
|---|---|
| `1f4c92a` | LaneScheduler + queued status + sessions for-loop replacement |
| `9e39728` | laneScheduler unit tests (10 scenarios) |
| `f5254b8` | Failing test: checkpoint classification must be per-item not per-heading |
| `cc7d2c4` | Fix: checkpoint classification — per-item label not H3 heading |
| `f35791d` | Failing tests: buildPromptFromStep must filter non-executable sub-steps |
| `15440d7` | Fix: filter non-executable sub-steps and inject ARC context in buildPromptFromStep |
| `f3f0c73` | Failing tests: checkpoint sub-step segmentation and pause detection |
| `dede690` | Fix: checkpoint sub-step segmentation, pause detection, and idempotency guard |

---

## Rollback Notes

All changes are self-contained to the scheduler module, laneRunner, sessions route, and astWalker parser. Rolling back requires reverting the 8 commits above. No database migrations or sidecar schema changes.

---

## Next Steps

- ARC-1293: Stream live agent output per lane in the UI — depends on ARC-1291 (already merged); can begin now.
- ARC-1294: Update runbook checkbox state in real time — batch with ARC-1293 or standalone; depends on ARC-1291.

