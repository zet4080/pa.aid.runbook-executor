# ARC-1380: Fix archive-guard silent failure — route to escalation checkpoint — Completion Summary

**Issue:** `issues/story-lifecycle/ARC-1380-fix-archive-guard-silent-failure.md`
**Implementation Plan:** `implementation_plans/story-lifecycle/ARC-1380-implementation-plan.md`
**Branch:** `ARC-1380` (pa.aid.conductor.ts, off `main`, not yet merged)
**Completed:** 2026-07-12
**Duration:** ~45min

## Acceptance Criteria Verification

| # | Criterion | Status | Evidence |
|---|-----------|--------|----------|
| 1 | Given `archiveStoryArtifacts`'s guard check fails (missing required file), when the check fails, then the story transitions to `awaiting_escalation_checkpoint` instead of silently returning. | Passed | `archiveStoryArtifacts` (`packages/server/src/git/planningArchiver.ts`) now returns an `ArchiveOutcome` discriminated union; both file-presence guards return `{ status: 'guard-failed', reason: 'summary-missing' \| 'issue-missing' }` instead of bare `return`. `laneRunner.ts`'s end-of-lane archive block inspects the result and, on `guard-failed`, calls `setStoryLifecycleState(current, arcKey, 'awaiting_escalation_checkpoint')`, pushes a synthetic `CheckpointQueueEntry` (`stepId: \`${arcKey}__archive-guard\``, `checkpointLevel: 'high'`), persists via `writeSidecar`, and sets the lane to `'paused'` (overriding the earlier optimistic `'done'`). Verified by 3 new tests in `laneRunner.test.ts` (`describe('runLane — archive-guard failure routes to escalation checkpoint (ARC-1380)')`): checkpoint entry with no `prUrl`/`planningArtifact` and `lastAgentMessage` containing `'completion summary'` (summary-missing case) or `'issue file'` (issue-missing case); `state.storyLifecycle?.['ARC-1380']?.state === 'awaiting_escalation_checkpoint'`. Regression test confirms `'already-archived'` and `'error'` outcomes remain non-escalating (lane stays `'done'`, no checkpoint pushed) — the deliberate scope boundary documented in the plan's Risks section. `npx vitest run src/__tests__/laneRunner.test.ts` → 95/95 passed. |
| 2 | Given the story is in `awaiting_escalation_checkpoint` due to an archive-guard failure, when the supervisor resumes after fixing the issue, then the archiving operation is re-attempted. | Passed | The new checkpoint entry deliberately carries no `prUrl` and no `planningArtifact`, so the existing generic "no prUrl" resume branch in `routes/checkpoints.ts` (unmodified, confirmed by `checkpoints.test.ts` passing 20/20 unmodified) dequeues the entry and re-parses the runbook from disk before calling `runLane` again. Because every prior step is already `checked: true`, `runLane`'s per-step loop (`packages/server/src/runner/laneRunner.ts:566`, `if (step.checked === true)`) skips straight to the same end-of-lane archive block, re-invoking `archiveStoryArtifacts`. Verified by the `'re-attempt / resume simulation (AC2)'` test in `laneRunner.test.ts`: first `runLane` call returns `guard-failed` → lane paused, one checkpoint entry; second `runLane` call (same steps, `archiveStoryArtifactsMock` reconfigured to return `{ status: 'archived' }`) results in `archiveStoryArtifactsMock` called exactly twice, lane ends `'done'`, and no duplicate checkpoint entry is added. `npx vitest run src/__tests__/laneRunner.test.ts` → 95/95 passed. |

## Implementation Summary

`archiveStoryArtifacts` (`packages/server/src/git/planningArchiver.ts`) was widened from a `void`-returning function to one returning a new exported `ArchiveOutcome` discriminated union (`'archived' | 'already-archived' | 'guard-failed' | 'error'`), with each existing early-return point updated to return the matching outcome. No guard condition or `console.warn` message text was changed — all ten pre-existing tests in `planningArchiver.test.ts` continue to pass unmodified, satisfying the issue's "Out of Scope: changing what the guards actually check for."

`laneRunner.ts`'s end-of-lane ARC-1370 archive block now captures the `ArchiveOutcome` and, on `status === 'guard-failed'`, routes to escalation: it builds a human-readable reason string, pushes a synthetic high-level `CheckpointQueueEntry` with a composite `stepId` (`${arcKey}__archive-guard`, following the existing `${storyKey}__<suffix>` naming convention), calls the ARC-1374 `setStoryLifecycleState` helper to transition the story to `'awaiting_escalation_checkpoint'`, persists state via `writeSidecar`, and overrides the lane status to `'paused'`. This is the first production call site to invoke `setStoryLifecycleState`.

No changes were made to `routes/checkpoints.ts` or the UI (`ResumeCard.tsx`) — the checkpoint entry deliberately omits `prUrl` and `planningArtifact` so the existing generic resume path applies unmodified, exactly as planned.

## Verification Steps

```bash
cd /repos/ARC-1380/packages/server
npx vitest run src/__tests__/planningArchiver.test.ts   # 20/20 passed
npx vitest run src/__tests__/laneRunner.test.ts          # 95/95 passed
npm test                                                  # 401/401 passed (full package suite, incl. checkpoints.test.ts unmodified)
npx tsc --noEmit                                          # clean, no type errors
```

## Tests Added/Modified

- `packages/server/src/__tests__/planningArchiver.test.ts` — 4 new tests asserting the four `ArchiveOutcome` variants (`guard-failed`/summary-missing, `guard-failed`/issue-missing, `already-archived`, `archived`). 10 pre-existing tests unmodified.
- `packages/server/src/__tests__/laneRunner.test.ts` — mock default for `archiveStoryArtifactsMock` updated to `.mockReturnValue({ status: 'archived' })`; `../state/sidecar.js` mock updated to expose the real (pure) `setStoryLifecycleState` via `vi.importActual`. New `describe('runLane — archive-guard failure routes to escalation checkpoint (ARC-1380)')` block with 5 tests: checkpoint routing for both guard-failure reasons, `storyLifecycle` state transition, non-escalating `'already-archived'`/`'error'` regression coverage, and the resume/re-attempt simulation (AC2).

## Rollback Notes

None. Change is additive to the return type (widened from `void`); the only production call site (`laneRunner.ts`) was updated in the same commit. No data migration required — `SidecarState.storyLifecycle` was already added by ARC-1374.

## Next Steps

None. Two deliberate scope boundaries are documented in the implementation plan's Risks & Open Questions and left for a future story if ever needed: (1) the outer-catch `'error'` outcome (infrastructure/IO failures) is not escalated by this story; (2) the idempotent `'already-archived'` skip is not escalated (correct, intentional no-op behavior).
