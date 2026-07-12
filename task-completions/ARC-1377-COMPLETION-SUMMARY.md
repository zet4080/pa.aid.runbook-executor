# ARC-1377 Track agent self-review (local-code-review) state explicitly in conductor state — Completion Summary

**Issue:** `issues/story-lifecycle/ARC-1377-track-agent-self-review-state.md`
**Implementation Plan:** `implementation_plans/story-lifecycle/ARC-1377-implementation-plan.md`
**Completed:** 2026-07-12
**Branch:** `ARC-1377` (pa.aid.conductor.ts, branched off `main`), commit `84307d2`
**Duration:** Single session

## Acceptance Criteria Verification

| # | Criterion | Status | Evidence |
|---|-----------|--------|----------|
| 1 | Given an agent runs local-code-review during code execution, when the loop completes (pass or exhausted retries), then the outcome and iteration count are recorded in the per-story sidecar state. | Passed | `setSelfReviewRecord` (`packages/server/src/state/sidecar.ts`) persists `{ outcome, iterations, updatedAt }` under `SidecarState.selfReview[storyKey]`. Wired into `runLane` (`packages/server/src/runner/laneRunner.ts`) via `parseLocalCodeReviewOutcome`, which scans the final post-retry-loop `result.events` for the skill's own verdict markers. Test evidence: `sidecar.test.ts` → `describe('setSelfReviewRecord')` round-trip test "round-trips through writeSidecar + readSidecar (AC1 — persists across restarts)"; `laneRunner.test.ts` → `describe('runLane — self-review outcome tracking (ARC-1377)')` tests "clean outcome: records selfReview..." and "escalated outcome: records selfReview...". All pass: `npx vitest run src/__tests__/sidecar.test.ts src/__tests__/laneRunner.test.ts` → 139 passed. |
| 2 | Given self-review passes cleanly, when laneRunner.ts checks story state, then it can determine "self-review complete, clean" without re-invoking the skill. | Passed | `getSelfReviewRecord(state, storyKey)` is a pure read (no file I/O, no skill invocation) returning `SelfReviewRecord \| undefined`. On a clean verdict, `runLane` writes the record and does **not** pause the lane, falling through to existing pending-gate logic unchanged. Test evidence: `laneRunner.test.ts` "clean outcome..." test asserts `getState().selfReview['ARC-1377-clean']` equals `{ outcome: 'clean', iterations: 1, ... }`, `getLaneState(...) === 'done'`, `checkpointQueue` is empty, and explicitly calls `getSelfReviewRecord(state, 'ARC-1377-clean')?.outcome === 'clean'` to prove the queryable-without-reinvocation property. |
| 3 | Given self-review exhausts its retry limit still failing, when laneRunner.ts checks story state, then the story transitions to awaiting_escalation_checkpoint. | Passed | On an escalated verdict, `runLane` calls `setStoryLifecycleState(withRecord, storyKey, 'awaiting_escalation_checkpoint')` (reusing the ARC-1374 canonical lifecycle helper), pushes a high-level `CheckpointQueueEntry`, sets the lane to `'paused'`, and returns immediately (no fall-through). Test evidence: `laneRunner.test.ts` "escalated outcome..." test asserts `state.storyLifecycle['ARC-1377-escalate'].state === 'awaiting_escalation_checkpoint'`, `checkpointQueue` has exactly one high-level entry keyed by the step id, and `getLaneState(...) === 'paused'`. |

## Implementation Summary

Followed the plan's single specified approach — no competing designs:

1. **Types** (`packages/server/src/state/types.ts`): Added `SelfReviewOutcome = 'clean' | 'escalated'`, `SelfReviewRecord { outcome, iterations, updatedAt }`, and the optional `SidecarState.selfReview?: Record<string, SelfReviewRecord>` field, grouped near the existing ARC-1374 `StoryLifecycleState`/`StoryLifecycleRecord` block in the same additive-union-type style.

2. **Persistence helpers** (`packages/server/src/state/sidecar.ts`): Added pure functions `getSelfReviewRecord` (returns `undefined` when absent — no forced default, unlike `getStoryLifecycleState`) and `setSelfReviewRecord` (returns a new immutable `SidecarState` with the entry replaced), mirroring the exact style of the ARC-1374 story-lifecycle helpers.

