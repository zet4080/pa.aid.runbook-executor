issue: ARC-1367
date: 2026-07-07
status: completed

# ARC-1367: Conductor: auto-claim story in runbook before dispatching agent — Completion Summary

**Issue:** `done/issues/parallel-lane-execution/ARC-1367-conductor-auto-claim-story.md`
**Implementation Plan:** `done/implementation_plans/parallel-lane-execution/ARC-1367-implementation-plan.md`
**Completed:** 2026-07-07
**Merge commit:** ba417cb072f66fb3eedba2e5028a74ff67032b3b (Merged in ARC-1367 — pull request #4)

## Acceptance Criteria Verification

| # | Criterion | Status | Evidence |
|---|-----------|--------|----------|
| 1 | Given conductor identifies next unclaimed story, when about to dispatch agent, then `🔒 Claimed:` checkbox set to `[x]` with ISO-8601 timestamp and lane name before agent process starts | Passed | `packages/server/src/runbook/markdownWriter.ts` — `markClaimedSubStepInMarkdown(runbookPath, storyKey, lane, timestamp)` locates the story's top-level line, scans indented sub-step block for first unchecked Claimed line, and replaces `- [ ]` with `- [x] 🔒 Claimed: {lane} / {timestamp}` using atomic write. `packages/server/src/runner/laneRunner.ts` — `markClaimedSubStepInMarkdown` called synchronously before `executeStep()`. Test: `laneRunner.test.ts` — "calls markClaimedSubStepInMarkdown before executeStep is called" verifies ordering. |
| 2 | Given claim written to runbook markdown, when write completes, then change committed and pushed to `main` in planning repo | Passed | `packages/server/src/git/claimGit.ts` (new) — `commitAndPushClaim(repoRoot, runbookPath, lane, storyKey)` runs `git add`, `git commit`, `git push`. Push failure caught and logged as warning (non-blocking). Called fire-and-forget via `void commitAndPushClaim(...).catch(...)` in `laneRunner.ts`. Tests: `claimGit.test.ts` — "runs git add, commit, and push in sequence" and "includes the correct lane and storyKey in the commit message". |
| 3 | Given story already claimed (conductor resuming after crash), when lane runner encounters story again, then claim step skipped without error (idempotent) | Passed | `markdownWriter.ts` — `markClaimedSubStepInMarkdown` checks if matching line already has `- [x]` and returns immediately without writing. Test: `markdownWriter.test.ts` — "is idempotent — does not modify a line already checked". |
| 4 | Given auto-claim implemented, when agent session starts, then `start-execution-session` skill no longer instructs agent to call `runbook_claim_story` as ceremony step | Passed | `/home/zimmermann/.config/opencode/skills/start-execution-session/SKILL.md` — removed "Claim the Story" subsection. Updated Common Mistakes table: "Starting a story without claiming it first" now reads "The conductor auto-claims before dispatch (ARC-1367)." |
| 5 | Unit test: auto-claim fires before first agent prompt is built | Passed | `packages/server/src/__tests__/laneRunner.test.ts` — describe block "runLane — auto-claim before agent dispatch (ARC-1367)" with 7 tests covering: ordering, correct args, git commit, skipping pre-checked steps, skipping checkpoint steps, non-blocking push failure, and multi-step lane coverage. |

## Implementation Summary

Implemented conductor-side automatic story claiming in the runbook before agent dispatch. The `laneRunner` now calls `markClaimedSubStepInMarkdown` synchronously before dispatching any agent step, then fires off `commitAndPushClaim` asynchronously (non-blocking) to commit the claim to the planning repo. This eliminates the need for agents to call `runbook_claim_story` as a manual ceremony step.

Key components:
- `markdownWriter.ts`: Added `markClaimedSubStepInMarkdown` — finds the `🔒 Claimed:` sub-item under the story key in runbook markdown and checks it off with lane + timestamp
- `claimGit.ts`: New module with `commitAndPushClaim` — runs `git add`, `git commit -m "chore({lane}): claim {KEY}"`, `git push` against the planning repo
- `laneRunner.ts`: Integrated auto-claim call at agent step dispatch, before `executeStep()`

## Verification Steps

```bash
# Run server tests
cd /repos/pa.aid.conductor.ts && npm test --workspace=packages/server

# Run TypeScript check
cd /repos/pa.aid.conductor.ts && npx tsc --noEmit
```

## Tests Added/Modified

- `packages/server/src/__tests__/laneRunner.test.ts` — added 7 auto-claim tests in "runLane — auto-claim before agent dispatch (ARC-1367)" describe block
- `packages/server/src/__tests__/markdownWriter.test.ts` — added claimed sub-step tests (idempotency, write, no-match guard)
- `packages/server/src/git/claimGit.test.ts` — new file, 5 unit tests for `commitAndPushClaim`

**Test results:** 312 passing, 2 failing (both pre-existing in `checkpointGit.test.ts`, unrelated to ARC-1367). TypeScript: 0 errors.

## Rollback Notes

None. The auto-claim is additive — if reverted, agents would need to call `runbook_claim_story` manually again.

## Next Steps

- ARC-1368 — Conductor: auto-provision feature git worktree for HIGH stories (depends on this story)