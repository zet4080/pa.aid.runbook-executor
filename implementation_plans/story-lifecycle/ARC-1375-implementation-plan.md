# ARC-1375 Extract generic PR-review-loop sub-machine parametrized by artifact kind - Implementation Plan

**Issue:** `issues/story-lifecycle/ARC-1375-extract-generic-pr-review-loop-sub-machine.md`
**Completion Summary:** `task-completions/ARC-1375-COMPLETION-SUMMARY.md` (TBD)
**Approach:** Single approach — pattern established. `checkpointGit.ts` already contains the two primitives this sub-machine composes (`isPRMerged`, `listUnresolvedPRComments`), both consumed via the same fetch-mocking test pattern already used throughout `checkpointGit.test.ts`. No competing design exists to choose between; Phase 1 (Approach) is skipped per the write-implementation-plan skill's rule that it may be skipped when exploration reveals a clear existing pattern to follow.
**Owner:** build agent
**Date:** 2026-07-12

## Scope & Alignment

This plan adds exactly one new shared function (plus its supporting types) to `packages/server/src/git/checkpointGit.ts`, and its unit tests to `packages/server/src/git/checkpointGit.test.ts`. It formalizes the resume/gate decision logic described in the issue's Goal into a single, artifact-kind-agnostic implementation — it does **not** wire this function into any real checkpoint/resume route or lane runner call site (that is explicitly Out of Scope, deferred to ARC-1376 and ARC-1378).

Acceptance criteria mapping:
- **AC1** (merged PR → "advance" regardless of `artifactKind`) → Step 1 (merge-check-first control flow) + Step 2 (3 tests: one per `artifactKind`, merged path).
- **AC2** (not merged, unresolved comments → trigger comment-addressing, re-queue) → Step 1 (`addressComments` callback invocation branch) + Step 2 (3 tests: one per `artifactKind`, comments-present path, asserting the injected callback was called with the unresolved comments and the returned outcome/message).
- **AC3** (not merged, zero unresolved comments → silent re-queue, no agent action, no error) → Step 1 (`still_awaiting_merge` branch, callback never invoked) + Step 2 (3 tests: one per `artifactKind`, zero-comments path, asserting the callback was NOT called and no error is thrown/returned).
- **AC4** (identical behavior across all three `artifactKind` values except kind-specific messaging/labels; single implementation) → Step 1 (one function driven by a single `ARTIFACT_KIND_LIFECYCLE_STATES` lookup table, not three near-duplicate functions) + Step 2 (the 9 required tests collectively prove the same three outcomes occur for `'plan'`, `'implementation'`, and `'decision'`, differing only in the returned `awaitingState`/`addressingState`/`renotifyMessage` text).

## Assumptions & Dependencies

