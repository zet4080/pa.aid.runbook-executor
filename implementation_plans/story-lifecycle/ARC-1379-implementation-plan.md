# ARC-1379 Fix worktree-provisioning silent failure — route to escalation checkpoint - Implementation Plan

**Issue:** `issues/story-lifecycle/ARC-1379-fix-worktree-provisioning-silent-failure.md`
**Completion Summary:** `task-completions/ARC-1379-COMPLETION-SUMMARY.md` (TBD)
**Approach:** Single approach — pattern established. `runLane` already has two established patterns for converting a thrown/caught failure into a paused lane with a `CheckpointQueueEntry` pushed to `checkpointQueue`: the retries-exhausted path (`checkpointLevel: 'high'`, no PR) and the `triggerPlanningArtifactPR` catch-block path (`checkpointLevel: 'high'`, no PR, used when a git-level throw must not leave the lane stuck). This story reuses that exact push-entry-and-pause shape for the `provisionWorktree` throw site, plus records `awaiting_escalation_checkpoint` via the already-merged (ARC-1374) `setStoryLifecycleState` helper. No competing designs exist; Phase 1 (Approach) is skipped per the write-implementation-plan skill's rule that Phase 1 may be skipped when exploration reveals a clear existing pattern to follow.
**Owner:** build agent
**Date:** 2026-07-12

## Scope & Alignment

This plan covers exactly the two In-Scope items from the issue:

