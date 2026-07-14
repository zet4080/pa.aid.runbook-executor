# ARC-1376 Migrate planning-artifact gate onto shared PR-review sub-machine (supersedes ARC-1372) - Implementation Plan

**Issue:** `issues/story-lifecycle/ARC-1376-migrate-planning-artifact-gate-onto-shared-sub-machine.md`
**Completion Summary:** `task-completions/ARC-1376-COMPLETION-SUMMARY.md` (TBD)
**Approach:** Single approach — pattern established. ARC-1375 already built `resumeArtifactPRReviewLoop` in `checkpointGit.ts` specifically so this story could call it instead of re-implementing merge/comment-gate logic. There is no competing design to choose between — this story is a rip-and-replace migration of existing call sites onto an already-tested shared function. Phase 1 (Approach) is skipped per the write-implementation-plan skill's rule that it may be skipped when exploration reveals a clear existing pattern to follow.
**Owner:** build agent
**Date:** 2026-07-13

## Scope & Alignment

This plan rewires the two existing ARC-1371 planning-artifact (plan/decision) PR-gate call sites — `checkpoints.ts`'s resume route and `laneRunner.ts`'s `triggerPlanningArtifactPR`/`resumeLaneFromPlanningPR` — onto ARC-1375's `resumeArtifactPRReviewLoop`. It removes the old poller-based (`startPlanningPRPoller`/`planningPRPollers`) merge-detection mechanism and the 409-rejection guard in the resume route, and removes the decision-document `status: pending`/`status: answered` markdown-field check in `resumeLaneFromPlanningPR`. `artifactKind: 'decision'` documents move onto the exact same explicit-resume-driven gate as `artifactKind: 'plan'` documents — no more special-cased polling or file-content parsing for decisions.

