# ARC-1306 Retry failed agent steps up to configurable limit — Implementation Plan

**Issue:** `issues/failure-handling/ARC-1306-retry-failed-steps.md`
**Completion Summary:** `task-completions/ARC-1306-COMPLETION-SUMMARY.md` (TBD)
**Approach:** In-loop retry within `runLane` — intercept the failure path before pausing, re-invoke `executeStep` up to `retryLimit` times, publish synthetic retry-announce events between attempts, then pause when retries exhausted
**Owner:** build agent
**Date:** 2026-07-01

## Scope & Alignment

This plan covers the automatic retry mechanism for failed agent steps inside the lane runner. It maps directly to the two acceptance criteria:

| AC | Step(s) that satisfy it |
|----|------------------------|
| AC 1: Retry remaining → spawn new subprocess for same step | Steps 1, 2, 3 |
| AC 2: Retry limit reached → no further auto-retries, escalation triggered | Steps 2, 4 |

Retry logging in lane output (goal statement) is addressed by Step 3 (synthetic event published to event bus).

Out of scope: escalation action after retries exhausted (ARC-1307 handles agent-level escalation; ARC-1308 handles notification). This plan ends at "lane paused, escalation hook ready".

## Assumptions & Dependencies

- ARC-1288 is merged. `runStep` in `subprocessManager.ts` is the subprocess entry point; `executeStep` in `stepRunner.ts` is the unit-of-work wrapper called by `runLane`.
- `SessionConfig.retryLimit` already exists in `state/types.ts` (default `3`). No new config field required.
- `retryLimit === 0` means no retries — one attempt only. A step that fails on the first attempt with `retryLimit === 0` pauses the lane immediately (consistent with current behaviour).
- The retry counter is per-step-invocation, not per-session. It resets for each step the lane runner processes.
- The `StepRecord` in `state/types.ts` needs a `retryCount` field so retry attempts are visible in session history (ARC-1312 consumers).
- ARC-1307 will read `retryCount` from the step record to drive agent-level escalation on each retry. This plan must expose `retryCount` in the record so ARC-1307 has a signal to act on.
- The event bus (`runner/eventBus.ts`) is the correct channel for emitting retry-announce messages to SSE subscribers; synthetic events with `type: 'retry'` are already handled as `UnknownEvent` by all consumers.

## Implementation Steps

### Step 1: Add `retryCount` field to `StepRecord`

**File:** `packages/server/src/state/types.ts`

**Action:** Add `retryCount: number` to the `StepRecord` interface, defaulting to `0` for new records. Update the initial record construction in `stepRunner.ts` (`executeStep`) to set `retryCount: 0`.

**Verification:** TypeScript compiler accepts the change with no errors; existing sidecar read/write round-trips `retryCount` correctly (tests in `sidecar.test.ts` or `state.test.ts` cover the round-trip).

### Step 2: Introduce `retryStep` helper in `laneRunner.ts`

**File:** `packages/server/src/runner/laneRunner.ts`

**Action:** Add a private async function `retryStep(stepId, prompt, retryLimit)` that:
- Accepts the step id, the already-constructed prompt string, and the retry limit from `SessionConfig`.
- Runs the attempt loop: calls `executeStep(stepId, prompt)` for attempt 0 (the initial call made by the caller), then for attempts 1..retryLimit on failure.
- Between each retry (before invoking `executeStep` again), publishes a synthetic `{ type: 'retry', attempt: N, stepId }` event via `publish(stepId, syntheticEvent)` so SSE subscribers see "Retry N of retryLimit" in the lane output.
- Returns `{ result: RunResult, retryCount: number }` — the final `RunResult` and the number of retries attempted (0 if the first attempt succeeded).

The caller (`runLane` agent-step block) replaces the current direct `executeStep` call with `retryStep`. On failure after exhausting retries, `runLane` pauses the lane as before.

**Verification:** Unit tests (Step 5) cover: success on first attempt (retryCount 0), success on Nth retry, exhaustion of retries.

### Step 3: Publish retry-announce synthetic events

**File:** `packages/server/src/runner/laneRunner.ts` (inside `retryStep`)

