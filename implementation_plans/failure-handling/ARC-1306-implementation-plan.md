# ARC-1306 Retry Failed Agent Steps Up to Configurable Limit - Implementation Plan
**Issue:** `issues/failure-handling/ARC-1306-retry-failed-steps.md`
**Completion Summary:** `task-completions/failure-handling/ARC-1306-retry-failed-steps-COMPLETION-SUMMARY.md` (TBD)
**Approach:** retry loop in laneRunner with retryCount persisted on StepRecord
**Owner:** build
**Date:** 2026-06-30

## Scope & Alignment

This plan implements automatic retry of failed runbook steps up to a configurable limit. It covers:

- Extending `StepRecord` with a `retryCount` field
- Adding a `RetryEvent` to the `OpenCodeEvent` union for UI visibility
- Replacing the single-attempt step execution in `runLane()` with a retry loop that reads `retryLimit` from session config
- Persisting `retryCount` to the sidecar after each retry increment
- Unit tests covering retry success, retry exhaustion, event emission, and sidecar persistence

**AC mapping:**
- AC 1 (step fails with retries remaining → new subprocess spawned): covered by Steps 3 and 4
- AC 2 (retry limit reached → no further auto-retry, escalation triggered): covered by Steps 4 and 5

Out of scope: manual retry triggered by supervisor; escalation tier selection (ARC-1307).

## Assumptions & Dependencies

- `SessionConfig.retryLimit` already exists in `src/state/types.ts` with `DEFAULT_SESSION_CONFIG.retryLimit = 3`; no config schema changes needed.
- `getState().config.retryLimit` is accessible inside `runLane()` via the existing `getState()` import in `src/runner/laneRunner.ts`.
- `runStep()` in `src/runner/subprocessManager.ts` returns `RunResult { outcome: success | failed, exitCode, events }` — value-based, no thrown exceptions.
- `publish(stepId, event)` in `src/runner/eventBus.ts` accepts any member of the `OpenCodeEvent` union and is already imported in `laneRunner.ts`.
- `writeSidecar()` in `src/state/sidecar.ts` accepts the full session state and is available in the runner layer.
- The `ErrorEvent.data.isRetryable` optional field exists but is not used to gate retries in this story — all failures are retryable up to `retryLimit`.
- TypeScript strict mode is enabled; all new fields require initialization at construction sites.

## Implementation Steps

### Step 1: Add `retryCount` to `StepRecord`
**Files:** `src/state/types.ts`
**Action:** Add the field `retryCount: number` to the `StepRecord` interface. This field records how many retry attempts have been made for a given step execution, starting at zero.
**Verification:** `npx tsc --noEmit` in `packages/server` completes without errors. Any existing code constructing a `StepRecord` literal will produce a type error if the field is missing, which drives the fix in Step 3.

### Step 2: Add `RetryEvent` to `OpenCodeEvent`
**Files:** `src/runner/types.ts`
**Action:** Define a new event variant `RetryEvent` with shape `{ type: retry; stepId: string; attempt: number; maxAttempts: number }` and add it to the `OpenCodeEvent` discriminated union. The `attempt` field is the 1-based retry attempt number; `maxAttempts` equals `retryLimit`.
**Verification:** Any exhaustive `switch` over `OpenCodeEvent.type` in the codebase produces a TypeScript compile error for the unhandled `retry` case, confirming union membership. Add a `default` or explicit `retry` arm to each such consumer. `npx tsc --noEmit` passes after.

### Step 3: Initialize `retryCount` in `executeStep`
**Files:** `src/runner/stepRunner.ts`
**Action:** At every site where a `StepRecord` is created or reset at the start of a step execution, set `retryCount: 0`. This ensures the field is always defined before the retry loop in `laneRunner.ts` reads or increments it.
**Verification:** `npx tsc --noEmit` passes (satisfying the type error surfaced in Step 1). Confirm via test that a freshly started step has `retryCount === 0`.

