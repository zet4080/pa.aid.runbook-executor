# ARC-1375 Extract generic PR-review-loop sub-machine parametrized by artifact kind — Completion Summary

**Issue:** `issues/story-lifecycle/ARC-1375-extract-generic-pr-review-loop-sub-machine.md`
**Implementation Plan:** `implementation_plans/story-lifecycle/ARC-1375-implementation-plan.md`
**Completed:** 2026-07-12
**Branch:** `ARC-1375` (worktree `/repos/ARC-1375`, off `main` in `pa.aid.conductor.ts`)
**Commit:** `2c81ef2` — "feat(story-lifecycle): extract generic PR-review-loop sub-machine parametrized by artifact kind"

## Acceptance Criteria Verification

| # | Criterion | Status | Evidence |
|---|-----------|--------|----------|
| 1 | Given an artifact PR is open and merged, when resume is triggered, then the shared function reports "advance" regardless of artifactKind | Passed | `checkpointGit.test.ts` `describe('resumeArtifactPRReviewLoop')` → `describe.each(ARTIFACT_KINDS)` → `'merged PR → outcome "advance", addressComments never called, fetch called once'` — 3 tests (one per `'plan'`/`'implementation'`/`'decision'`), each asserting `result` equals `{ outcome: 'advance', awaitingState, addressingState }` and that `fetch` is called exactly once (comments endpoint never queried). `npx vitest run src/git/checkpointGit.test.ts` → all pass. |
| 2 | Given an artifact PR is open, not merged, with unresolved comments, when resume is triggered, then the shared function triggers comment-addressing (revise file, commit, push) and re-queues the checkpoint | Passed | Same `describe.each` block → `'not merged, unresolved comments present → triggers addressComments and re-queues'` — 3 tests, each asserting the injected `addressComments` mock was called once with the one-element unresolved-comments array, and the result is `{ outcome: 'addressing_comments', awaitingState, addressingState, renotifyMessage }` where `renotifyMessage` contains `1 <kind> comment`. |
| 3 | Given an artifact PR is open, not merged, with zero unresolved comments, when resume is triggered, then the shared function re-queues the checkpoint silently with no agent action and no error/rejection | Passed | Same `describe.each` block → `'not merged, zero unresolved comments → silently re-queues, no agent action, no error'` — 3 tests, each asserting `addressComments` was never called, result is `{ outcome: 'still_awaiting_merge', awaitingState, addressingState }`, and `renotifyMessage` is `undefined`. No throw occurs (direct evidence of "no rejection"). |
| 4 | Given three different artifactKind values are passed, when the function runs, then behavior is identical except for artifact-kind-specific messaging/labels (single implementation, not three near-duplicates) | Passed | Structural: `resumeArtifactPRReviewLoop` is one function (not three) that reads `{ awaitingState, addressingState }` from the single `ARTIFACT_KIND_LIFECYCLE_STATES` lookup table and branches only on `isPRMerged`/comment-count outcomes, never on `artifactKind` string comparisons in the control flow. Test-level proof: all 9 required scenarios (3 branches × 3 kinds) are expressed as one `describe.each(ARTIFACT_KINDS)` block running the identical three `it(...)` bodies against each kind — the only per-kind variance is the expected `awaitingState`/`addressingState`/`renotifyMessage` text, confirming AC4. |

## Implementation Summary

Added to `packages/server/src/git/checkpointGit.ts` (additive only — no existing export or signature modified):