**Action:** Before each retry invocation (attempt ≥ 1), call `publish(stepId, retryEvent)` where `retryEvent` is an `UnknownEvent`-compatible object:
```ts
{ type: 'retry', timestamp: Date.now(), stepId, attempt: N, retryLimit }
```
This forwards through the event bus to all registered SSE subscribers for `stepId`, making the retry visible in the live lane output stream without changing the SSE transport layer.

**Verification:** Test asserts `publish` is called with `type: 'retry'` exactly `N-1` times when a step fails and succeeds on attempt N.

### Step 4: Persist `retryCount` in `StepRecord` after step resolution

**File:** `packages/server/src/runner/stepRunner.ts`

**Action:** The `executeStep` function currently builds the completed `StepRecord` after the subprocess resolves. Extend the function signature to accept an optional `retryCount: number` parameter (default `0`). Include `retryCount` in the `completedRecord` written to `stepLog`.

Alternatively — and preferably — `retryStep` in `laneRunner.ts` can update the `stepLog` entry directly after `retryStep` returns, by reading the current state, finding the existing record keyed by `stepId`, and setting `retryCount` on it before calling `writeSidecar`. This avoids touching `executeStep`'s signature (which ARC-1307 also reads).

**Chosen sub-approach:** `laneRunner.ts` updates the stepLog entry after `retryStep` returns. The update is a single synchronous `getState/setState/writeSidecar` triple, consistent with existing pattern in the file.

**Verification:** After a step that required 2 retries, `getState().stepLog[stepId].retryCount === 2`.

### Step 5: Update `runLane` call site

**File:** `packages/server/src/runner/laneRunner.ts`

**Action:** In the `runLane` agent-step block, replace:
```ts
result = await executeStep(step.id, prompt);
```
with:
```ts
const { result, retryCount } = await retryStep(step.id, prompt, getState().config.retryLimit);
```
Then, after `retryStep` returns, update `stepLog` with `retryCount` (per Step 4). The failure branch (`result.outcome === 'failed'`) already pauses the lane — no change needed there.

**Verification:** Existing `laneRunner.test.ts` test "transitions to paused and does not execute subsequent steps" continues to pass. New tests added in Step 6 cover retry-specific behaviour.

### Step 6: Add tests for retry behaviour

**File:** `packages/server/src/__tests__/laneRunner.test.ts`

**Action:** Add a new describe block `runLane — retry on failure (ARC-1306)` with test scenarios:
- `retryLimit: 0` — step fails once, lane pauses, `executeStep` called exactly once.
- `retryLimit: 2`, step fails twice then succeeds — `executeStep` called 3 times, lane continues (not paused), next step executes.
- `retryLimit: 2`, step fails all 3 attempts — `executeStep` called 3 times, lane paused.
- Retry announce event published: `publish` called `N-1` times with `type: 'retry'` when step succeeds on attempt N.
- `retryCount` reflected in `stepLog` after a step that required retries.

**Verification:** All new and existing tests pass under `pnpm test` in the server package.

## Testing & Validation

Run from `packages/server`:
```
pnpm test
```
All existing tests must remain green. The new describe block must pass all scenarios listed in Step 6.

End-to-end smoke: configure a session with `retryLimit: 2`, mock the subprocess to fail twice then succeed; assert lane status is `running` (not `paused`) after the step and that the SSE stream received two `retry` events.

## Risks & Open Questions

- **ARC-1307 coordination:** ARC-1307 will read `retryCount` from `StepRecord` to escalate agent level on each retry. The `retryCount` field added in Step 1/4 is the shared signal. ARC-1307's implementation plan must read this field — no additional change needed here.
- **`executeStep` signature change:** Step 4 deliberately avoids changing `executeStep`'s public signature to keep ARC-1307 unblocked. If ARC-1307 requires per-attempt agent-level injection into `executeStep`, that is ARC-1307's scope.
- **`retryLimit` from state vs. parameter:** `retryStep` reads `retryLimit` from `getState().config.retryLimit` at call time (not captured before the loop). This means a mid-session config change would be reflected on the next retry. This is acceptable — config mutation mid-session is not in scope for this story.
- **Concurrent retry events:** Each lane is sequential (ARC-1291). Two concurrent lanes retrying different steps publish to different stepId channels on the event bus, so no cross-lane event bleed.