- **Dependency ARC-1374 (satisfied, merged to `main`):** `StoryLifecycleState` (16-value union, including `awaiting_plan_review`, `addressing_plan_comments`, `awaiting_implementation_review`, `addressing_implementation_comments`, `awaiting_decision_review`, `addressing_decision_comments`) is already exported from `packages/server/src/state/types.ts` on `main` (commit `28637bb`, "Merged in ARC-1374"). This plan imports that type only — it does not add new lifecycle states.
- **No persistence in this story:** the new function does not call `getState`/`setState`/`writeSidecar`/`getStoryLifecycleState`/`setStoryLifecycleState`. It is a pure decision/orchestration function that returns the `awaitingState`/`addressingState` StoryLifecycleState values as data; the caller (a later story) is responsible for persisting them. This matches the issue's Out of Scope ("wiring this into the actual planning-artifact gate... this story only builds the reusable mechanism").
- **`isPRMerged` and `listUnresolvedPRComments` are called directly (not injected)** — both already live in `checkpointGit.ts` and are already unit-tested there via the file's established `vi.stubGlobal('fetch', ...)` pattern; there is no existing precedent in this codebase for injecting sibling in-module functions as dependencies, and doing so would add indirection with no testability benefit (the new function's tests can control both primitives' behavior by mocking `fetch` responses per-URL).
- **The "revise artifact file, commit, push" side effect is injected, not implemented here** — the shared function cannot know how to revise a plan file vs. a decision document vs. an implementation diff (that logic belongs to each future caller in ARC-1376/ARC-1378). The function accepts an `addressComments` callback and awaits it; this is the seam that keeps "single implementation, not three near-duplicates" (AC4) true even after real wiring lands later.
- **Degraded-mode fallback on API failure (design decision, not explicit in the ACs):** if `isPRMerged` throws, the function logs a warning and treats the PR as not-merged (falls through to the comment check) rather than propagating the error or guessing "advance". If `listUnresolvedPRComments` throws, the function logs a warning and returns `still_awaiting_merge` (the same outcome as the zero-comments case) rather than propagating the error or invoking `addressComments` with stale/unknown data. Both choices favor "no rejection and no agent action" (the issue's own language for the zero-comments branch) over guessing in the face of a transient network/env-var failure. This mirrors the codebase-wide convention (already used in `checkpoints.ts`'s `listUnresolvedPRComments` `.catch()` and `startPlanningPRPoller`'s per-tick `.catch()`) of never letting a single failed Bitbucket API call crash or permanently misroute a fire-and-forget flow.
- **Errors thrown by the caller-supplied `addressComments` callback are NOT swallowed** — they propagate out of `resumeArtifactPRReviewLoop` unchanged. The function's responsibility is orchestration/decision-making; per-comment reply/resolve resiliency (as already implemented in `checkpoints.ts`'s `addressPRComments`) is the caller's concern when it supplies the callback, not this shared function's.

## Implementation Steps

### Step 1 — Add `resumeArtifactPRReviewLoop` and its supporting types to `checkpointGit.ts`

**Files:** `packages/server/src/git/checkpointGit.ts`

**Action:**

Add a new type-only import from `../state/types.js`: `type { StoryLifecycleState }`.