Acceptance criteria mapping:
- **AC1** (open PR, unresolved comments, resume triggered → NOT 409, routes through shared comment-addressing flow) → Step 2 (remove the 409 guard block in `checkpoints.ts`) + Step 3 (new planning-artifact-specific resume branch that calls `resumeArtifactPRReviewLoop` and invokes its `addressComments` injection instead of `addressPRComments`'s ARC-1304 generic path).
- **AC2** (merged PR, resume triggered → advances to next phase, `AGENT_EXECUTING(code)` for plan artifacts) → Step 3 (the `'advance'` outcome branch dequeues the checkpoint and calls `runLane`, exactly mirroring today's post-merge resume behavior, now driven by an explicit resume call instead of a poller callback) + Step 4 (`setStoryLifecycleState` call recording the appropriate next-phase state).
- **AC3** (old `planningPRPollers` background polling removed) → Step 1 (delete `startPlanningPRPoller`/`planningPRPollers` map/startup-recovery wiring from `laneRunner.ts` and `index.ts`; `startPlanningPRPoller` itself stays in `checkpointGit.ts` only if still exported/tested elsewhere — see Assumptions, it is deleted too since ARC-1376 is its only caller).
- **AC4** (decision documents use PR-merge-gated sub-machine via `artifactKind: 'decision'`; old markdown-field mechanism removed) → Step 3 (decision artifacts go through the identical `resumeArtifactPRReviewLoop` call as plan artifacts, differing only in `artifactKind: 'decision'`) + Step 5 (delete the `status: pending`/`answered` `readFileSync` check in `resumeLaneFromPlanningPR`, and the `decisions/_template.md` `status`/`selected_approach` frontmatter contract's role in gating — the file's own content is no longer read by the resume path).

## Assumptions & Dependencies

- **Dependency ARC-1375 (satisfied, merged to `main` at commit `7d4ebec`):** `resumeArtifactPRReviewLoop`, `ArtifactKind`, `ArtifactKindLifecycleStates`, `ARTIFACT_KIND_LIFECYCLE_STATES`, `ResumeArtifactPRReviewLoopOptions`, `ResumeArtifactPRReviewLoopResult` are already exported from `packages/server/src/git/checkpointGit.ts`. This plan imports and calls that function; it does not modify its implementation.
- **`startPlanningPRPoller` has exactly one production call site pair to remove:** `laneRunner.ts` (`triggerPlanningArtifactPR`, `planningPRPollers` map) and `index.ts` (startup-recovery loop over `checkpointQueue` entries with `planningArtifact`+`prUrl` set). No other file imports `startPlanningPRPoller` from `checkpointGit.ts` in production code (confirmed: only `checkpointGit.ts` itself, `checkpointGit.test.ts`, `index.ts`, and `laneRunner.ts`/`laneRunner.test.ts` reference the symbol). This plan deletes the exported function and its tests entirely, since ARC-1376 removing "the old planningPRPollers background polling mechanism" (AC3) means the underlying poller primitive itself is dead code once its only callers are gone — keeping an untested, uncalled exported function would violate the "remove entirely" language in the issue's Goal.
- **`isPRMerged` (used internally by `resumeArtifactPRReviewLoop`) stays.** It is a pure Bitbucket-API helper with no polling behavior of its own — only the `setTimeout`-loop wrapper (`startPlanningPRPoller`) is removed. `isPRMerged` continues to be exported and used both directly (nowhere else, after this migration) and transitively via `resumeArtifactPRReviewLoop`.
- **`createPlanPR`/`createDecisionPR` (branch+PR creation at checkpoint-hit time) are unchanged.** This story only touches the *resume* side of the gate (what happens when the supervisor clicks Resume or a poller used to fire). PR creation at the point the artifact file is first detected (`triggerPlanningArtifactPR`'s call to `createPlanPR`/`createDecisionPR`, pushing the initial `CheckpointQueueEntry`) is Out of Scope — untouched.
- **The `CheckpointQueueEntry.planningArtifact` field's `artifactType: 'plan' | 'decision'` shape is unchanged.** `resumeArtifactPRReviewLoop`'s `ArtifactKind` is `'plan' | 'implementation' | 'decision'` — this story only ever passes `artifactType` (cast/mapped 1:1, since `'plan'`/`'decision'` are valid `ArtifactKind` members already) as `artifactKind`. No new field is added to `CheckpointQueueEntry` for this story (that is ARC-1378's job, which is Out of Scope here per the issue's explicit note that the new `artifactKind: 'implementation'` gate is "a separate artifactKind wired in a later story").
- **The `addressComments` callback injected into `resumeArtifactPRReviewLoop` reuses the existing ARC-1304 per-comment `executeStep` + reply + resolve pattern**, not a new mechanism. `checkpoints.ts` already has this logic in `addressPRComments` (spawns `executeStep` per unresolved comment, replies on the thread, resolves it). This plan extracts that inner per-comment loop into a shared helper reusable from both the existing ARC-1304 generic-runbook-checkpoint resume path (unchanged, stays on its own PR/comment logic — it does not call `resumeArtifactPRReviewLoop`, only planning-artifact entries do) and the new planning-artifact resume path's `addressComments` callback. This is a refactor-for-reuse, not new behavior — the callback's job (spawn agent on each comment, reply, resolve) is identical to what `addressPRComments` already does today for non-planning-artifact checkpoints.
- **`StoryLifecycleState` values for "next phase" on `outcome: 'advance'` (AC2's `AGENT_EXECUTING(code)` language):** ARC-1374's enum has `agent_executing` (no distinct `(code)`/`(plan)`/`(decision)` sub-variants — phase is carried separately via `StoryLifecycleRecord.phase: 'plan'|'code'|'decision'`). This plan calls `setStoryLifecycleState(current, storyKey, 'agent_executing', 'code')` for plan-artifact merges (AC2's literal wording) and `setStoryLifecycleState(current, storyKey, 'agent_executing', 'decision')` for decision-artifact merges, matching the existing `(state, phase)` two-argument convention already used elsewhere in `laneRunner.ts` (e.g. `sidecar.test.ts`'s `setStoryLifecycleState(state, 'ARC-1374', 'agent_executing', 'code')` pattern). This is additive lifecycle-state bookkeeping using ARC-1374's existing API — no new states are introduced.
- **`checkpoints.ts`'s existing 409-guard removal does not change the generic (non-planning-artifact) ARC-1304 comment-thread resume path.** That path (an entry with a `prUrl` but no `planningArtifact` — e.g. ordinary checkpoint/GATE PRs from `createCheckpointBranchAndPR`) is untouched; only entries with `planningArtifact !== undefined` are re-routed onto `resumeArtifactPRReviewLoop`.
- **No test file for `startPlanningPRPoller` survives** — the `describe('startPlanningPRPoller', ...)` block in `checkpointGit.test.ts` (3 tests) is deleted alongside the function, per the "remove entirely" instruction in AC3.
- **`decisions/_template.md` in the planning repo (this repo, not `pa.aid.conductor.ts`) is out of scope for this plan's code changes** — it lives in `pa.aid.runbook-executor`, not the application repo, and this plan only touches `pa.aid.conductor.ts` application code per the issue's In Scope list (`checkpointGit.ts`, `checkpoints.ts`, `laneRunner.ts`). The template's `status`/`selected_approach` frontmatter fields become presentational-only for decision documents once this migration lands (no code reads them anymore) — noting this in Risks below since it is a documentation follow-up, not an application-code task.

## Implementation Steps

### Step 1 — Remove `startPlanningPRPoller` and the `planningPRPollers` registry

**Files:** `packages/server/src/git/checkpointGit.ts`, `packages/server/src/git/checkpointGit.test.ts`, `packages/server/src/runner/laneRunner.ts`, `packages/server/src/index.ts`

**Action:**

In `checkpointGit.ts`: delete the exported `startPlanningPRPoller` function (the `setTimeout`-based polling loop under the "ARC-1371: Planning-artifact ... PR approval gate" section comment) entirely. Leave `isPRMerged` and `listUnresolvedPRComments` untouched — both are still used by `resumeArtifactPRReviewLoop`.

In `checkpointGit.test.ts`: delete the `describe('startPlanningPRPoller', ...)` block (3 tests: "calls onMerged once...", "stop() prevents further calls...", "survives a thrown poll error...").

In `laneRunner.ts`: remove the `startPlanningPRPoller` import from the `../git/checkpointGit.js` import statement. Delete the `planningPRPollers` module-level `Map<string, { stop: () => void }>` declaration and its `.clear()` call inside `_resetLaneStateForTesting`. In `triggerPlanningArtifactPR`, delete the `if (result.prUrl !== undefined) { const poller = startPlanningPRPoller({...}); planningPRPollers.set(...); } else { console.warn(...) }` block — replace it with nothing extra beyond what Step 3 adds (the checkpoint entry push and lane-pause already happen unconditionally above this block and are preserved as-is; only the poller-start/no-poller-warning branching is deleted).

In `index.ts`: remove the `startPlanningPRPoller` import from `./git/checkpointGit.js`. Delete the "ARC-1371: Startup recovery for in-flight planning-artifact PR pollers" `for` loop (the block iterating `initialState.checkpointQueue` and calling `startPlanningPRPoller` for entries with `planningArtifact !== undefined && prUrl !== undefined`). No replacement logic is needed here — this story's explicit-resume model has no startup-time work to recover (there is no in-memory poller state to reconstruct).

**Verification:** `run_typecheck` on `packages/server` compiles cleanly with zero references to `startPlanningPRPoller`/`planningPRPollers` remaining anywhere in `packages/server/src` (confirm via `grep -rn "startPlanningPRPoller\|planningPRPollers" packages/server/src` returning no matches outside comments/commit-message strings, if any survive from history).

### Step 2 — Remove the 409-rejection guard in the resume route

**Files:** `packages/server/src/routes/checkpoints.ts`

**Action:**

Delete the `if (entry.planningArtifact !== undefined && entry.prUrl !== undefined) { res.status(409).json({...}); return; }` block inside the `router.post('/checkpoints/:stepId/resume', ...)` handler (the block commented "ARC-1371: planning-artifact ... PR checkpoints with an open prUrl resume automatically when the PR is merged ... Reject with 409"). This satisfies AC1's "does NOT reject with 409" requirement directly.

**Verification:** Re-reading the handler top-to-bottom confirms no code path returns HTTP 409 for any `planningArtifact` entry; the only remaining branch point is Step 3's new planning-artifact-specific handling.

### Step 3 — Add a planning-artifact-specific resume branch that calls `resumeArtifactPRReviewLoop`

**Files:** `packages/server/src/routes/checkpoints.ts`, `packages/server/src/runner/laneRunner.ts`

**Action:**

In `checkpoints.ts`, immediately after the entry-lookup (`const entry = current.checkpointQueue[entryIndex];`) and before the existing generic `if (entry.prUrl) { ... }` ARC-1304 branch, insert a new `if (entry.planningArtifact !== undefined) { ... }` branch that:

1. Responds `res.json({ ok: true })` immediately (matching the existing async-branch convention — the heavy lifting happens fire-and-forget).
2. If `entry.prUrl === undefined` (the AC6-fallback case from ARC-1371, where the artifact was committed straight to `main` because PR creation failed): falls through to the existing no-`prUrl` dequeue-and-resume path unchanged (reuses the existing `resumeLane` helper already defined at the bottom of the file) — there is no PR to gate on, so `resumeArtifactPRReviewLoop` is not called.
3. Otherwise, calls `await resumeArtifactPRReviewLoop({ prUrl: entry.prUrl, artifactKind: entry.planningArtifact.artifactType, addressComments: (comments) => addressPlanningArtifactComments(entry, comments, current.repoRoot) })`, where `addressPlanningArtifactComments` is the shared per-comment helper described in Step 4 (extracted from `addressPRComments`'s existing per-comment loop body).
4. Branches on the returned `outcome`:
   - `'advance'` → dequeue the entry (reuse `resumeLane(entry, entryIndex, current)`, which already re-parses the runbook from disk and calls `runLane`), and additionally call `setStoryLifecycleState` with the phase-specific next state from the Assumptions section (`'agent_executing'`, phase `'code'` for `artifactType === 'plan'`, phase `'decision'` for `artifactType === 'decision'`), persisting via `writeSidecar` before the dequeue's own `setState`/`writeSidecar` pair (so the lifecycle-state write is not clobbered).
   - `'addressing_comments'` → do NOT dequeue the entry (mirrors today's ARC-1304 behavior for the generic path — comments are being addressed, the checkpoint stays queued); instead, patch the existing `CheckpointQueueEntry` in `current.checkpointQueue` with `renotifyMessage: result.renotifyMessage`, persist via `setState`/`writeSidecar` (reusing the same find-by-`stepId`/replace-in-array pattern already used in `addressPRComments`).
   - `'still_awaiting_merge'` → no state mutation at all (silent re-queue, per AC3's "no rejection and no agent action" language already proven by ARC-1375's tests) — the entry stays in the queue exactly as it was.
5. Wraps the `resumeArtifactPRReviewLoop` call in a `.catch()` (or `try`/`catch` given `async`/`await` usage) that logs via `console.error('[ARC-1376] resumeArtifactPRReviewLoop failed for ...')` and falls through to the existing generic `resumeLane(entry, entryIndex, current)` fallback — mirroring the existing "never leave the lane permanently stuck on an API failure" convention already used for the `listUnresolvedPRComments(...).catch(...)` fallback a few lines below in the same file.

Add the necessary import: `resumeArtifactPRReviewLoop` from `../git/checkpointGit.js` (added to the existing import list already containing `listUnresolvedPRComments`, `replyToThread`, `resolveThread`, `fetchCommitSha`, `type BitbucketPRComment`). Add `setStoryLifecycleState` and `getState`/`setState` are already imported; add `setStoryLifecycleState` from `../state/sidecar.js` (new import) alongside the existing `writeSidecar` import from the same module.

In `laneRunner.ts`, no functional change is needed to `triggerPlanningArtifactPR` beyond Step 1's poller-block deletion — it still creates the checkpoint entry with `planningArtifact` and `prUrl` set exactly as before; only the resume side changes.

**Verification:** `run_typecheck` compiles; the new branch's control flow is traced step-by-step against AC1 (comments present → `addressComments` invoked, no 409), AC2 (merged → `runLane` called, lifecycle state advanced), AC3 (no poller anywhere in the call graph — resume is the only trigger), confirmed in Step 6's test additions.

### Step 4 — Extract `addressPlanningArtifactComments`, a shared per-comment addressing helper

**Files:** `packages/server/src/routes/checkpoints.ts`

**Action:**

Refactor the existing `addressPRComments` function's per-comment loop body (spawn `executeStep` with the comment as prompt, fetch commit SHA, reply on thread, resolve thread — the `for (const comment of unresolvedComments) { ... }` block) into a new exported (for testability) async function `addressPlanningArtifactComments(entry: CheckpointQueueEntry, unresolvedComments: BitbucketPRComment[], repoRoot: string): Promise<void>` that performs exactly that per-comment loop and returns once all comments are processed — but does NOT itself update `renotifyMessage` or persist state (that responsibility moves to the caller in Step 3, since `resumeArtifactPRReviewLoop` already builds the `renotifyMessage` string itself and returns it as part of `ResumeArtifactPRReviewLoopResult`, so patching the queue entry happens once, in the `checkpoints.ts` resume-branch's `'addressing_comments'` case, not twice).

Leave `addressPRComments` itself unchanged in signature and behavior for the existing generic (non-planning-artifact) ARC-1304 resume path — it continues to compute and persist its own `renotifyMessage` exactly as today. `addressPRComments` internally calls the same per-comment loop logic; whether it calls the new shared helper or keeps its own inline copy is an implementation detail, but the loop body (executeStep → fetchCommitSha → replyToThread → resolveThread) must not diverge between the two call sites — extracting a shared private helper function used by both `addressPRComments` and the new planning-artifact branch is the mechanism that guarantees this (rather than copy-pasting the loop twice).

**Verification:** `run_typecheck` compiles; Step 6's tests confirm both call sites (`addressPRComments` for generic checkpoints, `addressPlanningArtifactComments` for planning-artifact checkpoints via `resumeArtifactPRReviewLoop`'s injected callback) invoke `executeStep`/`replyToThread`/`resolveThread` identically per comment.

### Step 5 — Remove the decision-document `status: pending`/`answered` markdown-field check

**Files:** `packages/server/src/runner/laneRunner.ts`, `packages/server/src/__tests__/laneRunner.test.ts`

**Action:**

Delete the exported `resumeLaneFromPlanningPR` function entirely, along with its `readFileSync`-based `status: pending`/`answered` check (the block reading `decisions/${storyKey}-approach-decision.md` and re-pausing if not `answered`). This function's only caller was the now-deleted `startPlanningPRPoller` `onMerged` callback (in `triggerPlanningArtifactPR`) and `index.ts`'s startup-recovery loop (both removed in Step 1) — with both callers gone, this function is dead code. The `readFileSync` import from `'fs'` and the `path` import (if no longer used elsewhere in the file — confirm via a full-file grep for `path.` before removing) are cleaned up if they become unused.

In `laneRunner.test.ts`: delete the `describe('resumeLaneFromPlanningPR (ARC-1371)', ...)` block (3 tests: "dequeues by stepId and calls runLane for a plan artifact", "for a decision artifact with status: pending ... re-pauses without calling runLane", "for a decision artifact with status: answered ... dequeues and calls runLane"). Also delete the `describe('runLane — planning-artifact PR detection and triggering (ARC-1371)', ...)` tests that assert `startPlanningPRPollerMock` was/was-not called (`'triggerPlanningArtifactPR pauses the lane, pushes a CheckpointQueueEntry with planningArtifact set, and starts a poller when prUrl is present'`, `'triggerPlanningArtifactPR fallback path (no prUrl) pauses without starting a poller and logs a warning'`) — these two tests' *poller* assertions are removed, but their non-poller assertions (checkpoint entry pushed with correct `planningArtifact`/`prUrl` shape, lane paused) are preserved by rewriting the tests to drop only the `startPlanningPRPollerMock` expectations while keeping the `state.checkpointQueue[0]` shape assertions intact. Remove `startPlanningPRPollerMock` and its `vi.mock('../git/checkpointGit.js', ...)` entry from the test file's mock setup once no test references it.

**Verification:** `run_typecheck` and `run_tests` confirm zero remaining references to `resumeLaneFromPlanningPR`, `status: pending`, or `status: answered` markdown-parsing anywhere in `packages/server/src` (`grep -rn "resumeLaneFromPlanningPR\|status: answered\|status: pending" packages/server/src` returns no matches).

### Step 6 — Unit tests for the new resume-route planning-artifact branch

**Files:** `packages/server/src/__tests__/checkpoints.test.ts`

**Action:**

Replace the existing `describe('POST /api/checkpoints/:stepId/resume — ARC-1371 planning-artifact PR guard', ...)` block (currently 2 tests asserting the 409 response and the AC6-fallback-resumes-normally case) with a new `describe('POST /api/checkpoints/:stepId/resume — ARC-1376 planning-artifact PR-review-loop gate', ...)` block. Add `vi.mock('../git/checkpointGit.js', ...)` entry for `resumeArtifactPRReviewLoop` (a new mock alongside the existing `listUnresolvedPRComments`/`replyToThread`/`resolveThread`/`fetchCommitSha`/`createCheckpointBranchAndPR` mocks), containing:

- A test asserting that an entry with `planningArtifact` set and `prUrl` present, when `resumeArtifactPRReviewLoop` resolves `{ outcome: 'advance', awaitingState, addressingState }`, results in: `res.status` is 200 (never 409 — direct proof of AC1's "does NOT reject with 409"), the entry is dequeued from `checkpointQueue`, and `runLane` is called (proof of AC2's "advances to the next phase").
- A test asserting that when `resumeArtifactPRReviewLoop` resolves `{ outcome: 'addressing_comments', renotifyMessage: '...', awaitingState, addressingState }`, the entry stays in `checkpointQueue` with its `renotifyMessage` field updated to the returned value, and `runLane` is NOT called (proof that comments-present routes through the comment-addressing flow rather than resuming, per AC1).
- A test asserting that when `resumeArtifactPRReviewLoop` resolves `{ outcome: 'still_awaiting_merge', awaitingState, addressingState }`, the entry stays in `checkpointQueue` completely unmodified and `runLane` is NOT called.
- A test asserting `resumeArtifactPRReviewLoop` is called with `artifactKind: 'decision'` (not `'plan'`) when `entry.planningArtifact.artifactType === 'decision'`, and that no file-read (`readFileSync`/`fs`) of any `decisions/*.md` file occurs anywhere in the resume path — direct proof of AC4 that decision documents use the same PR-merge-gated mechanism as plans, with the old markdown-field mechanism gone.
- A test asserting the AC6-fallback case (entry has `planningArtifact` set but `prUrl === undefined`) still resumes normally without calling `resumeArtifactPRReviewLoop` at all (preserves existing behavior — this path predates ARC-1376 and is unaffected by it).
- A test confirming that a `resumeArtifactPRReviewLoop` rejection (thrown error) is caught and falls through to a normal dequeue-and-resume (`runLane` still called), matching the resiliency convention used elsewhere in this file for `listUnresolvedPRComments` failures.

Add or extend tests in `checkpoints.test.ts` for the extracted `addressPlanningArtifactComments` helper (if exported) or, if kept module-private, verify its effects indirectly via the `executeStep`/`replyToThread`/`resolveThread` mock call assertions already used for `addressPRComments` in the existing `describe('addressPRComments', ...)`-equivalent tests in this file — applied to the `'addressing_comments'` outcome test above (assert `executeStep` was called once with the unresolved comment's body as part of the prompt, and `replyToThread`/`resolveThread` were each called once for that comment).

**Verification:** `run_tests` — all new and existing tests in `checkpoints.test.ts` pass; the removed 409-guard tests no longer exist (confirmed no test asserts `res.status).toBe(409)` anywhere in the file).

### Step 7 — Update `laneRunner.test.ts` for the poller/resumeLaneFromPlanningPR removal, and re-verify `triggerPlanningArtifactPR` tests

**Files:** `packages/server/src/__tests__/laneRunner.test.ts`

**Action:**

Per Step 5's test deletions, confirm the remaining `describe('runLane — planning-artifact PR detection and triggering (ARC-1371)', ...)` tests (plan-artifact detection triggers `createPlanPR`, decision-artifact detection triggers `createDecisionPR`, no-artifact regression guard, non-approach-decision documents not diverted, `createPlanPR` throwing still raises a checkpoint) all continue to pass unmodified — none of these depend on `startPlanningPRPoller` or `resumeLaneFromPlanningPR`, only on `triggerPlanningArtifactPR`'s checkpoint-push-and-pause behavior, which Step 1 preserves. Rename the surviving `describe` block's ARC reference comment if desired for clarity (optional; not required for correctness) but do not change its assertions beyond removing the `startPlanningPRPollerMock` expectations per Step 5.

**Verification:** `run_tests` on `packages/server` — full suite green, with the poller-specific and `resumeLaneFromPlanningPR`-specific tests gone and all other `laneRunner.test.ts` tests (in particular the `checkpoints.test.ts`-adjacent ARC-1367/1368/1369/1370/1377/1379/1380 suites, none of which touch planning-artifact logic) passing unchanged.

### Step 8 — Full verification pass

**Files:** N/A (verification only)

**Action:** Run the full `packages/server` test suite, typecheck, and lint. Confirm zero regressions in unrelated suites (`sidecar.test.ts`, `planningArchiver.test.ts`, `worktreeGit.test.ts`, etc.) and confirm the two files most likely to have residual references — `index.ts` (no longer imports `startPlanningPRPoller`/`resumeLaneFromPlanningPR`) and `checkpointGit.ts` (no longer exports `startPlanningPRPoller`) — compile and import cleanly.

**Verification:** `run_typecheck` returns clean; `run_tests` returns all green; `run_lint` reports no new violations (repo has no lint script configured per ARC-1375's completion summary — confirm this is still the case, or run whatever lint tool is detected).

## Testing & Validation

- `run_typecheck` — confirms all edited files (`checkpointGit.ts`, `checkpoints.ts`, `laneRunner.ts`, `index.ts`) compile cleanly with zero references to removed symbols (`startPlanningPRPoller`, `planningPRPollers`, `resumeLaneFromPlanningPR`).
- `run_tests` — confirms:
  - AC1: a new `checkpoints.test.ts` test proves a planning-artifact resume never returns 409 and instead calls `resumeArtifactPRReviewLoop`, whose `'addressing_comments'` outcome updates `renotifyMessage` without dequeuing.
  - AC2: a new `checkpoints.test.ts` test proves the `'advance'` outcome dequeues the entry and calls `runLane`, with `setStoryLifecycleState` advancing the story to `agent_executing`/phase `code` (or `decision`).
  - AC3: `grep -rn "startPlanningPRPoller\|planningPRPollers" packages/server/src` returns zero matches; the deleted poller test suite no longer exists; no test spies on a poller ever firing.
  - AC4: a new `checkpoints.test.ts` test proves `resumeArtifactPRReviewLoop` is called with `artifactKind: 'decision'` for decision-type entries, and no `readFileSync`/markdown-status parsing occurs anywhere in the resume path; `resumeLaneFromPlanningPR` and its 3 status-parsing tests are gone.
  - Full existing `checkpointGit.test.ts`, `checkpoints.test.ts`, and `laneRunner.test.ts` suites pass with only the described additions/removals — no unrelated test's assertions change.
- `run_lint` — confirms no style violations in the four edited production files and two edited test files.

## Risks & Open Questions

- **`decisions/_template.md` (in this planning repo) still describes the old `status: pending`/`answered` workflow to human reviewers.** This plan does not touch that file (it lives in `pa.aid.runbook-executor`, out of scope per the issue's file list which only names conductor application files). Once this migration ships, a decision document's `status`/`selected_approach` frontmatter fields are no longer read by any code path — they become purely informational for the human reviewer's own bookkeeping. This is a documentation-debt follow-up, not a blocker for this story; flagging it so a future story (or a quick doc fix) can update the template's reviewer instructions to say "merge this PR to resume" instead of "set status: answered".
- **`createDecisionPR`'s PR description text (in `checkpointGit.ts`) still tells the reviewer to "edit status: answered and selected_approach: Option N ... before merging."** This string is data (a PR description), not gating logic, and `createDecisionPR` itself is Out of Scope for this story (only the resume/gate side is in scope, per the Assumptions section). Leaving this instructional text unchanged after this migration would confuse reviewers (it references a mechanism that no longer exists). This plan intentionally does not change it because `checkpointGit.ts`'s PR-creation functions are explicitly Out of Scope, but this is worth flagging to the supervisor as PR-description text that will need a follow-up correction (either in this story's PR itself as a small additional fix, or as a fast-follow) so decision-doc reviewers are not told to do something the code no longer checks for.
- **The `addressPlanningArtifactComments`/`addressPRComments` extraction (Step 4) touches a function used by the existing, working ARC-1304 generic-checkpoint comment flow.** The refactor must be verified not to change `addressPRComments`'s existing behavior for non-planning-artifact checkpoints — Step 6/7's regression tests on the untouched generic path are the safety net here.
- **Cross-lane caution (per runbook Wave 3 gate note):** checkpoint-management and parallel-lane-execution lanes' relevant stories (ARC-1371, ARC-1367–ARC-1370) are already fully checked off (`[x]`) in their respective runbooks as of this plan's writing — no active concurrent work was found touching `checkpointGit.ts`/`laneRunner.ts` in those lanes' runbooks. This plan proceeds without additional coordination, but the executing agent should re-check `git log`/`git status` on `pa.aid.conductor.ts`'s `main` immediately before starting Step 1, in case a same-day merge from another lane landed after this plan was written.
