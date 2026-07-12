# ARC-1374 Add canonical StoryLifecycleState enum and per-story state field - Implementation Plan

**Issue:** `issues/story-lifecycle/ARC-1374-add-canonical-story-lifecycle-state-enum.md`
**Completion Summary:** `task-completions/ARC-1374-COMPLETION-SUMMARY.md` (TBD)
**Approach:** Single approach — pattern established. This story follows the existing "optional additive field with `??` default fallback" pattern already used for `prPollIntervalMs`, `conflictingSession`, `closedSessions`, and `stepLog` on `SidecarState`. No trade-off exists between competing designs; Phase 1 (Approach) is skipped per the write-implementation-plan skill's rule that Phase 1 may be skipped when exploration reveals a clear existing pattern to follow, even at HIGH risk tier.
**Owner:** build agent
**Date:** 2026-07-12

## Scope & Alignment

This plan covers the three In-Scope items from the issue, and only those:

1. `state/types.ts` additions — `StoryLifecycleState` (16-value union), `StoryLifecyclePhase` (`'plan' | 'code' | 'decision'`), a `StoryLifecycleRecord` interface, a `DEFAULT_STORY_LIFECYCLE_STATE` constant, and a new optional `storyLifecycle` field on `SidecarState`.
2. `sidecar.ts` persistence — two new pure helper functions (`getStoryLifecycleState`, `setStoryLifecycleState`) that read/write the new field on a `SidecarState` object. No I/O changes are needed to `readSidecar`/`writeSidecar` themselves — they already serialize/deserialize the whole `SidecarState` object generically.
3. Backward-compatible default handling — `getStoryLifecycleState` returns `DEFAULT_STORY_LIFECYCLE_STATE` (`'unclaimed'`) whenever `storyLifecycle` or the specific story key is absent, so pre-ARC-1374 sidecar files load and read without error or throw.

