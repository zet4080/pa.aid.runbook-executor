issue: ARC-1374
date: 2026-07-12
status: in-review

# ARC-1374: Add canonical StoryLifecycleState enum and per-story state field — Completion Summary

**Issue:** `issues/story-lifecycle/ARC-1374-add-canonical-story-lifecycle-state-enum.md`
**Implementation Plan:** `implementation_plans/story-lifecycle/ARC-1374-implementation-plan.md`
**Completed:** 2026-07-12
**Commit:** `1ac3374` — "feat(story-lifecycle): add canonical StoryLifecycleState enum and per-story state field" (branch `ARC-1374`, pushed to origin, PR not yet opened)

## Acceptance Criteria Verification

| # | Criterion | Status | Evidence |
|---|-----------|--------|----------|
| 1 | Given the new types are added to `state/types.ts`, when existing code compiles, then no existing behavior changes (pure additive type + field, no migration of call sites yet) | Passed | `packages/server/src/state/types.ts:13-49` adds `StoryLifecycleState` (16-value string-literal union), `StoryLifecyclePhase` (`'plan'\|'code'\|'decision'`), `StoryLifecycleRecord` interface, and `DEFAULT_STORY_LIFECYCLE_STATE` constant as pure additions — no existing type or field is modified. `types.ts:156` adds `storyLifecycle?: Record<string, StoryLifecycleRecord>` as an **optional** field on `SidecarState`, following the existing additive-field pattern (`prPollIntervalMs`, `conflictingSession`, `closedSessions`, `stepLog`). `run_typecheck` returned 0 errors. Full test suite (392/392 tests, 22/22 files) passed with zero call sites in `laneRunner.ts`, `checkpointGit.ts`, or `checkpoints.ts` touched, confirming no behavior change. |
| 2 | Given a story begins executing, when its lifecycle state changes, then the new per-story state field in the sidecar is updated and persisted across restarts | Partially met (by design) | `packages/server/src/state/sidecar.ts` adds `setStoryLifecycleState(state, storyKey, newState, phase?)`, which returns a new `SidecarState` with the story's entry shallow-merged into `storyLifecycle` (no clobbering of other keys) and an ISO `updatedAt` timestamp. The persistence and restart-survival mechanism is proven correct end-to-end via the round-trip test at `packages/server/src/__tests__/sidecar.test.ts:220-228` (`setStoryLifecycleState` → `writeSidecar` → fresh `readSidecar` call simulating a restart → deep-equal assertion on the reloaded entry). **However**, per the issue's own Out of Scope section ("migrating any actual call sites in laneRunner.ts/checkpointGit.ts/checkpoints.ts to use the new enum … that's later stories"), no real execution code path calls `setStoryLifecycleState` yet — the field remains unpopulated in real sidecar files until a later ARC-1373 story wires it in. This story delivers the persistence *mechanism* only, as scoped by the approved implementation plan. |
| 3 | Given the sidecar schema changes, when older sidecar.json files without the new field are loaded, then they load without error (backward compatible default state) | Passed | `getStoryLifecycleState(state, storyKey)` in `sidecar.ts` returns `state.storyLifecycle?.[storyKey]?.state ?? DEFAULT_STORY_LIFECYCLE_STATE` (`'unclaimed'`), acting as the single safe-read choke point for both an entirely absent `storyLifecycle` map and an absent individual story key. Regression test `packages/server/src/__tests__/sidecar.test.ts:230-249` writes a raw JSON fixture to disk shaped like a pre-ARC-1374 sidecar (`version: 1`, all pre-existing required fields, no `storyLifecycle` key at all), calls `readSidecar`, asserts it returns non-null without throwing, then calls `getStoryLifecycleState` on the result and asserts it returns `'unclaimed'`. |

## Implementation Summary

Added the foundational types and persistence plumbing for the story lifecycle state machine (ARC-1373), with zero behavior change to existing call sites:

