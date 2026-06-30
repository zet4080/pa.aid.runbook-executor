# ARC-1292 Run multiple lanes concurrently up to concurrency limit — Implementation Plan

**Issue:** `issues/parallel-lane-execution/ARC-1292-concurrent-lanes.md`
**Completion Summary:** `task-completions/ARC-1292-COMPLETION-SUMMARY.md` (written after execution)
**Approach:** Encapsulated LaneScheduler module with completion callback (Option 1)
**Owner:** build-agent
**Date:** 2026-06-30

---

## Scope & Alignment

This plan implements concurrent lane execution bounded by `SessionConfig.maxConcurrentLanes`. It covers:

- A new `LaneScheduler` module that manages an active-lanes set and a waiting queue, starts up to the configured limit immediately, and automatically dequeues the next lane when capacity frees.
- Replacing the uncapped `for` loop in `POST /api/sessions/start` with a single `scheduler.startSession()` call.
- Exposing a `"queued"` lane status so consumers can observe waiting lanes.

**AC mapping:**

| AC | Satisfied by |
|---|---|
| AC 1 — exactly N lanes run, rest queue when M > N | Step 1 (LaneScheduler) + Step 3 (sessions route) |
| AC 2 — next queued lane starts automatically when capacity frees | Step 1 (dequeue-on-completion logic in LaneScheduler) |

---

## Assumptions & Dependencies

- `runLane(runbookPath, steps): Promise<void>` is the public lane execution contract from `runner/laneRunner.ts`; it resolves when the lane reaches `done` or `paused`. This plan does not change `runLane`.
- `SessionConfig.maxConcurrentLanes` is already declared and defaulted to 3 in `state/types.ts`. It is read from `getState().config.maxConcurrentLanes` at session start.
- `getRunbook(runbookPath)` in `runner/runbookCache.ts` remains the step-resolution point; the sessions route continues to call it before passing data to the scheduler.
- `laneStatusMap` in `runner/laneRunner.ts` owns the canonical status for running/paused/done lanes. The new `"queued"` status is set by the scheduler on lanes not yet started; the scheduler clears it by calling `runLane` (which sets `"running"` internally).
- Node.js is single-threaded: state mutations between `await` points are safe. After each `await runLane(...)` completion the scheduler re-reads concurrency state from its own in-memory `active` set (not from `getState()`) — the active set is local to the scheduler and not shared with other request handlers.
- ARC-1291 (sequential step execution) is merged. ARC-1288 (opencode CLI subprocess) is merged. No other in-lane dependencies for this story.

---

## Implementation Steps

### Step 1: Create `runner/laneScheduler.ts` — LaneScheduler with active set and queue

**Files:** `packages/server/src/runner/laneScheduler.ts` (new file)

**Action:** Define and export a `startSession` function. The module maintains:
- `active: Set<string>` — runbook paths whose `runLane` promise has not yet resolved
- `queue: Array<{ runbookPath: string; steps: RunbookStep[] }>` — lanes waiting to start

`startSession(lanes, maxConcurrent)` receives the full list of `{ runbookPath, steps }` pairs and the concurrency limit. It immediately starts the first `min(maxConcurrent, lanes.length)` entries; the remainder are pushed to `queue` with their `laneStatusMap` entry set to `"queued"` via `setLaneQueued`.

Each lane is launched via a private `launchLane(entry)` helper that:
1. Adds the path to `active`
2. Calls `runLane(path, steps)` and awaits its resolution
3. Removes the path from `active`
4. If `queue` is non-empty, shifts the next entry and calls `launchLane` recursively

`launchLane` is fire-and-forget from `startSession` (called with `void launchLane(...)`) — errors are caught and logged; unhandled rejections must not propagate.

Export a `_resetForTesting()` function that clears `active` and `queue` (same convention as `_resetLaneStateForTesting` in `laneRunner.ts`).

**Interface contract:**
```ts
export function startSession(
  lanes: Array<{ runbookPath: string; steps: RunbookStep[] }>,
  maxConcurrent: number,
): void
export function _resetForTesting(): void
```

**Verification:** `tsc --noEmit` compiles cleanly. Importing `startSession` and `_resetForTesting` in a test file resolves without error.

---

### Step 2: Add `"queued"` status to `LaneStatus` and expose setter from `laneRunner.ts`

**Files:** `packages/server/src/runner/laneRunner.ts`

**Action:** Extend the exported `LaneStatus` type union to include `"queued"`:

```ts
export type LaneStatus = 'idle' | 'queued' | 'running' | 'paused' | 'done';
```

Add an exported `setLaneQueued(runbookPath: string): void` function that writes `"queued"` into `laneStatusMap`. The scheduler calls this for each lane it defers before `runLane` is called; `runLane` continues to overwrite with `"running"` when execution actually begins.

**Verification:** `tsc --noEmit` passes. After calling `setLaneQueued(path)`, `getLaneState(path)` returns `"queued"`. After `runLane` is subsequently called for the same path, `getLaneState(path)` returns `"running"` (verified via existing `laneRunner.test.ts` status transition tests which must still pass).

