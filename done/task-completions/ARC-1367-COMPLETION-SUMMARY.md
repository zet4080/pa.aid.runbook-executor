```markdown
---
issue: ARC-1367
date: 2026-07-07
status: completed
---

# ARC-1367: Conductor: auto-claim story in runbook before dispatching agent

## Goal
This issue implemented automatic story claiming in the runbook before dispatching an agent, ensuring a consistent and error-free process.

## Acceptance Criteria
| Criterion | Evidence |
|-----------|-----------|
| AC1: Claim checkbox set before agent dispatch | - **File:** `packages/server/src/runbook/markdownWriter.ts` — new function `markClaimedSubStepInMarkdown(runbookPath, storyKey, lane, timestamp)` that locates the story's top-level line by storyKey, scans the indented sub-step block for the first unchecked Claimed line, and replaces `- [ ]` with `- [x] 🔒 Claimed: {lane} / {timestamp}` using atomic write.
- **File:** `packages/server/src/runner/laneRunner.ts` — in `runLane()`, within the `if (step.type === 'agent')` block, `markClaimedSubStepInMarkdown` is called synchronously before `laneActiveStep.set()` and `executeStep()`.
- **Test:** `laneRunner.test.ts` — "calls markClaimedSubStepInMarkdown before executeStep is called" verifies ordering by recording call order in an array and asserting `claimIdx < execIdx`.
| AC2: Change committed and pushed to main | - **File:** `packages/server/src/git/claimGit.ts` (new) — `commitAndPushClaim(repoRoot, runbookPath, lane, storyKey)` runs `git -C {repoRoot} add {runbookPath}`, `git -C {repoRoot} commit -m "chore({lane}): claim {storyKey}"`, `git -C {repoRoot} push`. Push failure is caught and logged as warning (non-blocking).
- **File:** `packages/server/src/runner/laneRunner.ts` — `commitAndPushClaim` called fire-and-forget via `void commitAndPushClaim(...).catch(...)` immediately after `markClaimedSubStepInMarkdown`.
- **Test:** `claimGit.test.ts` — "runs git add, commit, and push in sequence" and "includes the correct lane and storyKey in the commit message".
- **Test:** `laneRunner.test.ts` — "calls commitAndPushClaim with repoRoot, runbookPath, lane, and storyKey".
| AC3: Idempotent — no-op if already claimed | - **File:** `packages/server/src/runbook/markdownWriter.ts` — `markClaimedSubStepInMarkdown` checks if the matching line already has `- [x]` and returns immediately without writing.
- **Test:** `markdownWriter.test.ts` — "is idempotent — does not modify a line already checked" verifies file is unchanged when Claimed line already checked.
| AC4: start-execution-session skill no longer instructs manual claim | - **File:** `/home/zimmermann/.config/opencode/skills/start-execution-session/SKILL.md` — Removed the "Claim the Story" subsection (which instructed calling `runbook_claim_story`). Updated Common Mistakes table: "Starting a story without claiming it first" now reads "The conductor auto-claims before dispatch (ARC-1367).".
| AC5: Unit test — auto-claim fires before first agent prompt is built | - **File:** `packages/server/src/__tests__/laneRunner.test.ts` — describe block "runLane — auto-claim before agent dispatch (ARC-1367)" with 6 tests:
  - "calls markClaimedSubStepInMarkdown before executeStep is called"
  - "calls markClaimedSubStepInMarkdown with correct runbookPath, storyKey, lane, and timestamp"
  - "calls commitAndPushClaim with repoRoot, runbookPath, lane, and storyKey"
  - "does NOT call markClaimedSubStepInMarkdown for pre-checked (skipped) steps"
  - "does NOT call markClaimedSubStepInMarkdown for checkpoint-type steps"
  - "agent dispatch proceeds even when commitAndPushClaim rejects (push failure is non-blocking)"
  - "auto-claim fires for each agent step in a multi-step lane"
|
## Test Results
- Server tests: 312 passing, 2 failing (both pre-existing in `checkpointGit.test.ts`, unrelated to ARC-1367)
- New tests added: 20 (6 laneRunner auto-claim tests + 10 markdownWriter claimed tests + 5 claimGit tests minus 1 pre-existing)
- TypeScript: 0 errors (verified with `tsc --noEmit`)

## Files Modified
- `packages/server/src/runbook/markdownWriter.ts` — added `markClaimedSubStepInMarkdown`
- `packages/server/src/git/claimGit.ts` — new file, `commitAndPushClaim`
- `packages/server/src/runner/laneRunner.ts` — integrated auto-claim before dispatch
- `packages/server/src/__tests__/laneRunner.test.ts` — added auto-claim test suite + mocks
- `packages/server/src/__tests__/markdownWriter.test.ts` — added claimed sub-step tests
- `packages/server/src/git/claimGit.test.ts` — new file, unit tests for `commitAndPushClaim`
- `/home/zimmermann/.config/opencode/skills/start-execution-session/SKILL.md` — removed manual claim ceremony
```