Explicitly NOT covered (per issue's Out of Scope): no call site in `laneRunner.ts`, `checkpointGit.ts`, or `checkpoints.ts` is modified to actually set/read this field during real execution, and no rollup derivation of legacy enums (ARC-1381) is implemented. This plan only adds the type, the persistence field, and the pure helper functions plus their unit tests — proving the field round-trips through `writeSidecar`/`readSidecar` and survives `reconcileState`/`loadOrCreateState`, without changing any existing runtime behavior.

Acceptance criteria mapping:
- **AC1** (types added, existing behavior unchanged) → Step 1 (additive types only, no existing field/type modified) + Step 5 (typecheck/lint/full existing test suite still green).
- **AC2** (per-story state field updated and persisted across restarts) → Step 2 (`setStoryLifecycleState`) + Step 3 (round-trip test: `setStoryLifecycleState` → `writeSidecar` → `readSidecar` → field present).
- **AC3** (older sidecar.json files without the field load without error, backward-compatible default) → Step 1 (field is optional: `storyLifecycle?: ...`) + Step 2 (`getStoryLifecycleState` default fallback) + Step 4 (regression test loading a pre-ARC-1374-shaped sidecar fixture and asserting no throw + default value).

## Assumptions & Dependencies

- No dependencies (issue states "None.").
- `SidecarState.version` stays `1` — this is an additive, backward-compatible field, matching the precedent of `prPollIntervalMs`, `conflictingSession`, `closedSessions`, and `stepLog`, none of which bumped the version when added.
- The per-story key used to index `storyLifecycle` is the bare issue key (e.g. `"ARC-1374"`), matching the `storyKey` convention already used throughout `laneRunner.ts`, `checkpointGit.ts`, `claimGit.ts`, `worktreeGit.ts`, and `planningArchiver.ts` (obtained via `stripNamespace(step.id)` at call sites — not touched by this story, but the key shape must match for future stories to wire it in without re-keying).
- `readSidecar`'s existing validity gate (`parsed.version !== 1` → treat as missing) is unaffected — it only inspects the `version` field, never `storyLifecycle`, so old files pass through unchanged.
- `routes/state.ts` (`GET`/`PATCH /api/state`) is not modified in this story. `GET /api/state` already spreads the full state object (`res.json({ ...state, laneState })`), so once `storyLifecycle` is populated by a future story it will be visible in the API response automatically, with no route change required. `PATCH /api/state`'s allowlist (`selectedRunbooks`, `checkpointQueue`, `config`) is left untouched — `storyLifecycle` is not externally patchable, consistent with `stepState` and `stepLog` being internally managed only.
- `reconcile.ts`'s `reconcileState`/`loadOrCreateState` use object-spread (`{ ...raw, stepState: ..., checkpointQueue: ... }`) which already preserves any field not explicitly overwritten — `storyLifecycle` passes through unchanged with zero code change to `reconcile.ts`. This plan adds a regression test to lock in that contract rather than changing the implementation.

## Implementation Steps

### Step 1 — Add lifecycle types and the sidecar field

**Files:** `packages/server/src/state/types.ts`

**Action:**
Add, near the top of the file (after `QueuePolicy`/before or after `SessionConfig` — grouped with the other standalone union-type declarations, following the existing style of `QueuePolicy`/`CheckpointLevel`/`LaneStatus`, which are plain string-literal unions rather than TypeScript `enum`):

- `export type StoryLifecycleState = 'unclaimed' | 'claimed' | 'worktree_provisioning' | 'agent_executing' | 'retrying' | 'awaiting_escalation_checkpoint' | 'awaiting_plan_review' | 'addressing_plan_comments' | 'agent_self_review' | 'awaiting_implementation_review' | 'addressing_implementation_comments' | 'awaiting_decision_review' | 'addressing_decision_comments' | 'completion_summary_drafting' | 'archiving' | 'archived';` — all 16 values exactly as enumerated in the issue goal, no additions/omissions.
- `export type StoryLifecyclePhase = 'plan' | 'code' | 'decision';`
- `export const DEFAULT_STORY_LIFECYCLE_STATE: StoryLifecycleState = 'unclaimed';`
- `export interface StoryLifecycleRecord { state: StoryLifecycleState; phase?: StoryLifecyclePhase; updatedAt: string; }` — `phase` optional because not every state is phase-scoped (e.g. `unclaimed`, `claimed`, `archived` have no associated review-loop phase); `updatedAt` is an ISO timestamp string, following the same convention as `CheckpointQueueEntry.hitAt`.

Add a new optional field to the `SidecarState` interface, placed after `stepLog` with a doc comment matching the style of the existing optional-field comments (e.g. `closedSessions`, `stepLog`):

- `storyLifecycle?: Record<string, StoryLifecycleRecord>;` — doc comment: "Per-story canonical lifecycle state (ARC-1374), keyed by bare story key (e.g. 'ARC-1374'). Absent on sidecars persisted before this story — callers use `getStoryLifecycleState` for a safe default. Not yet written to by any call site (ARC-1374 is types + persistence plumbing only; laneRunner/checkpointGit/checkpoints migration is a later story)."

**Verification:** `packages/server` compiles with `run_typecheck` — no type errors introduced. Confirm the union has exactly 16 members by counting entries against the issue's goal sentence.

### Step 2 — Add persistence helper functions to sidecar.ts

**Files:** `packages/server/src/state/sidecar.ts`

**Action:**
Import `DEFAULT_STORY_LIFECYCLE_STATE`, `StoryLifecyclePhase`, `StoryLifecycleState`, `StoryLifecycleRecord` alongside the existing `DEFAULT_SESSION_CONFIG`/`SidecarState` import from `./types.js`.

Add two new exported pure functions (no file I/O — consistent with `reconcileState`'s pure-transformation style; callers are responsible for `setState`/`writeSidecar`, matching the pattern every other mutation site in `laneRunner.ts` already follows):

- `getStoryLifecycleState(state: SidecarState, storyKey: string): StoryLifecycleState` — returns `state.storyLifecycle?.[storyKey]?.state ?? DEFAULT_STORY_LIFECYCLE_STATE`. This is the single defaulting choke point satisfying AC3: any reader (present or future) that goes through this function never observes `undefined`, regardless of whether the sidecar predates ARC-1374 or the specific story key has never been recorded.
- `setStoryLifecycleState(state: SidecarState, storyKey: string, newState: StoryLifecycleState, phase?: StoryLifecyclePhase): SidecarState` — returns a new `SidecarState` object with `storyLifecycle` shallow-copied and the entry for `storyKey` replaced by `{ state: newState, phase, updatedAt: new Date().toISOString() }`. Does not mutate the input `state` object (matches immutable-update style used everywhere else in this file and in `laneRunner.ts`/`reconcile.ts`).

**Verification:** `run_typecheck` passes. Unit tests in Step 3 exercise both functions directly.

### Step 3 — Unit tests for the new helpers and round-trip persistence

**Files:** `packages/server/src/__tests__/sidecar.test.ts`

**Action:** Add a new `describe('getStoryLifecycleState')` / `describe('setStoryLifecycleState')` block (placed after the existing `createFreshState` block, before the ARC-1310 lock-file section) covering these scenarios:

- `getStoryLifecycleState` returns `'unclaimed'` when `storyLifecycle` is entirely absent from the state object (simulating a pre-ARC-1374 sidecar).
- `getStoryLifecycleState` returns `'unclaimed'` when `storyLifecycle` is present but does not contain the queried `storyKey` (simulating a sidecar that already tracks some stories but not this one).
- `getStoryLifecycleState` returns the stored `state` value for a `storyKey` present in `storyLifecycle`.
- `setStoryLifecycleState` returns a new object (input `state` reference is unchanged / not mutated) with the new entry present under `storyLifecycle[storyKey]`.
- `setStoryLifecycleState` called twice for two different `storyKey`s preserves both entries (no clobbering) — exercises the shallow-copy-then-merge behavior.
- `setStoryLifecycleState` called twice for the same `storyKey` overwrites the prior entry for that key only.
- Round-trip: build a sample state via `makeSampleState()`, apply `setStoryLifecycleState`, then `writeSidecar` + `readSidecar` (the existing round-trip pattern already used in the `writeSidecar` describe block) — assert the read-back state's `storyLifecycle[storyKey]` deep-equals what was set. This is the direct evidence for AC2 ("updated and persisted across restarts" — restart is simulated by the fresh `readSidecar` call reading from disk rather than from the in-memory object).
- Backward-compat load: write a raw JSON sidecar file to disk that has `version: 1` and all currently-required `SidecarState` fields but no `storyLifecycle` key at all (i.e. an object shaped like a pre-ARC-1374 sidecar), call `readSidecar`, assert it returns non-null and does not throw, then call `getStoryLifecycleState` on the result and assert it returns `'unclaimed'`. This is the direct evidence for AC3.

**Verification:** `run_tests` — all new and existing tests in `sidecar.test.ts` pass.

### Step 4 — Regression test: field survives reconciliation and fresh-state creation

**Files:** `packages/server/src/__tests__/reconcile.test.ts`

**Action:** Add two tests to lock in the "no `reconcile.ts` code change needed" assumption:

- `reconcileState` preserves an existing `storyLifecycle` field unchanged when given a `SidecarState` that already has one populated (build via `makeSidecarState({ storyLifecycle: { 'ARC-9999': { state: 'claimed', updatedAt: '...' } } })`, call `reconcileState`, assert the returned state's `storyLifecycle` deep-equals the input's).
- `loadOrCreateState` on a fresh repo (no existing sidecar file) returns a state whose `storyLifecycle` is `undefined` (matching the "not initialized in `createFreshState`, callers use `getStoryLifecycleState` for the default" design — mirrors how `closedSessions`/`stepLog` are also left `undefined` by `createFreshState` rather than initialized to `{}`).

**Verification:** `run_tests` — both new tests pass; no existing `reconcile.test.ts` test changes behavior (confirms AC1 — pure addition, no regression).

### Step 5 — Full verification pass

**Files:** N/A (verification only)

**Action:** Run the full `packages/server` test suite and typecheck to confirm the additive change introduces zero regressions anywhere else in the codebase (`laneRunner.test.ts`, `checkpoints.test.ts`, `state.test.ts`, `runbooks.summary.test.ts`, etc. all reference `SidecarState`/`makeSampleState`-style fixtures and must continue to pass unmodified, since none of them set or read `storyLifecycle`).

**Verification:** `run_typecheck` on `/repos/<ARC-1374 worktree>` returns clean; `run_tests` returns all green with no test files other than `sidecar.test.ts` and `reconcile.test.ts` modified.

## Testing & Validation

- `run_typecheck` — confirms the 16-value union, the new `SidecarState.storyLifecycle` optional field, and the two new sidecar.ts helper signatures compile cleanly against every existing call site (none of which reference the new field, so none should require changes).
- `run_tests` — confirms:
  - New `sidecar.test.ts` scenarios (Step 3) prove default-fallback behavior (AC3) and update-then-persist-then-reload round-trip behavior (AC2).
  - New `reconcile.test.ts` scenarios (Step 4) prove the field is inert with respect to markdown-driven reconciliation and fresh-session bootstrap (supports AC1's "no existing behavior changes").
  - The full existing suite (`laneRunner.test.ts`, `checkpoints.test.ts`, `state.test.ts`, `stepRunner.test.ts`, `sessions.test.ts`, `runbooks.summary.test.ts`, `runbooks.detail.test.ts`) passes unmodified — direct evidence for AC1.
- `run_lint` — confirms no style/lint violations in the two edited files.

## Risks & Open Questions

- **No migration script is written or needed.** Because `storyLifecycle` is optional (`storyLifecycle?: ...`) and `readSidecar`'s only structural validation gate checks `version === 1` (unaffected by this change), every existing `.runbook-executor-state.json` file on disk today continues to parse successfully with `storyLifecycle` simply `undefined`. `getStoryLifecycleState` is the single safe-read choke point that turns that `undefined` into the documented default (`'unclaimed'`) for any future caller — no destructive rewrite of on-disk files is required or performed by this story.
- **Default value choice:** the issue's acceptance criteria offers two options in parentheses — a fixed default (`'unclaimed'`) or a value "inferred from existing flags" (e.g. cross-referencing `stepState`/`checkpointQueue` to guess a more accurate historical state). This plan takes the fixed-default option because (a) it is unambiguous and requires no heuristic that could later conflict with the real inference logic that ARC-1381 (legacy-enum rollup) will need to build carefully and deliberately, and (b) the issue's Out of Scope section explicitly defers "deriving legacy enums as rollups" to ARC-1381, and a flag-inference heuristic here would be doing part of that work prematurely and inconsistently. If a future story needs richer inference, it can layer that on top of `getStoryLifecycleState` without touching this story's contract.
- **No wiring into `laneRunner.ts`/`checkpointGit.ts`/`checkpoints.ts` in this story** — by design, per Out of Scope. This means after this story merges, `storyLifecycle` will remain empty (`undefined`, defaulting to `'unclaimed'` for every story) in real sidecar files until a later story in this epic starts calling `setStoryLifecycleState` from actual execution code. This is expected and intentional, not a gap in this story.