---

### Step 3: Replace uncapped `for` loop in `routes/sessions.ts` with `startSession`

**Files:** `packages/server/src/routes/sessions.ts`

**Action:** In the `POST /api/sessions/start` handler, replace the `for (const runbookPath of current.selectedRunbooks)` block with:
1. Build a `resolvedLanes` array by mapping `current.selectedRunbooks` through `getRunbook`, preserving the existing cache-miss `console.warn` and skip pattern.
2. Call `startSession(resolvedLanes, current.config.maxConcurrentLanes)`.

The response shape (`{ ok: true, sessionStartedAt }`) is unchanged. The `writeSidecar` / `setState` calls that precede lane launch are unchanged.

**Verification:** All existing tests in `__tests__/sessions.test.ts` pass without modification. `POST /api/sessions/start` with a single runbook selected still starts that lane.

---

### Step 4: Write `__tests__/laneScheduler.test.ts` — unit tests for scheduler

**Files:** `packages/server/src/__tests__/laneScheduler.test.ts` (new file)

**Action:** Write tests that mock `runLane` from `runner/laneRunner.js` (same `vi.mock` pattern as `laneRunner.test.ts` uses for `executeStep`) and verify scheduler concurrency behaviour. Tests call `_resetForTesting()` and `_resetLaneStateForTesting()` in `beforeEach`. Cover these scenarios:

**Scenario 1 — AC 1: concurrency cap is respected at session start.**
Given `maxConcurrent = 2` and 4 lanes, when `startSession` is called, then `runLane` is invoked for exactly 2 paths immediately and the remaining 2 have `getLaneState` returning `"queued"`.

**Scenario 2 — AC 2: dequeue triggers automatically on lane completion.**
Given `maxConcurrent = 1` and 2 lanes, when the first lane's mock promise resolves, then `runLane` is invoked for the second lane without any external trigger.

**Scenario 3 — edge: maxConcurrent ≥ lane count, no queuing.**
Given `maxConcurrent = 5` and 3 lanes, all 3 `runLane` calls are initiated immediately; no lane has status `"queued"`.

**Scenario 4 — edge: empty lane list.**
Given an empty `lanes` array, `startSession` returns without error and `runLane` is never called.

**Scenario 5 — error resilience.**
Given a lane whose `runLane` mock rejects, the rejection is caught and the next queued lane still starts (dequeue proceeds normally).

**Verification:** `vitest run` passes all new tests. No existing tests are broken.

---

### Step 5: Verify full test suite and type-check pass

**Files:** all test files under `packages/server/src/__tests__/`

**Action:** Run `vitest run` from `packages/server/`. Run `tsc --noEmit` from `packages/server/`. Both must report zero errors.

**Verification:** Clean output from both commands with no failures or type errors.

---

## Testing & Validation

All verification is via `vitest run` in `packages/server/` and `tsc --noEmit`.

**Scenario coverage:**

- **Concurrency cap at start (AC 1):** With `maxConcurrentLanes = 2` and 4 lanes, only 2 `runLane` calls are initiated immediately. The remaining 2 lanes have status `"queued"` in `laneStatusMap`.
- **Auto-dequeue on completion (AC 2):** After the first completing lane's promise resolves, the scheduler picks the next queued entry and calls `runLane` for it without any external trigger.
- **No cap exceeded at any point:** At no point during a multi-lane session does the count of concurrently executing lanes exceed `maxConcurrentLanes`.
- **Single lane — no queuing:** With `maxConcurrentLanes = 3` and 1 lane, the lane starts immediately; no queuing occurs. Existing `sessions.test.ts` scenarios cover this regression path.
- **All lanes fit within limit:** With `maxConcurrentLanes ≥ lane count`, all lanes start immediately with no queue entries.
- **Error in one lane does not stop others:** A rejected `runLane` promise is caught; queued lanes continue to dequeue normally.
- **Session route response unchanged:** `POST /api/sessions/start` still returns `{ ok: true, sessionStartedAt }` regardless of how many lanes are queued.

---

## Risks & Open Questions

- **`laneStatusMap` is module-level in `laneRunner.ts`**: The scheduler writes `"queued"` via `setLaneQueued` before calling `runLane`. If `runLane` is ever called directly (bypassing the scheduler), the queued status will never be set. This is acceptable: `runLane` is an internal API and `startSession` is the only call-site after this story lands.
- **Session close while lanes are queued (out of scope per AC):** `POST /api/sessions/close` does not cancel queued or running lanes per the issue's out-of-scope list. Queued lanes will start even after a close event. Noted here for future work; no action required in this story.
- **`maxConcurrentLanes = 0` edge case:** The issue does not specify behaviour for a zero limit. The implementation treats `maxConcurrent <= 0` as "start no lanes immediately — all queue forever". This edge is validated in tests but is not an explicit AC.