1. Wrap the `provisionWorktree(storyKey)` call in `packages/server/src/runner/laneRunner.ts` (`runLane`, inside the `isHighStory(step)` branch) in a try/catch.
2. On catch, route the failure into the escalation-checkpoint queue: push a `CheckpointQueueEntry` (`checkpointLevel: 'high'`) onto `checkpointQueue`, persist state via `writeSidecar`, set the lane status to `'paused'`, and (via ARC-1374's `setStoryLifecycleState`) record the story's canonical lifecycle state as `'awaiting_escalation_checkpoint'`. The function then returns without executing the agent step.

Explicitly NOT covered (per issue's Out of Scope): `worktreeGit.ts`'s own provisioning logic (idempotency checks, `execSync` calls, branch-exists detection) is untouched — only the caller's error handling in `laneRunner.ts` changes.

Acceptance criteria mapping:
- **AC1** (worktree provisioning throws → story transitions to `awaiting_escalation_checkpoint` instead of failing silently) → Step 1 (try/catch + checkpoint push) + Step 2 (`setStoryLifecycleState` call) + Step 4 (unit tests asserting both the checkpoint entry and the lifecycle-state write).
- **AC2** (supervisor resumes after fixing the issue → the exact same worktree-provisioning operation is re-attempted, uniform resume behavior, no special-case routing) → Step 3 (no changes to `checkpoints.ts`'s resume route or `resumeLane`/`runLane`'s step-skip logic: the failed step's `checked` field is still `false` in the markdown, so re-parsing and re-invoking `runLane` naturally re-enters the same `isHighStory(step)` branch and re-calls `provisionWorktree(storyKey)` — this is the same "uniform resume" mechanism the retries-exhausted and planning-artifact-PR-catch paths already rely on) + Step 4 (regression test proving the resume path re-attempts provisioning with no special-cased branch).

## Assumptions & Dependencies

- **Dependency ARC-1374 is satisfied**: `StoryLifecycleState` (including the `'awaiting_escalation_checkpoint'` member), `DEFAULT_STORY_LIFECYCLE_STATE`, `StoryLifecycleRecord`, and the pure helpers `getStoryLifecycleState`/`setStoryLifecycleState` are already merged into `pa.aid.conductor.ts` `main` (`packages/server/src/state/types.ts` lines 13–29, 38, 43–49; `packages/server/src/state/sidecar.ts` lines 113–139). This plan imports and calls `setStoryLifecycleState` — it does not redefine any lifecycle types.
- `provisionWorktree(storyKey: string, appRepoRoot?: string): void` in `packages/server/src/git/worktreeGit.ts` throws a plain `Error` (via unwrapped `execSync` calls) on any git failure other than "worktree already exists" (which returns early, no throw). This is unchanged — confirmed by reading the current implementation.
- The `storyKey` used for `provisionWorktree` in `runLane` is already computed via `stripNamespace(step.id)` (line ~575) before the call — the same `storyKey` value is reused for `setStoryLifecycleState(state, storyKey, 'awaiting_escalation_checkpoint')`, so no new key-derivation logic is needed.
- No new field is added to `CheckpointQueueEntry` — the existing shape (`stepId`, `runbookPath`, `checkpointLevel`, `hitAt`, optionally `lastAgentMessage`) is sufficient, matching the retries-exhausted and `triggerPlanningArtifactPR`-catch push patterns exactly. This story does not add a `worktreeFailure`-specific field to the queue entry because the issue's AC2 explicitly asks for "uniform resume behavior, no special-case routing" — a generic high-level checkpoint entry with `stepId: step.id` is what every other resume-and-retry step in `runLane` already produces, and `checkpoints.ts`'s resume route already handles a plain (non-`planningArtifact`, non-`prUrl`) entry by dequeuing and calling `runLane` again from disk state.
- The catch block captures the error message for logging (`console.error`) so failures are diagnosable in server logs, following the existing style used in `triggerPlanningArtifactPR`'s catch block (`console.error(...String(err))`) — this is not user-visible state, just operational logging.
- `laneActiveStep`/`notifyLaneStep`/`endLaneStep` bookkeeping has not yet been entered at the point `provisionWorktree` is called (it is called before `laneActiveStep.set(runbookPath, step.id)` and `notifyLaneStep` later in the function) — so no `endLaneStep`/`laneActiveStep.delete` cleanup is needed in the new catch block; nothing was set yet.
- The pre-existing unit test `'pauses the lane and pushes a checkpoint entry when provisionWorktree throws'` (`packages/server/src/__tests__/laneRunner.test.ts`, describe block `runLane — auto-provision worktree for HIGH stories (ARC-1368)`) currently asserts `runLane` **rejects** with the thrown error (`await expect(runLane(...)).rejects.toThrow(...)`) — this assertion is the exact silent-failure behavior this story is fixing (an uncaught throw propagating out of `runLane`, which is a fire-and-forget async function per its own doc comment: "must not throw unhandled rejections"). This test must be rewritten to assert the new caught-and-paused behavior instead of deleted, since it is the direct regression coverage for this story's core AC.

## Implementation Steps

### Step 1 — Wrap `provisionWorktree` call in try/catch and push an escalation checkpoint on failure

**Files:** `packages/server/src/runner/laneRunner.ts`

**Action:**
In `runLane`, inside the `if (step.type === 'agent')` block, replace the current unguarded call:

```
worktreePath = `/repos/${storyKey}`;
provisionWorktree(storyKey); // throws on failure → caught as step failure → lane paused
```

with a try/catch that, on failure:
1. Logs the error via `console.error` with a `[ARC-1379]` prefix and the `storyKey`, mirroring the `[ARC-1371]`/`[ARC-1302]` prefix convention already used elsewhere in this file.
2. Calls `setStoryLifecycleState(getState(), storyKey, 'awaiting_escalation_checkpoint')`, then `setState(...)` and `writeSidecar(current.repoRoot, ...)` with the result — following the exact `getState()` → mutate → `setState()` → `writeSidecar()` sequence already used at every other checkpoint-push site in this file (e.g. the retries-exhausted block).
3. Pushes a `CheckpointQueueEntry` with `stepId: step.id`, `runbookPath`, `checkpointLevel: 'high'`, `hitAt: new Date().toISOString()` onto `checkpointQueue` (same shape as the retries-exhausted entry at the bottom of the retry loop), persisting via the same `setState`/`writeSidecar` pair (can be combined with step 2's state update into a single `setState`/`writeSidecar` call to avoid two separate persisted writes).
4. Sets `laneStatusMap.set(runbookPath, 'paused')`.
5. Returns from `runLane` (`return;`) — matching the `return;` used at every other pause site in this function, so the function resolves normally instead of rejecting.

The `worktreePath` variable and the subsequent `buildPromptFromStep(step, runbookPath, worktreePath)` call, `executeStep` dispatch, etc. must NOT execute when this catch fires — the `return` accomplishes this since it exits the whole `for` loop iteration and the function.

**Verification:** `run_typecheck` passes (no type errors from the new `CheckpointQueueEntry`/`SidecarState` usage). Manually trace the control flow: confirm no code path after the catch's `return` can still run `provisionWorktree`'s caller logic for that step.

### Step 2 — Import `setStoryLifecycleState`

**Files:** `packages/server/src/runner/laneRunner.ts`

**Action:** Add `setStoryLifecycleState` to the existing `import { writeSidecar } from '../state/sidecar.js';` import statement (near the top of the file, alongside the other `../state/*` imports), producing `import { writeSidecar, setStoryLifecycleState } from '../state/sidecar.js';`. No new import line is needed — this extends the existing one.

**Verification:** `run_typecheck` passes; no unused-import lint warning (the function is used in Step 1's catch block).

### Step 3 — Confirm uniform resume behavior requires no changes to the resume path

**Files:** N/A (verification/analysis only — no code changes in this step)

**Action:** Confirm by inspection that the existing resume mechanism in `packages/server/src/routes/checkpoints.ts` (`POST /api/checkpoints/:stepId/resume`) already satisfies AC2 with zero modification:
- The new checkpoint entry pushed in Step 1 has no `prUrl` and no `planningArtifact` field, so it falls through `checkpoints.ts`'s existing "No prUrl" branch (line ~172 onward): the entry is dequeued from `checkpointQueue`, state is persisted, and `parseRunbook(entry.runbookPath)` + `runLane(entry.runbookPath, steps)` are invoked fire-and-forget — re-parsing from disk.
- Because the failed step was never marked `checked` in the markdown (the catch in Step 1 returns before reaching `markStepCheckedInMarkdown`), the re-parsed `step.checked` is still `false`. `runLane` re-enters its `for` loop, reaches the same step, re-evaluates `isHighStory(step)` (still true — the checkpoint sub-step is still unchecked), and calls `provisionWorktree(storyKey)` again with the same `storyKey` — i.e., "the exact same worktree-provisioning operation is re-attempted" with no special-case routing, satisfying AC2 verbatim.
- No new resume-route branch, no new `stepId` suffix convention (like `__checkpoint` or `__plan-pr`), and no new dispatch logic is needed because this checkpoint entry deliberately reuses the plain `step.id` shape that `checkpoints.ts` already treats as "just re-run the lane."

**Verification:** Code-read confirmation only (documented above) plus the Step 4 test `'resume after worktree-provisioning failure re-attempts provisionWorktree with the same storyKey'`, which exercises this exact flow end-to-end at the `runLane` level (a full `checkpoints.ts` route-level integration test is unnecessary since the route's generic "no prUrl" resume path is unmodified and already covered by its own existing test suite).

### Step 4 — Update and add unit tests in `laneRunner.test.ts`

**Files:** `packages/server/src/__tests__/laneRunner.test.ts`

**Action:**
1. **Rewrite** the existing test `'pauses the lane and pushes a checkpoint entry when provisionWorktree throws'` (in `describe('runLane — auto-provision worktree for HIGH stories (ARC-1368)')`) to match the new non-throwing, paused-and-queued behavior instead of the old `rejects.toThrow(...)` assertion. New assertions:
   - `await runLane(RUNBOOK_PATH, [step])` resolves (does not reject).
   - `executeStepMock` was not called (worktree failure occurred before dispatch).
   - `getLaneState(RUNBOOK_PATH)` is `'paused'`.
   - `getState().checkpointQueue` has length 1, with `stepId` equal to the step's `id`, `checkpointLevel: 'high'`, and a string `hitAt`.
2. **Add** a new test in the same describe block: `'records awaiting_escalation_checkpoint lifecycle state when provisionWorktree throws'` — asserts (via importing `getStoryLifecycleState` from `../state/sidecar.js` in the test, applied to `getState()`) that the story's lifecycle state is `'awaiting_escalation_checkpoint'` after the throw, using the same `storyKey` (`'ARC-1368'`) the existing `makeHighStoryStep` helper already derives via `stripNamespace`.
3. **Add** a new test: `'resume after worktree-provisioning failure re-attempts provisionWorktree with the same storyKey'` — first call `runLane` with `provisionWorktreeMock` throwing once (mock a first-call throw, e.g. via `mockImplementationOnce` throwing then a plain no-op for the second call), confirm the paused state and queued entry from point 1, then re-invoke `runLane(RUNBOOK_PATH, [step])` directly (simulating the resume route's re-dispatch with the same unchanged `step` object, whose `checked` is still `false`) and assert `provisionWorktreeMock` was called a second time with `'ARC-1368'` and that `executeStepMock` is now called (provisioning succeeded on retry) — proving no special-case routing exists and the same operation runs again.
4. Confirm no other existing test in the `ARC-1368` describe block or elsewhere in the file asserts on the old throwing behavior (only the one test identified above does, per the earlier `grep` of `rejects.toThrow`).

**Verification:** `run_tests` — the rewritten test and both new tests pass; no other existing test in `laneRunner.test.ts` regresses (in particular the three `does NOT call provisionWorktree...` tests in the same describe block remain unaffected since they don't exercise the throw path).

### Step 5 — Full verification pass

**Files:** N/A (verification only)

**Action:** Run `run_typecheck`, `run_lint`, and `run_tests` against `packages/server` in the `ARC-1379` worktree to confirm the two-file change (`laneRunner.ts`, `laneRunner.test.ts`) introduces zero regressions in the broader suite (`checkpoints.test.ts`, `sidecar.test.ts`, `reconcile.test.ts`, etc., none of which reference the modified catch block).

**Verification:** All three checks return clean/green.

## Testing & Validation

- `run_typecheck` — confirms the new `setStoryLifecycleState` import and its call signature (`(state: SidecarState, storyKey: string, newState: StoryLifecycleState) => SidecarState`) compile cleanly against the `SidecarState` returned by `getState()`.
- `run_tests` — confirms:
  - The rewritten "pauses the lane and pushes a checkpoint entry" test proves AC1 (no more unhandled rejection; lane transitions to `'paused'` with a queued `CheckpointQueueEntry` instead of failing silently).
  - The new "records awaiting_escalation_checkpoint lifecycle state" test proves AC1's specific state-machine requirement (transitions to `awaiting_escalation_checkpoint`, not just a generic pause).
  - The new "resume ... re-attempts provisionWorktree" test proves AC2 (uniform resume, same operation re-attempted, no special-case routing).
  - The full existing `laneRunner.test.ts` suite (all `describe` blocks — ARC-1368, ARC-1367, ARC-1369, ARC-1370, ARC-1371, ARC-1306 retry logic, crash recovery) passes unmodified aside from the one rewritten test, confirming no regression to unrelated lane-runner behavior.
- `run_lint` — confirms no style/lint violations in the two edited files.

## Risks & Open Questions

- **No `checkpoints.ts` changes are needed or made** — confirmed by Step 3's control-flow trace: the new checkpoint entry is deliberately shaped identically to the existing retries-exhausted entry (no `prUrl`, no `planningArtifact`), so it flows through `checkpoints.ts`'s pre-existing generic "dequeue and re-run `runLane` from disk" branch with zero new code. If a future story (e.g. ARC-1380, which has an identical `awaiting_escalation_checkpoint` requirement for archive-guard failures) needs a distinguishing marker on the queue entry (e.g. a `failureReason` field) to show a different message in the UI, that is a separate concern not required by this story's AC — this story only requires the *state transition* and *uniform resume*, not a differentiated UI message. Noting this so ARC-1380 authors are aware the same generic shape is being reused rather than a per-failure-type field.
- **Story-lifecycle state is not read anywhere yet in the resume path.** This story only *writes* `awaiting_escalation_checkpoint` via `setStoryLifecycleState`; nothing in `checkpoints.ts` or elsewhere currently *reads* `getStoryLifecycleState` to branch behavior. This matches ARC-1374's own stated scope ("types + persistence plumbing only... migration is a later story") — this story is one of the first real call sites, and per its own In Scope ("try/catch wrapping... routing caught errors into the escalation-checkpoint queue") only requires making the write-side transition observable in the sidecar, not building new read-side consumers. If a later end-to-end regression story (ARC-1383) expects to assert on this field being read by some UI/route, that is out of this story's scope.
- **Batch context:** this plan is executed as part of Wave 2 / Batch 2-A alongside ARC-1375, ARC-1377, and ARC-1380. ARC-1380 has a structurally identical goal (guard-failure → `awaiting_escalation_checkpoint`) applied to `archiveStoryArtifacts` instead of `provisionWorktree` — implementers of ARC-1380 should follow this same try/catch + `setStoryLifecycleState` + generic-`CheckpointQueueEntry` pattern for consistency, though that is documented in ARC-1380's own plan, not enforced by this one.