### Step 4: Implement retry loop in `runLane`
**Files:** `src/runner/laneRunner.ts`
**Action:** Replace the single `executeStep()` call for each step with a loop. The loop:
1. Reads `retryLimit` from `getState().config.retryLimit` once before the step begins.
2. Calls `executeStep()` (which internally calls `runStep()`).
3. On `result.outcome === success`: breaks out of the retry loop and continues to the next step.
4. On `result.outcome === failed` and `stepRecord.retryCount < retryLimit`: increments `stepRecord.retryCount`, emits a `RetryEvent` via `publish(stepId, { type: retry, stepId, attempt: stepRecord.retryCount, maxAttempts: retryLimit })`, persists the updated record via `writeSidecar()`, then iterates.
5. On `result.outcome === failed` and retries exhausted (`stepRecord.retryCount >= retryLimit`): sets lane status to `paused` in `laneStatusMap`, logs a structured message describing retry exhaustion (step ID, final retry count, outcome), and returns — leaving the door open for ARC-1307 escalation logic to read the paused state.

The loop must be bounded: maximum iterations equal `retryLimit + 1` (initial attempt plus retries).
**Verification:** Unit tests (Step 6) pass. Manual inspection confirms the loop cannot execute more than `retryLimit + 1` times.

### Step 5: Persist `retryCount` in sidecar after each retry increment
**Files:** `src/runner/laneRunner.ts`, `src/state/sidecar.ts`
**Action:** Confirm that the `writeSidecar()` call inside the retry branch (Step 4, iteration 4) writes the full session state including the updated `StepRecord.retryCount`. No changes to `writeSidecar()` itself are expected — verify it already serializes the full state graph that includes `StepRecord`.
**Verification:** In the integration/unit test for "step fails once, succeeds on retry", assert that the sidecar written after the first failure contains `retryCount: 1`.

### Step 6: Write tests
**Files:** `src/__tests__/laneRunner.test.ts`
**Action:** Add or extend test scenarios covering:
- **Retry-then-success:** Mock `runStep` to fail on the first call and succeed on the second. Assert lane continues past the step, final `retryCount === 1`, and a `RetryEvent` with `attempt: 1` was published.
- **Retry exhaustion:** Mock `runStep` to fail `retryLimit + 1` times. Assert lane is paused, `retryCount === retryLimit`, and no additional calls to `runStep` occur after exhaustion.
- **RetryEvent emission:** Assert that one `RetryEvent` is published per retry attempt, with correct `attempt` and `maxAttempts` values.
- **Sidecar persistence:** Assert that `writeSidecar` is called with a state containing the incremented `retryCount` after each retry.
**Verification:** `npm test` in `packages/server` passes with all new scenarios green.

## Testing & Validation

Run `npm test` in `packages/server` after implementing all steps. All pre-existing tests must continue to pass.

End-to-end validation: start a local runbook session with a step whose prompt is designed to fail (e.g., references a non-existent tool). Confirm via sidecar inspection that `retryCount` increments on each attempt, `RetryEvent` entries appear in the steps event log, and the lane transitions to `paused` after `retryLimit` failures.

Type-safety validation: run `npx tsc --noEmit` in `packages/server` and confirm zero errors.

## Risks & Open Questions

- **`executeStep` reset behavior:** If `executeStep` resets the `StepRecord` on each invocation (re-initializing `retryCount: 0`), the retry counter will never accumulate. The retry loop in `laneRunner.ts` must own the `retryCount` increment and pass the existing record through, or `executeStep` must receive a flag to skip re-initialization on retry. Confirm the call signature during implementation.
- **Concurrency:** `runLane` is sequential within a lane. No concurrent retry concerns within a single lane. Cross-lane state access via `getState()` is read-only for `retryLimit`, so no race condition is introduced.
- **`retryLimit = 0`:** When `retryLimit` is zero, the loop executes once (initial attempt), fails, and immediately pauses the lane. This is the correct behavior and should be covered by a test assertion.

---