Add a new section (mirroring the file's existing `// --- ARC-XXXX: ... ---` section-comment convention), placed after the `resolveThread` function at the end of the file, containing:

- `export type ArtifactKind = 'plan' | 'implementation' | 'decision';` — the three kinds named in the issue goal.
- `export type ArtifactPRReviewOutcome = 'advance' | 'addressing_comments' | 'still_awaiting_merge';` — the three resume branches from the issue goal, one per AC1/AC2/AC3.
- `export interface ArtifactKindLifecycleStates { awaitingState: StoryLifecycleState; addressingState: StoryLifecycleState; }`
- `export const ARTIFACT_KIND_LIFECYCLE_STATES: Record<ArtifactKind, ArtifactKindLifecycleStates>` — a literal object mapping `'plan'` → `{ awaitingState: 'awaiting_plan_review', addressingState: 'addressing_plan_comments' }`, `'implementation'` → `{ awaitingState: 'awaiting_implementation_review', addressingState: 'addressing_implementation_comments' }`, `'decision'` → `{ awaitingState: 'awaiting_decision_review', addressingState: 'addressing_decision_comments' }`. This single table is what makes AC4 ("single implementation, not three near-duplicates") true — the control flow below reads from this table instead of branching on `artifactKind` with duplicated logic.
- `export interface ResumeArtifactPRReviewLoopOptions { prUrl: string; artifactKind: ArtifactKind; addressComments: (unresolvedComments: BitbucketPRComment[]) => Promise<void>; }`
- `export interface ResumeArtifactPRReviewLoopResult { outcome: ArtifactPRReviewOutcome; awaitingState: StoryLifecycleState; addressingState: StoryLifecycleState; renotifyMessage?: string; }` — `renotifyMessage` is only populated when `outcome === 'addressing_comments'`, following the existing `CheckpointQueueEntry.renotifyMessage` convention and format (`Agent addressed N comment(s) — [view changes]`), extended with the artifact-kind name so the three kinds produce distinguishable messages while sharing one code path (AC4).
- `export async function resumeArtifactPRReviewLoop(opts: ResumeArtifactPRReviewLoopOptions): Promise<ResumeArtifactPRReviewLoopResult>` implementing this control flow:
  1. Look up `{ awaitingState, addressingState }` from `ARTIFACT_KIND_LIFECYCLE_STATES[opts.artifactKind]`.
  2. Call `isPRMerged(opts.prUrl)`. On success `true` → return `{ outcome: 'advance', awaitingState, addressingState }` immediately, without checking comments at all (AC1: merged wins regardless of comment state). On thrown error, log via `console.warn('[ARC-1375] isPRMerged failed for ...')` and proceed to step 3 as if not merged (degraded-mode fallback per Assumptions).
  3. Call `listUnresolvedPRComments(opts.prUrl)`. On thrown error, log via `console.warn('[ARC-1375] listUnresolvedPRComments failed for ...')` and return `{ outcome: 'still_awaiting_merge', awaitingState, addressingState }` (degraded-mode fallback per Assumptions). On success with `length === 0` → return `{ outcome: 'still_awaiting_merge', awaitingState, addressingState }` (AC3 — no `renotifyMessage`, no call to `addressComments`). On success with `length > 0` → `await opts.addressComments(comments)` (propagating any error it throws, per Assumptions), then build `renotifyMessage` as `` `Agent addressed ${n} ${opts.artifactKind} comment${n === 1 ? '' : 's'} — [view changes]` `` and return `{ outcome: 'addressing_comments', awaitingState, addressingState, renotifyMessage }` (AC2).

**Verification:** `run_typecheck` on `packages/server` compiles cleanly with the new exports; no existing export or function signature in `checkpointGit.ts` is modified (only additive).

### Step 2 — Unit tests for all three resume branches × all three artifact kinds

**Files:** `packages/server/src/git/checkpointGit.test.ts`

**Action:**

Add a new local test helper (placed near the file's existing `mockFetch` helper, following its style) that routes `fetch` responses by URL substring instead of returning one canned response for every call — needed because `resumeArtifactPRReviewLoop` makes up to two sequential `fetch` calls (the PR-detail GET consumed by `isPRMerged`, then the comments GET consumed by `listUnresolvedPRComments`) that must return different bodies in the same test. The helper takes an array of `{ match: string; ok: boolean; json?: () => Promise<unknown> }` route entries (matched via `url.includes(match)`, checked in order) and `vi.stubGlobal('fetch', ...)` with a mock that inspects the call's URL argument and returns the first matching route's response; an unmatched URL throws a descriptive error so a test-setup mistake fails loudly rather than silently returning `{}`.

Add a new `describe('resumeArtifactPRReviewLoop', ...)` block (placed after the existing `describe('resolveThread', ...)` block) with the required env-var `beforeEach`/`afterEach` pattern already used elsewhere in the file, containing:

- Three tests (one per `artifactKind`: `'plan'`, `'implementation'`, `'decision'`) for the **merged** branch: route the PR-detail URL to `{ state: 'MERGED' }`; assert the result is `{ outcome: 'advance', awaitingState: <kind's awaiting state>, addressingState: <kind's addressing state> }`; assert the injected `addressComments` mock was never called; assert `fetch` was called exactly once (comments endpoint never queried, per AC1's "regardless of" language — merged short-circuits the comment check).
- Three tests (one per `artifactKind`) for the **not-merged, comments present** branch: route the PR-detail URL to `{ state: 'OPEN' }` and the comments URL to `{ values: [{ id: 1, content: { raw: 'fix this' }, resolved: false }] }`; assert the injected `addressComments` mock was called once with that one-element comments array; assert the result is `{ outcome: 'addressing_comments', awaitingState, addressingState, renotifyMessage }` where `renotifyMessage` contains both `'1'` and the artifact-kind string (e.g. asserts `renotifyMessage` matches a pattern containing `1 plan comment`, `1 implementation comment`, or `1 decision comment` respectively).
- Three tests (one per `artifactKind`) for the **not-merged, zero comments** branch: route the PR-detail URL to `{ state: 'OPEN' }` and the comments URL to `{ values: [] }`; assert the injected `addressComments` mock was never called; assert the result is `{ outcome: 'still_awaiting_merge', awaitingState, addressingState }` with `renotifyMessage` undefined; assert the function does not throw (direct evidence for AC3's "no rejection").
- One resiliency test: PR-detail fetch call throws a network error (route entry with a rejecting implementation, or a non-2xx `ok: false` response) → comments endpoint still queried and returns zero comments → result is `still_awaiting_merge`, confirming the degraded-mode fallback from Assumptions does not crash the caller.
- One resiliency test: PR-detail returns `OPEN`, comments fetch throws/returns non-2xx → result is `still_awaiting_merge`, `addressComments` never called.
- One resiliency test: PR-detail returns `OPEN`, comments returns one unresolved comment, and the injected `addressComments` mock is set to reject → `resumeArtifactPRReviewLoop(...)` rejects with the same error (propagation, not swallowing, per Assumptions).

**Verification:** `run_tests` — all 12 new tests pass; no existing test in `checkpointGit.test.ts` changes behavior (confirms the addition is purely additive).

### Step 3 — Full verification pass

**Files:** N/A (verification only)

**Action:** Run the full `packages/server` test suite, typecheck, and lint to confirm the new function and its tests introduce zero regressions elsewhere (in particular `laneRunner.test.ts` and `checkpoints.test.ts`, which mock `checkpointGit.js` wholesale and therefore must continue to pass unmodified since neither references `resumeArtifactPRReviewLoop`).

**Verification:** `run_typecheck` returns clean; `run_tests` returns all green with only `checkpointGit.ts` and `checkpointGit.test.ts` modified; `run_lint` reports no new violations.

## Testing & Validation

- `run_typecheck` — confirms the new exported types (`ArtifactKind`, `ArtifactPRReviewOutcome`, `ArtifactKindLifecycleStates`, `ResumeArtifactPRReviewLoopOptions`, `ResumeArtifactPRReviewLoopResult`) and the `resumeArtifactPRReviewLoop` signature compile cleanly, including the type-only import of `StoryLifecycleState` from `../state/types.js`.
- `run_tests` — confirms:
  - The 9 required scenarios (3 branches × 3 artifact kinds) all pass, directly satisfying AC1, AC2, AC3, and AC4 (AC4 is satisfied by the fact that all 9 scenarios run through the same function body, differing only in the table-driven `awaitingState`/`addressingState`/`renotifyMessage` values).
  - The 3 resiliency scenarios pass, proving the degraded-mode fallback design decision behaves as documented.
  - The full existing `checkpointGit.test.ts` suite and the wider `packages/server` suite (`laneRunner.test.ts`, `checkpoints.test.ts` in particular, since both mock the whole `checkpointGit.js` module) pass unmodified.
- `run_lint` — confirms no style violations in the two edited files.

## Risks & Open Questions

- **This mechanism is intentionally unwired until ARC-1376/ARC-1378.** After this story merges, `resumeArtifactPRReviewLoop` exists and is fully tested but is not called from `checkpoints.ts`'s resume route or `laneRunner.ts`'s `resumeLaneFromPlanningPR`. Those two existing call sites keep their current ARC-1304/ARC-1371 logic unchanged until the later stories migrate them onto this shared function. This is expected, per the issue's Out of Scope.
- **Degraded-mode fallback on `isPRMerged`/`listUnresolvedPRComments` failure is a design decision beyond the literal ACs**, documented above and covered by 3 supplementary tests. If a future story (ARC-1376/ARC-1378) needs different failure semantics once real wiring happens (e.g., surfacing the failure to the supervisor rather than silently requeuing), that is a decision for that story, not a gap in this one.
- **`addressComments` callback contract is intentionally minimal** (`(comments) => Promise<void>`, no `prUrl`/`artifactKind`/`repoRoot` parameters) — the future caller in ARC-1376/ARC-1378 is expected to close over those values when constructing the callback (e.g., a closure capturing `entry.prUrl` and the artifact file path). If a future story finds it needs additional context passed explicitly rather than via closure, that is a non-breaking additive change to `ResumeArtifactPRReviewLoopOptions`, not a redesign.
