# ARC-1376 Migrate planning-artifact gate onto shared PR-review sub-machine (supersedes ARC-1372) — Completion Summary

**Issue**: [issues/story-lifecycle/ARC-1376-migrate-planning-artifact-gate-onto-shared-sub-machine.md](issues/story-lifecycle/ARC-1376-migrate-planning-artifact-gate-onto-shared-sub-machine.md)
**Implementation Plan**: [implementation_plans/story-lifecycle/ARC-1376-implementation-plan.md](implementation_plans/story-lifecycle/ARC-1376-implementation-plan.md)
**Completed**: 2026-07-14
**Branch**: ARC-1376 (worktree at /repos/ARC-1376, pushed to origin/ARC-1376 in pa.aid.conductor.ts)
**Final commit hash**: 39ee2554acc28b5ee8141d7863688ee35473e679

## Acceptance Criteria Verification
| Acceptance Criteria | Status | Evidence |
|----------------------|--------|----------|
| Given a planning-artifact PR is open with unresolved comments, when resume is triggered, then the system does NOT reject with 409 and instead routes through the shared comment-addressing flow. | Passed | the 409 guard block was deleted from `packages/server/src/routes/checkpoints.ts`'s resume route. New test `checkpoints.test.ts` describe block "POST /api/checkpoints/:stepId/resume — ARC-1376 planning-artifact PR-review-loop gate" contains test "routes unresolved comments through the shared comment-addressing flow without dequeuing (does NOT reject with 409)" which asserts `res.status` is 200 (not 409) and that `runLane` is not called while `renotifyMessage` is set via the resumeArtifactPRReviewLoop 'addressing_comments' outcome. Also test "invokes executeStep/replyToThread/resolveThread once per unresolved comment via the addressComments callback" proves the shared addressPlanningArtifactComments helper is invoked. Command: `npx vitest run src/__tests__/checkpoints.test.ts` → 25/25 passed. |
| Given a planning-artifact PR is merged, when resume is triggered, then the system advances to the next phase (AGENT_EXECUTING(code) for plan artifacts). | Passed | new test "never returns 409 and advances to the next phase when resumeArtifactPRReviewLoop resolves outcome 'advance'" in checkpoints.test.ts asserts the entry is dequeued (checkpointQueue becomes empty), `setStoryLifecycleState` is called (via real sidecar.js import) to record `agent_executing`/phase `code` for plan artifacts (phase `decision` for decision artifacts, covered by the separate decision-artifactKind test), and `runLane` is called. Command: `npx vitest run src/__tests__/checkpoints.test.ts` → passed. |
| Given the old planningPRPollers background polling mechanism, when this migration lands, then it is removed (no longer needed — merge detection happens only on explicit resume). | Passed | `startPlanningPRPoller` function deleted from `packages/server/src/git/checkpointGit.ts`; `planningPRPollers` Map and its `.clear()` call deleted from `packages/server/src/runner/laneRunner.ts`; the ARC-1371 startup-recovery for-loop deleted from `packages/server/src/index.ts`. Verified via `grep -rn "startPlanningPRPoller|planningPRPollers" packages/server/src` returning zero matches outside test-mock variable names that were also removed. The `describe('startPlanningPRPoller', ...)` 3-test block was deleted from checkpointGit.test.ts. Full typecheck (`npx tsc --noEmit -p packages/server`) passes clean with zero dangling references. |
| Given decision documents previously used a status:pending/answered markdown field mechanism, when this migration lands, then decision documents also use the new PR-merge-gated sub-machine instead (via artifactKind: 'decision'), and the old markdown-field mechanism is removed. | Passed | the `resumeLaneFromPlanningPR` function (which contained the `readFileSync`-based `status: pending`/`answered` check reading `decisions/${storyKey}-approach-decision.md`) was deleted entirely from laneRunner.ts, along with its 3 corresponding tests in laneRunner.test.ts (`describe('resumeLaneFromPlanningPR (ARC-1371)', ...)`). New test "calls resumeArtifactPRReviewLoop with artifactKind 'decision' for decision-type entries, with no readFileSync/markdown-status parsing" in checkpoints.test.ts proves decision-type checkpoint entries route through `resumeArtifactPRReviewLoop` with `artifactKind: 'decision'` identically to plan artifacts. Command: `grep -rn "resumeLaneFromPlanningPR|status: answered|status: pending" packages/server/src` → only remaining match is a PR-description string literal in `createDecisionPR` (data, not gating logic — noted as a pre-existing, out-of-scope follow-up in the implementation plan's Risks section). |

## Implementation Summary
Rewired checkpoints.ts's resume route to add a new `entry.planningArtifact !== undefined` branch calling ARC-1375's `resumeArtifactPRReviewLoop`, replacing the old poller+409-guard mechanism. Extracted `addressPlanningArtifactComments` (per-comment executeStep/reply/resolve loop) out of the existing `addressPRComments` so both the generic ARC-1304 comment-thread path and the new planning-artifact path share identical per-comment logic. Removed `startPlanningPRPoller`, `planningPRPollers`, the index.ts startup-recovery loop, and the dead `resumeLaneFromPlanningPR` function (with its markdown status-parsing). No changes to `createPlanPR`/`createDecisionPR` (PR creation) — only the resume/gate side changed, per the plan's explicit scope boundary.

## Verification Steps

```sh
cd /repos/ARC-1376/packages/server
npx tsc --noEmit -p .
npx vitest run src/__tests__/checkpoints.test.ts src/__tests__/laneRunner.test.ts src/git/checkpointGit.test.ts
cd /repos/ARC-1376
npm run test --workspace=packages/server   # full suite: 429/429 passed, 22 test files
grep -rn "startPlanningPRPoller\|planningPRPollers\|resumeLaneFromPlanningPR" packages/server/src   # zero matches
```

## Tests Added/Modified
- `packages/server/src/git/checkpointGit.test.ts` — deleted the `startPlanningPRPoller` describe block (3 tests); updated the createPlanPR/createDecisionPR/isPRMerged import destructuring to drop `startPlanningPRPoller`.
- `packages/server/src/__tests__/laneRunner.test.ts` — removed `startPlanningPRPollerMock` and its `vi.mock` wiring; rewrote the two `triggerPlanningArtifactPR` tests to drop poller assertions while preserving checkpoint-entry/prUrl shape assertions; deleted the entire `describe('resumeLaneFromPlanningPR (ARC-1371)', ...)` block (3 tests) and its now-unused `parseRunbookMock`.
- `packages/server/src/__tests__/checkpoints.test.ts` — replaced the old `describe('... ARC-1371 planning-artifact PR guard', ...)` block (2 tests: 409 assertion + AC6 fallback) with a new `describe('... ARC-1376 planning-artifact PR-review-loop gate', ...)` block (8 tests) covering: advance outcome, addressing_comments outcome (no 409), per-comment callback invocation, still_awaiting_merge no-op, decision artifactKind wiring, AC6 no-prUrl fallback, and reject-falls-through-to-resume. Added `resumeArtifactPRReviewLoop` to the checkpointGit.js mock and imported real `setStoryLifecycleState`/`getStoryLifecycleState` via `vi.importActual` in the sidecar.js mock.

Test results: server package 429/429 passed across 22 test files (`npm run test --workspace=packages/server`); UI package unaffected (192/192 passed, out of scope but confirmed no regressions).

## Rollback Notes
None — this is a pure migration with no data migration needed; the old `planningPRPollers` Map was purely in-memory and never persisted. Sidecar state's `checkpointQueue[].planningArtifact` shape is unchanged.

## Next Steps
- Follow-up (noted in the implementation plan's Risks section, out of scope for this story): `createDecisionPR`'s PR description text in `checkpointGit.ts` still instructs reviewers to "edit status: answered and selected_approach: Option N... before merging" — this string is now stale since the code no longer reads that field. A fast-follow should update this PR-description text to say "merge this PR to resume" instead.
- Follow-up (also noted, out of scope, lives in the planning repo not the app repo): `decisions/_template.md` in `pa.aid.runbook-executor` still documents the old status:pending/answered workflow for human reviewers; should be updated to reflect the new merge-to-resume model.
- ARC-1378 (separate story, out of scope here): wire the new `artifactKind: 'implementation'` gate for the implementation-review checkpoint.