- **`packages/server/src/state/types.ts`** — `StoryLifecycleState` (16-value string-literal union covering the full lifecycle from `unclaimed` through `archived`), `StoryLifecyclePhase` (`'plan'|'code'|'decision'`), `StoryLifecycleRecord` interface (`{ state, phase?, updatedAt }`), `DEFAULT_STORY_LIFECYCLE_STATE` constant (`'unclaimed'`), and a new optional `storyLifecycle?: Record<string, StoryLifecycleRecord>` field on `SidecarState`, keyed by bare story key (e.g. `"ARC-1374"`) to match the existing `storyKey` convention used elsewhere in the codebase.
- **`packages/server/src/state/sidecar.ts`** — two new pure helper functions: `getStoryLifecycleState` (safe read with default fallback) and `setStoryLifecycleState` (immutable update, shallow-merges the story's entry without touching other keys or mutating the input object). No changes to `readSidecar`/`writeSidecar` were needed since they already serialize/deserialize the whole `SidecarState` object generically.
- **`packages/server/src/__tests__/sidecar.test.ts`** — 9 new tests covering: default fallback when `storyLifecycle` is entirely absent, default fallback when the specific story key is absent, correct read of a stored value, immutability of `setStoryLifecycleState` (input reference unchanged), no-clobber across multiple story keys, overwrite-same-key behavior, round-trip persistence via `writeSidecar`+`readSidecar`, and the pre-ARC-1374 backward-compat fixture.
- **`packages/server/src/__tests__/reconcile.test.ts`** — 2 new regression tests locking in that `reconcileState` passes an existing `storyLifecycle` field through unchanged, and that `loadOrCreateState` on a fresh repo leaves `storyLifecycle` as `undefined` (consistent with how `closedSessions`/`stepLog` are also left uninitialized rather than defaulted to `{}`).

No I/O layer, route (`routes/state.ts`), or reconciliation logic changes were required — the additive/optional design and existing object-spread patterns in `reconcile.ts` absorb the new field automatically, which the new regression tests confirm.

## Verification Steps

```bash
# TypeScript check
cd /repos/ARC-1374 && npx tsc --noEmit --project packages/server
# → 0 errors

# Full server test suite
cd /repos/ARC-1374 && npm test --workspace=packages/server
# → 392/392 tests passed, 22/22 test files passed

# Lint
# → no lint tool configured in this repo (N/A)
```

## Tests Added/Modified

- `packages/server/src/__tests__/sidecar.test.ts` — 9 new tests: `getStoryLifecycleState` default fallback (absent field, absent key), correct value read, `setStoryLifecycleState` immutability, no-clobber across keys, overwrite-same-key, round-trip persistence, pre-ARC-1374 backward-compat fixture.
- `packages/server/src/__tests__/reconcile.test.ts` — 2 new regression tests: `reconcileState` passthrough of existing `storyLifecycle`, `loadOrCreateState` leaves `storyLifecycle` undefined on a fresh session.

**Test results:** 392/392 passing (22/22 files). TypeScript: 0 errors. Lint: no lint tool configured in this repo.

## Local Code Review

Verdict: **APPROVE WITH SUGGESTIONS** (1 review iteration). 1 non-blocking nit noted at `sidecar.ts:127-131` regarding optional cleanup of the conditional spread used for the `phase` field in `setStoryLifecycleState` — no blockers or majors identified.

## Rollback Notes

None required. The change is purely additive: a new optional field and two new pure helper functions with no existing call sites modified. Reverting the commit removes the types/helpers/tests with no migration or data cleanup needed, since no real sidecar file has ever been written with a populated `storyLifecycle` field by this story.

## Next Steps

- A later ARC-1373 story must wire `setStoryLifecycleState`/`getStoryLifecycleState` into the real execution call sites (`laneRunner.ts`, `checkpointGit.ts`, `checkpoints.ts`) so the field is actually populated during live story execution — this is the remaining half of AC2, explicitly deferred per this issue's Out of Scope section.
- ARC-1381 (deriving legacy enums as rollups from the new canonical state) depends on this story's types and is explicitly out of scope here.
- PR for branch `ARC-1374` has not yet been opened — to be opened manually by the human supervisor outside of the agent workflow.