3. **Detection + wiring** (`packages/server/src/runner/laneRunner.ts`): Added `parseLocalCodeReviewOutcome(events)`, a helper in the same family as the existing `extractLastAgentMessage`, which scans an agent step's final (post-retry-loop) `OpenCodeEvent[]` for the local-code-review skill's own mandated verdict text — `"✅ Local review clean (iteration N/3)."` or `"🔴 ESCALATE — local review iteration N/3 still has unresolved findings."` — taking the last chronological match across all events so that earlier "Iteration N/3" progress lines never produce a false positive. Wired into `runLane` immediately after the existing ARC-1306 retry-handling block closes and before the existing pending-gate check: a clean verdict persists the record with no lane pause; an escalated verdict persists the record, transitions the canonical `StoryLifecycleState` to `awaiting_escalation_checkpoint`, pushes a high checkpoint, pauses the lane, and returns.

4. **Tests**: Added `describe('getSelfReviewRecord')` / `describe('setSelfReviewRecord')` blocks to `sidecar.test.ts` (7 new tests: default-undefined behavior, present-but-missing-key, present-and-found, immutability, no-clobber across keys, overwrite-same-key, round-trip persistence, backward-compat load of a pre-ARC-1377 sidecar). Added `describe('runLane — self-review outcome tracking (ARC-1377)')` to `laneRunner.test.ts` (6 new tests: clean path, escalated path, no-marker regression guard, last-occurrence correctness, post-retry-loop detection, no-double-push guard against the existing ARC-1306 retry-exhausted checkpoint path).

No changes were made to the local-code-review skill's own logic, wording, or iteration cap — per the issue's explicit Out of Scope. No route changes were needed: `GET /api/state` already spreads the full `SidecarState` (so `selfReview` is automatically visible once populated), and `PATCH /api/state`'s allowlist already excludes internally-managed fields by construction (whitelist of `selectedRunbooks`/`checkpointQueue`/`config` only).

One local-code-review finding was caught and fixed during the review loop: an unused `getSelfReviewRecord` import in `laneRunner.ts` (the helper is consumed by `laneRunner.test.ts` directly, not by `runLane` itself, since `runLane` only ever *writes* the record — no real caller of the read path exists yet, matching the plan's stated precedent that ARC-1378 will be the first consumer). Removed the unused import; verified with a clean `npx tsc --noEmit`.

## Verification Steps

```bash
cd /repos/ARC-1377/packages/server
npx tsc --noEmit                 # clean, zero errors
npx vitest run                   # 407 passed (22 test files)
cd ../ui && npx tsc --noEmit     # clean, zero errors
cd /repos/ARC-1377 && npm test   # 407 server + 192 UI tests, all passed
```

## Tests Added/Modified

- `packages/server/src/__tests__/sidecar.test.ts` — added `describe('getSelfReviewRecord')` and `describe('setSelfReviewRecord')` (7 tests total), placed after the existing ARC-1374 `setStoryLifecycleState` block.
- `packages/server/src/__tests__/laneRunner.test.ts` — added `describe('runLane — self-review outcome tracking (ARC-1377)')` (6 tests total), placed after the existing ARC-1306 retry-behaviour block; extended the module-level `vi.mock('../state/sidecar.js', ...)` to include working (non-stub) implementations of `getSelfReviewRecord`, `setSelfReviewRecord`, and `setStoryLifecycleState` so the mocked module still exercises real state-shape logic under test.

## Rollback Notes

None. This is a purely additive change: one new optional `SidecarState` field, two new exported pure functions, one new internal helper function, and one new detection block in `runLane` that only activates when the local-code-review skill's own marker text is present in a step's events. Pre-existing sidecars with no `selfReview` key continue to load and behave identically (verified by the backward-compat test). No database/schema migration applies (`version: 1` unchanged).

## Next Steps

- ARC-1378 (new human implementation-review gate, depends on this story) will be the first real caller of `getSelfReviewRecord`. Not part of this story's scope.