- `ArtifactKind` — `'plan' | 'implementation' | 'decision'`
- `ArtifactPRReviewOutcome` — `'advance' | 'addressing_comments' | 'still_awaiting_merge'`
- `ArtifactKindLifecycleStates` interface and `ARTIFACT_KIND_LIFECYCLE_STATES` — the single lookup table mapping each `ArtifactKind` to its `{ awaitingState, addressingState }` pair of `StoryLifecycleState` values (imported type-only from `../state/types.js`, per the ARC-1374 dependency)
- `ResumeArtifactPRReviewLoopOptions` / `ResumeArtifactPRReviewLoopResult` interfaces
- `resumeArtifactPRReviewLoop(opts)` — the shared resume/gate decision function:
  1. Calls `isPRMerged(prUrl)`. On `true` → returns `{ outcome: 'advance', ... }` immediately, without checking comments (AC1: merged wins regardless of comment state). On throw, logs a `console.warn` and proceeds as if not merged (degraded-mode fallback).
  2. Calls `listUnresolvedPRComments(prUrl)`. On throw, logs a `console.warn` and returns `still_awaiting_merge` (degraded-mode fallback — same outcome as zero comments, never guesses "advance" or invokes the callback with unknown data).
  3. Zero comments → returns `still_awaiting_merge` silently, no callback invocation (AC3).
  4. One or more comments → awaits the caller-supplied `addressComments(comments)` callback (propagating any error it throws, unmodified), then returns `{ outcome: 'addressing_comments', renotifyMessage }` where `renotifyMessage` follows the existing `CheckpointQueueEntry.renotifyMessage` convention/format used elsewhere in the codebase (`checkpoints.ts`), extended with the artifact-kind name (AC2).

This mechanism is intentionally **not wired** into `checkpoints.ts`'s resume route or `laneRunner.ts`'s `resumeLaneFromPlanningPR` — those two existing ARC-1304/ARC-1371 call sites are unchanged. Wiring is explicitly Out of Scope for this story, deferred to ARC-1376 (planning-artifact gate) and ARC-1378 (new implementation-review gate).

## Verification Steps

```bash
# Typecheck (packages/server)
cd /repos/ARC-1375/packages/server && npx tsc --noEmit
# → clean, no output

# Run the new function's tests in isolation (12 tests)
cd /repos/ARC-1375/packages/server && npx vitest run src/git/checkpointGit.test.ts -t "resumeArtifactPRReviewLoop"
# → Tests 12 passed | 41 skipped (53)

# Run the full checkpointGit.test.ts suite (53 tests: 41 pre-existing + 12 new)
cd /repos/ARC-1375/packages/server && npx vitest run src/git/checkpointGit.test.ts
# → Tests 53 passed (53)

# Run the full packages/server suite — confirms zero regressions,
# in particular laneRunner.test.ts (90 tests) and checkpoints.test.ts
# (20 tests), both of which mock checkpointGit.js wholesale.
cd /repos/ARC-1375/packages/server && npx vitest run
# → Test Files 22 passed (22); Tests 404 passed (404)
```

`run_lint` reports no lint script configured in this repo (`No lint script found in package.json`) — not applicable; no other lint tool detected.

## Tests Added/Modified

- `packages/server/src/git/checkpointGit.test.ts`:
  - New `mockFetchRoutes()` test helper — routes `fetch` responses by URL substring (in order), needed because `resumeArtifactPRReviewLoop` makes up to two sequential `fetch` calls (PR-detail GET then comments GET) that must return different bodies in the same test. Throws loudly on an unmatched URL rather than silently returning `{}`.
  - New `describe('resumeArtifactPRReviewLoop', ...)` block with 12 tests:
    - 9 tests via `describe.each(ARTIFACT_KINDS)` — 3 branches (merged/advance, unresolved-comments/addressing, zero-comments/still-awaiting) × 3 artifact kinds (`'plan'`, `'implementation'`, `'decision'`)
    - 3 resiliency tests: `isPRMerged` throws → falls through to comment check; `listUnresolvedPRComments` throws/non-2xx → `still_awaiting_merge`, callback never invoked; `addressComments` rejects → error propagates unmodified (not swallowed)
  - No existing test in the file was modified; all 41 pre-existing tests pass unchanged.

## Rollback Notes

None — purely additive change to `checkpointGit.ts` and `checkpointGit.test.ts`. No call sites reference the new exports yet, so reverting the commit has zero effect on any running behavior.

## Next Steps

- ARC-1376 — Migrate the planning-artifact (plan/decision) PR approval gate onto this shared sub-machine (supersedes ARC-1372).
- ARC-1378 — Add the new human implementation-review gate after self-review passes, using this shared sub-machine (depends on ARC-1375 and ARC-1377).
