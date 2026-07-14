# ARC-1378 Add new human implementation-review gate after self-review passes - Implementation Plan

**Issue:** `issues/story-lifecycle/ARC-1378-add-implementation-review-gate.md`
**Completion Summary:** `task-completions/ARC-1378-COMPLETION-SUMMARY.md` (TBD)
**Approach:** Single approach — pattern established. This story follows the exact same pattern ARC-1376 used to add the planning-artifact gate: (1) add a new discriminator field to `CheckpointQueueEntry`, (2) push a checkpoint entry after a specific trigger condition (here: self-review passes clean), (3) wire the resume handler to call `resumeArtifactPRReviewLoop` with `artifactKind: 'implementation'`. The sub-machine logic (ARC-1375) and lifecycle states (`awaiting_implementation_review` / `addressing_implementation_comments`) already exist — this story is pure wiring, zero new mechanisms. Phase 1 (Approach) is skipped per the write-implementation-plan skill's rule that Phase 1 may be skipped when exploration reveals a clear existing pattern to follow, even at HIGH risk tier.
**Owner:** build agent
**Date:** 2026-07-14

## Scope & Alignment

This plan covers the three In-Scope items from the issue, and only those:

1. **laneRunner.ts wiring** — insert a new checkpoint-push after self-review passes clean (lines 682–686 in current code), before the existing pending-gate logic (line 711+). Open a real PR for the story's implementation branch at this point (the actual code delivery branch, not a side-channel artifact). Set the new `implementationReview` discriminator field (parallel to `planningArtifact`).
2. **CheckpointQueueEntry extension** — add a new optional `implementationReview?: { storyKey: string; branch: string }` field to `types.ts` (parallel to the existing `planningArtifact` field from ARC-1371/1376).
3. **checkpoints.ts resume handler** — add a new `if (entry.implementationReview !== undefined)` block (mirroring the ARC-1376 `planningArtifact` block at lines 170–227) that calls `resumeArtifactPRReviewLoop({ artifactKind: 'implementation', ... })` and handles the three outcomes ('advance' → completion_summary_drafting, 'addressing_comments' → re-queue with renotifyMessage, 'still_awaiting_merge' → silent re-queue).

Explicitly NOT covered (per issue's Out of Scope): the shared sub-machine itself (ARC-1375 — already built), self-review state tracking (ARC-1377 — already built), and any changes to the sub-machine logic or lifecycle state enum (the two required states — `awaiting_implementation_review` / `addressing_implementation_comments` — already exist in `types.ts` as of ARC-1374).

Acceptance criteria mapping:
- **AC1** (local-code-review passes clean → real PR opened, lane pauses at awaiting_implementation_review, does not proceed directly to completion-summary/archive) → Step 1 (checkpoint push at line ~686) + Step 2 (PR creation with correct branch) + Step 4 (regression test: self-review clean → checkpoint pushed with `implementationReview` set, lane status = 'paused').
- **AC2** (implementation-review PR is merged → lane advances to completion_summary_drafting) → Step 3 (resume handler, `outcome === 'advance'` branch: set lifecycle state to `'completion_summary_drafting'`, dequeue, resume lane) + Step 5 (resume test: merged PR → outcome 'advance' → lifecycle state updated, entry removed from queue).
- **AC3** (implementation-review PR has unresolved comments → shared comment-addressing flow runs, checkpoint re-queues) → Step 3 (resume handler, `outcome === 'addressing_comments'` branch: calls the same shared `resumeArtifactPRReviewLoop` used by ARC-1376, which internally delegates to `addressComments` callback when comments exist, then re-queues with `renotifyMessage`) + Step 5 (resume test: unresolved comments → outcome 'addressing_comments' → entry stays in queue with `renotifyMessage` set).
- **AC4** (no story can reach archived state without having passed through awaiting_implementation_review with a merged PR) → This is a natural consequence of Step 1 (the new checkpoint is mandatory before the lane can reach 'done' status and trigger archiving at line 852) + Step 6 (end-to-end regression test: story with clean self-review → checkpoint pushed → PR merged → resume → completion-summary step → archive — verify the sequence cannot be bypassed, and verify a story with self-review pass but no resume call remains paused and never archives).

## Assumptions & Dependencies

- Dependencies explicitly stated in issue: ARC-1375 (shared sub-machine) and ARC-1377 (self-review state tracking) — both confirmed complete per the runbook (Wave 2 gate checked off 2026-07-13).
- The lifecycle states `awaiting_implementation_review` and `addressing_implementation_comments` already exist in the `StoryLifecycleState` union (added by ARC-1374, lines 13–29 of `types.ts`).
- The shared `resumeArtifactPRReviewLoop` function in `checkpointGit.ts` already supports `artifactKind: 'implementation'` — the mapping table `ARTIFACT_KIND_LIFECYCLE_STATES` (lines 726–733) includes the entry `implementation: { awaitingState: 'awaiting_implementation_review', addressingState: 'addressing_implementation_comments' }`.
- The story's implementation branch name follows the existing convention: `<storyKey>` (e.g. `ARC-1378`), matching the pattern used throughout `laneRunner.ts`, `checkpointGit.ts`, and `worktreeGit.ts`.
- The PR opened at this gate is the **actual code delivery PR** — the branch that holds the story's implementation (provisioned by the worktree step earlier in the lane), not a separate artifact-review branch (contrast with `planningArtifact`, which opens a side-channel `plan/<lane>/<storyKey>` branch for the plan file only).
- This gate fires **only** when `parseLocalCodeReviewOutcome` returns `{ outcome: 'clean' }` — it does NOT fire when self-review is absent (no `local-code-review` skill invoked), or when outcome is `'escalated'` (that path already pushes its own `awaiting_escalation_checkpoint` entry at lines 688–707, mutually exclusive with this gate).
- The gate is inserted **before** the existing pending-gate logic (line 711+) and **before** the planning-artifact detection logic (lines 789–798) — it becomes the new earliest possible pause point after self-review, ensuring no subsequent artifact-commit or archive logic runs until the implementation PR is merged.
- The implementation-review checkpoint's `prUrl` field is populated via fire-and-forget PR creation in `checkpointGit.ts` (exactly as ARC-1376 does for planning artifacts and ARC-1302 does for regular checkpoints). If PR creation fails, the `prUrl` remains `undefined` and the resume handler falls through to unconditional resume (AC6 fallback from ARC-1376, reused here).
- No migration of existing sidecar state is needed — `implementationReview` is an optional field on `CheckpointQueueEntry`, defaulting to `undefined` on all pre-existing entries (matches the `planningArtifact` precedent from ARC-1371/1376).

## Implementation Steps

### Step 1 — Add `implementationReview` field to `CheckpointQueueEntry`

**Files:** `packages/server/src/state/types.ts`

**Action:**
Add a new optional field to the `CheckpointQueueEntry` interface (lines 96–122), placed immediately after the `planningArtifact` field (line 121), with a parallel doc comment:

```typescript
  /**
   * Set when this checkpoint entry represents an implementation-code-review
   * approval-gate PR rather than a planning-artifact or runbook checkoff
   * checkpoint (ARC-1378). The PR URL is the story's real implementation branch.
   * Undefined for all other checkpoint types.
   */
  implementationReview?: { storyKey: string; branch: string };
```

**Verification:** `run_typecheck` on `packages/server` passes with zero new errors. Confirm the interface change does not break any existing consumers (existing code constructs `CheckpointQueueEntry` objects without this field — TypeScript optional-field rules ensure backward compatibility).

### Step 2 — Push implementation-review checkpoint after self-review passes clean

**Files:** `packages/server/src/runner/laneRunner.ts`

**Action:**
Insert a new checkpoint-push block immediately after the `if (selfReviewOutcome.outcome === 'clean')` branch at lines 682–686, replacing the existing fall-through comment ("AC2 — no lane pause on clean; fall through to the existing pending-gate logic unchanged") with actual checkpoint logic:

1. Extract `storyKey` from `step.id` (already done at line 673: `const storyKey = stripNamespace(step.id);`).
2. Determine the implementation branch name: `const implBranch = storyKey;` (matches the worktree branch name created earlier in the lane by `ensureWorktreeForStory` — the story's own implementation branch, not a side-channel artifact branch).
3. Set lifecycle state to `awaiting_implementation_review` via `setStoryLifecycleState(withRecord, storyKey, 'awaiting_implementation_review')`.
4. Push a new `CheckpointQueueEntry` with `checkpointLevel: 'high'`, `stepId: step.id`, and `implementationReview: { storyKey, branch: implBranch }`.
5. Persist state via `setState` / `writeSidecar`.
6. Fire-and-forget PR creation: call `createCheckpointBranchAndPR({ repoRoot, lane, stepId: step.id, branchOverride: implBranch })` (new `branchOverride` parameter added in Step 2b below). On success, update the entry's `prUrl` field in the queue (mirroring the pattern at lines 753–776 for sub-step checkpoints).
7. Set `laneStatusMap.set(runbookPath, 'paused')`.
8. Return early (no fall-through to pending-gate logic).

**Step 2b — Extend `createCheckpointBranchAndPR` to accept `branchOverride`:**

**Files:** `packages/server/src/git/checkpointGit.ts`

**Action:**
Add an optional `branchOverride?: string` parameter to the `CreateCheckpointBranchAndPROptions` interface (around line 600). When `branchOverride` is provided, use it as the branch name for the PR; when absent, fall back to the existing `checkpoint/${lane}/${stepId}` pattern.

Update the `createCheckpointBranchAndPR` function body (lines ~610–660) to replace the hardcoded branch pattern with:
```typescript
const branch = opts.branchOverride ?? `checkpoint/${lane}/${stepId}`;
```

**Verification:**
- `run_typecheck` passes.
- `run_lint` passes.
- Unit test in Step 4 exercises the new checkpoint push: mock `parseLocalCodeReviewOutcome` to return `{ outcome: 'clean', iterations: 2 }`, run a lane step that invokes `local-code-review`, assert the checkpoint queue gains an entry with `implementationReview` set and `prUrl` populated (via mock or real git call), assert lane status = 'paused'.

### Step 3 — Wire resume handler to call shared sub-machine for implementation-review checkpoints

**Files:** `packages/server/src/routes/checkpoints.ts`

**Action:**
Add a new `if (entry.implementationReview !== undefined)` block in the `POST /checkpoints/:stepId/resume` handler (lines 153–279), placed **before** the existing `if (entry.planningArtifact !== undefined)` block (line 170). This ensures implementation-review checkpoints are handled first if both discriminators were somehow set (defensive ordering, though in practice they are mutually exclusive).

The block follows the exact structure of the ARC-1376 `planningArtifact` block (lines 170–227):

1. Respond immediately with `res.json({ ok: true });`.
2. If `entry.prUrl === undefined`, call `resumeLane(entry, entryIndex, current)` unconditionally (AC6 fallback — PR creation failed, no gate to enforce).
3. Extract `{ storyKey, branch } = entry.implementationReview` and `prUrl = entry.prUrl`.
4. Call `resumeArtifactPRReviewLoop({ prUrl, artifactKind: 'implementation', addressComments: (comments) => addressImplementationComments(entry, comments, current.repoRoot, branch) })` (the `addressImplementationComments` helper is defined in Step 3b below).
5. Handle the three outcomes:
   - **'advance'** — PR is merged:
     - Set lifecycle state to `'completion_summary_drafting'` via `setStoryLifecycleState(latest, storyKey, 'completion_summary_drafting')`.
     - Call `resumeLane(entry, entryIndex, withLifecycle)` to dequeue the entry and resume the lane.
   - **'addressing_comments'** — agent addressed comments, revision pushed:
     - Update the entry's `renotifyMessage` field with `result.renotifyMessage` (same logic as lines 199–216 in the `planningArtifact` block).
     - Re-persist state. Entry stays in queue (no `resumeLane` call).
   - **'still_awaiting_merge'** — no comments, not merged:
     - No-op, entry stays in queue, no state mutation (same as line 219).
6. Wrap in `.catch()` — log error and fall through to `resumeLane` on unexpected failure (same as lines 221–224).

**Step 3b — Add `addressImplementationComments` helper:**

**Files:** `packages/server/src/routes/checkpoints.ts`

**Action:**
Add a new internal helper function below the existing `addressPlanningArtifactComments` function (around line 310+):

```typescript
/**
 * Invoked by the implementation-review sub-machine when unresolved PR comments
 * exist on the story's implementation branch. Agent revises the code in the
 * worktree, commits, and pushes the revision to the implementation branch.
 * ARC-1378.
 */
async function addressImplementationComments(
  entry: CheckpointQueueEntry,
  comments: BitbucketPRComment[],
  repoRoot: string,
  branch: string,
): Promise<void> {
  // Prompt agent to address comments in the story's worktree
  // (implementation branch was already checked out by earlier worktree-provisioning step).
  // Agent revises code, commits, pushes to `branch`.
  // Placeholder for agent orchestration call — actual implementation deferred to execution.
  throw new Error(`[ARC-1378] addressImplementationComments not yet implemented for ${entry.stepId}`);
}
```

(Note: The actual agent orchestration call is out of scope for this plan — this story only wires the gate; the comment-addressing agent flow is a follow-on story if needed. For now, the helper throws, which propagates to the `.catch()` in Step 3's resume handler and triggers the fallback `resumeLane` path, ensuring the lane is never permanently blocked.)

**Verification:**
- `run_typecheck` passes.
- `run_lint` passes.
- Unit test in Step 5 exercises all three resume outcomes: mocked `resumeArtifactPRReviewLoop` returns each outcome in turn, assert correct state transitions, entry queue mutations, and lane resume calls.

### Step 4 — Unit test: self-review clean → implementation-review checkpoint pushed

**Files:** `packages/server/src/__tests__/laneRunner.test.ts`

**Action:**
Add a new test in the `describe('ARC-1377: self-review outcome tracking')` block (lines 1269–1330), titled `"ARC-1378: self-review clean → implementation-review checkpoint pushed, lane paused"`:

1. Mock `parseLocalCodeReviewOutcome` to return `{ outcome: 'clean', iterations: 2 }`.
2. Mock `createCheckpointBranchAndPR` to return `{ prUrl: 'https://bitbucket.org/proalpha/pa.aid.conductor.ts/pull-requests/999' }`.
3. Mock `executeStep` to return `{ outcome: 'success', events: [{ type: 'text', text: '...' }] }` (triggers self-review detection).
4. Set up a minimal runbook with a single HIGH story step whose substeps include `[ ] Run local-code-review`.
5. Call `runLane(...)`.
6. Assert:
   - `getState().checkpointQueue.length === 1`
   - `checkpointQueue[0].implementationReview !== undefined`
   - `checkpointQueue[0].implementationReview.storyKey === 'ARC-1378'` (or the test fixture's story key)
   - `checkpointQueue[0].implementationReview.branch === 'ARC-1378'`
   - `checkpointQueue[0].prUrl === 'https://bitbucket.org/proalpha/pa.aid.conductor.ts/pull-requests/999'`
   - `laneStatusMap.get(runbookPath) === 'paused'`
   - Lifecycle state for the story is `'awaiting_implementation_review'` (via `getStoryLifecycleState(getState(), 'ARC-1378')`).

**Verification:** `run_tests` — new test passes, no existing tests regress.

### Step 5 — Unit test: resume handler covers all three sub-machine outcomes

**Files:** `packages/server/src/__tests__/checkpoints.test.ts`

**Action:**
Add a new `describe('ARC-1378: implementation-review checkpoint resume')` block (placed after the ARC-1376 `planning-artifact` test block, around line 760+), with three tests:

1. **"merged PR → outcome 'advance' → lifecycle state updated to completion_summary_drafting, entry dequeued, lane resumes"**:
   - Mock `isPRMerged` to return `true`.
   - Populate `checkpointQueue` with an entry having `implementationReview: { storyKey: 'ARC-1378', branch: 'ARC-1378' }` and `prUrl: 'https://...'`.
   - POST `/checkpoints/<stepId>/resume`.
   - Await async completion (use `setImmediate` callback or `await new Promise(resolve => setImmediate(resolve))` to flush the fire-and-forget promise).
   - Assert `getState().checkpointQueue.length === 0` (entry dequeued).
   - Assert `getStoryLifecycleState(getState(), 'ARC-1378') === 'completion_summary_drafting'`.
   - Assert `runLane` was called (via mock or spy).

2. **"unresolved comments → outcome 'addressing_comments' → entry stays in queue, renotifyMessage set"**:
   - Mock `isPRMerged` to return `false`.
   - Mock `listUnresolvedPRComments` to return `[{ id: '1', text: 'Fix this', resolved: false }]`.
   - Mock `addressImplementationComments` to resolve successfully (no-op).
   - Populate queue with `implementationReview` entry.
   - POST `/checkpoints/<stepId>/resume`.
   - Await async completion.
   - Assert `getState().checkpointQueue.length === 1` (entry NOT dequeued).
   - Assert `checkpointQueue[0].renotifyMessage !== undefined` and contains `"Agent addressed 1 implementation comment"`.

3. **"no comments, not merged → outcome 'still_awaiting_merge' → silent re-queue, no state change"**:
   - Mock `isPRMerged` to return `false`.
   - Mock `listUnresolvedPRComments` to return `[]`.
   - Populate queue with `implementationReview` entry.
   - POST `/checkpoints/<stepId>/resume`.
   - Await async completion.
   - Assert `getState().checkpointQueue.length === 1` (entry NOT dequeued).
   - Assert `checkpointQueue[0].renotifyMessage === undefined` (no agent action, no update).
   - Assert lifecycle state unchanged (still `'awaiting_implementation_review'`).

**Verification:** `run_tests` — all three new tests pass, no regressions in existing checkpoint tests.

### Step 6 — End-to-end regression test: story cannot archive without passing implementation-review gate

**Files:** `packages/server/src/__tests__/laneRunner.test.ts`

**Action:**
Add a new test in a top-level `describe('ARC-1378: end-to-end lifecycle enforcement')` block, titled `"story with clean self-review pauses at implementation-review checkpoint and cannot archive until PR is merged"`:

1. Set up a minimal runbook for a single story with substeps: `[ ] Implement`, `[ ] Run local-code-review`, `[ ] Write completion summary`.
2. Mock `parseLocalCodeReviewOutcome` to return `{ outcome: 'clean', iterations: 1 }` after the `local-code-review` substep.
3. Mock `createCheckpointBranchAndPR` to return a `prUrl`.
4. Call `runLane(...)`.
5. Assert lane pauses with an `implementationReview` checkpoint in the queue (same assertions as Step 4).
6. Assert lane status = `'paused'` (NOT `'done'`).
7. Simulate no resume call (do not call POST `/checkpoints/:stepId/resume`).
8. Assert `archiveStoryArtifacts` is **not** called (verify via mock/spy that the archive logic at line 858 is never reached).
9. Simulate PR merge: mock `isPRMerged` to return `true`, then POST `/checkpoints/:stepId/resume`.
10. Await async completion.
11. Assert checkpoint dequeued.
12. Assert lane resumes and reaches `'done'` status (the completion-summary substep executes next).
13. Assert `archiveStoryArtifacts` is eventually called (lane completes and triggers archive at line 858).

This test directly proves AC4: no story can bypass the gate and reach archived state without the implementation-review PR being merged.

**Verification:** `run_tests` — new test passes, confirms the enforcement contract.

### Step 7 — Full verification pass

**Files:** N/A (verification only)

**Action:** Run the full `packages/server` test suite and typecheck to confirm the additive changes introduce zero regressions in unrelated subsystems (`runbooks.summary.test.ts`, `sessions.test.ts`, `state.test.ts`, `sidecar.test.ts`, etc. all reference `CheckpointQueueEntry` and must continue to pass with the new optional field).

**Verification:**
- `run_typecheck` on `/repos/<ARC-1378 worktree>` returns clean.
- `run_tests` returns all green — only `laneRunner.test.ts` and `checkpoints.test.ts` have new tests; all other test files pass unmodified.
- `run_lint` returns clean.

## Testing & Validation

- **`run_typecheck`** — confirms the new `implementationReview` optional field compiles cleanly against all existing `CheckpointQueueEntry` construction sites (none are required to set the field, ensuring backward compatibility).
- **`run_tests`** — confirms:
  - Step 4 test: self-review clean path pushes the new checkpoint and pauses the lane (AC1).
  - Step 5 tests: all three resume outcomes ('advance', 'addressing_comments', 'still_awaiting_merge') are handled correctly by the new resume handler block (AC2, AC3).
  - Step 6 test: end-to-end enforcement — story cannot archive without the implementation-review PR merge (AC4).
  - No regressions in existing tests (`laneRunner.test.ts` ARC-1377 block, `checkpoints.test.ts` ARC-1376 block, `sidecar.test.ts`, `runbooks.*.test.ts`) — confirms the additive change is isolated.
- **`run_lint`** — confirms no style/lint violations in the three edited files (`types.ts`, `laneRunner.ts`, `checkpoints.ts`).

## Risks & Open Questions

- **Agent comment-addressing orchestration is stubbed out.** The `addressImplementationComments` helper in Step 3b throws `Error("not yet implemented")`, which means the 'addressing_comments' outcome will currently trigger the `.catch()` fallback in the resume handler and fall through to unconditional `resumeLane`. This is acceptable for this story because (a) the issue's Out of Scope explicitly does not include the agent's comment-revision logic, and (b) the fallback ensures the lane is never permanently blocked by the new gate (degraded-mode resilience, matching ARC-1376's error-handling pattern). A follow-on story can implement the agent orchestration call inside `addressImplementationComments` to complete the comment-addressing loop.
- **PR creation failure fallback (AC6 from ARC-1376) is reused here.** If `createCheckpointBranchAndPR` fails (e.g. Bitbucket API unavailable), the checkpoint entry's `prUrl` remains `undefined`, and the resume handler falls through to unconditional `resumeLane` (lines 173–178 in the ARC-1376 block, same logic reused for `implementationReview`). This means the human review gate is skipped when PR creation fails — a known trade-off from ARC-1376 that ensures the lane is never permanently blocked by transient infrastructure failures. If strict enforcement is required in all cases, a future story can add a retry/manual-override flow.
- **No migration of existing sidecar state is needed.** `implementationReview` is an optional field, defaulting to `undefined` on all pre-ARC-1378 checkpoint entries. Existing sessions continue to work unchanged; the new gate only applies to stories that execute after this story merges.
- **Mutual exclusivity of `planningArtifact` vs `implementationReview` is enforced by control flow, not schema.** A `CheckpointQueueEntry` could theoretically have both fields set (TypeScript allows it), but in practice the lane runner's control flow ensures they are mutually exclusive: planning-artifact checkpoints are pushed at lines 795–797 (triggered by `detectPlanningArtifact`), while implementation-review checkpoints are pushed in this story at the new self-review-clean block (lines ~682–710), and these are two non-overlapping code paths. If stronger enforcement is desired, a future story can introduce a tagged union type for `CheckpointQueueEntry` (discriminated by `kind: 'planning' | 'implementation' | 'generic'`) — but that is a breaking schema change and is deferred here to minimize risk.
- **The implementation-review checkpoint fires only once per story.** If the PR is closed/reopened or comments are added after the first resume, the checkpoint does not re-queue automatically — the supervisor must manually resume again. This is consistent with the planning-artifact gate's behavior (ARC-1376) and is acceptable because (a) the issue's acceptance criteria only require one pause point (after self-review passes), not continuous polling, and (b) background polling is explicitly out of scope (the sub-machine is resume-driven only, per ARC-1375's design). If continuous monitoring is needed, a future story can add a background poller similar to `prPollIntervalMs` for runtime checkpoints.
