# ARC-1380 Fix archive-guard silent failure — route to escalation checkpoint - Implementation Plan

**Issue:** `issues/story-lifecycle/ARC-1380-fix-archive-guard-silent-failure.md`
**Completion Summary:** `task-completions/ARC-1380-COMPLETION-SUMMARY.md` (TBD)
**Approach:** Single approach — pattern established. Exploration found two existing precedents in `laneRunner.ts` for the exact "operation fails → push a high-level `CheckpointQueueEntry`, persist state, pause the lane" shape this story needs: the retries-exhausted branch inside the agent-step retry loop in `runLane`, and the unexpected-throw handler inside `triggerPlanningArtifactPR` (used when `createPlanPR`/`createDecisionPR`'s underlying git commands throw). ARC-1374 (merged to `main`) already supplies the exact state-transition primitive this story needs (`setStoryLifecycleState`) with `'awaiting_escalation_checkpoint'` already present in the `StoryLifecycleState` union. No competing designs exist; Phase 1 (Approach) is skipped per the write-implementation-plan skill's rule that it may be skipped when exploration reveals a clear existing pattern to follow, even at HIGH risk tier — matching the precedent set by the ARC-1374 plan.
**Owner:** build agent
**Date:** 2026-07-12

## Scope & Alignment

This plan covers exactly the two In-Scope items from the issue:

1. **`planningArchiver.ts` guard-failure paths** — `archiveStoryArtifacts` currently returns `void` and its two file-presence guards (missing completion summary, missing issue file) only `console.warn` and `return`. This plan changes the function's return type to a discriminated result so callers can distinguish a genuine guard failure from the pre-existing idempotent "already archived" skip and from the pre-existing outer-catch "unexpected error" path — without changing what either guard actually checks for (issue's Out of Scope).
2. **Routing into the escalation-checkpoint queue** — `laneRunner.ts`'s end-of-lane archive block (the `ARC-1370` block at the bottom of `runLane`) is extended to inspect the new result, and on a guard failure: push a high-level `CheckpointQueueEntry` (mirroring the two existing escalation-style checkpoint pushes already in this file), set the story's canonical lifecycle state to `'awaiting_escalation_checkpoint'` via `setStoryLifecycleState` (ARC-1374), persist via `writeSidecar`, and pause the lane instead of leaving it silently `'done'` with a missing archive.

Acceptance criteria mapping:
- **AC1** ("guard check fails (missing required file) → transitions to `awaiting_escalation_checkpoint` instead of silently returning") → Step 1 (result type distinguishes the two file-missing guards from all other outcomes) + Step 3 (`laneRunner.ts` pushes the checkpoint and calls `setStoryLifecycleState(..., 'awaiting_escalation_checkpoint')` only for `status: 'guard-failed'`).
- **AC2** ("supervisor resumes after fixing the issue → archiving operation is re-attempted") → Step 3's checkpoint entry deliberately carries no `prUrl` and no `planningArtifact` field, so the existing generic "no prUrl" resume branch in `POST /api/checkpoints/:stepId/resume` (`routes/checkpoints.ts`) dequeues it and calls `runLane` again with the runbook re-parsed from disk. Because every prior step is already `checked: true` in the markdown, `runLane`'s per-step loop skips straight through to the same end-of-lane archive block, which calls `archiveStoryArtifacts` again — a true retry of the exact same operation, with zero special-case routing added to `checkpoints.ts`. Step 4 adds a test proving this re-attempt behavior at the `laneRunner`/`archiveStoryArtifacts` boundary.

Explicitly NOT covered (per issue's Out of Scope and the AC's literal "missing required file" wording):
- The pre-existing idempotency guard (`destIssuePath` already exists → "already archived, skipping") is a legitimate no-op, not a failure — it is left returning a non-escalating outcome and is not routed to the checkpoint queue.
- The pre-existing outer `catch` block (unexpected errors — e.g. a `readdirSync`/`mkdirSync`/`renameSync`/git-command failure) is left returning a non-escalating outcome. It is a different failure class than "guard check fails (missing required file)" and is explicitly outside this story's In Scope (`planningArchiver.ts guard-failure paths` refers to the two named file-presence guards, matching the issue's Goal examples). Documented as a deliberate scope boundary in Risks & Open Questions, not a gap introduced by this story.
- No change to `routes/checkpoints.ts` — the existing generic resume path already handles this new checkpoint type correctly with zero modification (see AC2 mapping above).
- No UI changes — `ResumeCard.tsx` already renders any `CheckpointQueueEntry` generically by `checkpointLevel`, `stepId`, `runbookPath`, and `lastAgentMessage`; no new component or prop is needed for this checkpoint to display and resume correctly.

## Assumptions & Dependencies

- **Dependency ARC-1374 is satisfied**: `StoryLifecycleState` (including `'awaiting_escalation_checkpoint'`), `SidecarState.storyLifecycle`, and the pure helpers `getStoryLifecycleState`/`setStoryLifecycleState` are already merged to `main` in `pa.aid.conductor.ts` (`packages/server/src/state/types.ts`, `packages/server/src/state/sidecar.ts`). This plan is the first real call site that invokes `setStoryLifecycleState` from execution code — ARC-1374 intentionally left all wiring to later stories.
- `archiveStoryArtifacts` is called from exactly one production call site today: the end-of-lane block in `runLane` (`packages/server/src/runner/laneRunner.ts`). No other caller exists, so widening its return type from `void` to a discriminated union is a safe, fully-controlled change.
- The composite checkpoint `stepId` for this new checkpoint type must be unique and must not collide with any real runbook step ID or with the existing `${step.id}__checkpoint` / `${storyKey}__plan-pr` / `${storyKey}__decision-pr` composite ID conventions. `${arcKey}__archive-guard` follows the existing `${storyKey}__<suffix>` naming convention and does not collide with any of those three existing suffixes.
- `routes/checkpoints.ts`'s resume route requires no changes: it branches on `entry.planningArtifact` (ARC-1371 gate) and `entry.prUrl` (ARC-1304 comment-loop gate); a checkpoint entry with both fields absent falls through to the plain "dequeue → re-parse runbook from disk → `runLane`" path, which is exactly the re-attempt behavior AC2 requires.
- `ResumeCard.tsx`/`CheckpointQueueEntry` (client-side mirror in `packages/ui/src/types/runbook.ts`) require no changes: the new checkpoint entry uses only fields the type and component already support (`stepId`, `runbookPath`, `checkpointLevel`, `hitAt`, `lastAgentMessage`).
- The `'already-archived'` idempotency outcome and the outer-catch `'error'` outcome are non-escalating by design in this story (see Scope & Alignment) — this is a deliberate, documented scope boundary, not an oversight.

## Implementation Steps

### Step 1 — Add `ArchiveOutcome` result type; change `archiveStoryArtifacts` to return it

**Files:** `packages/server/src/git/planningArchiver.ts`

**Action:**
Add a new exported discriminated union type, placed directly above the `archiveStoryArtifacts` JSDoc comment:

- `export type ArchiveOutcome = { status: 'archived' } | { status: 'already-archived' } | { status: 'guard-failed'; reason: 'summary-missing' | 'issue-missing' } | { status: 'error'; message: string };`

Change the `archiveStoryArtifacts` signature from `: void` to `: ArchiveOutcome`, and update its JSDoc to document the four outcomes (mirroring the existing "Guards:" bullet list style already in the comment). Update each existing early-return / end-of-function point to return the matching outcome, **without changing any existing guard condition or `console.warn` message text** (all four existing warn strings — `"summary missing"`, `"issue file missing"`, `"already archived"`, `"push failed"`, `"archiveStoryArtifacts failed"` — stay byte-for-byte identical so the existing `planningArchiver.test.ts` warn-message assertions keep passing unmodified):

- Summary-missing guard (currently `console.warn(...); return;`) → after the existing `console.warn`, `return { status: 'guard-failed', reason: 'summary-missing' };`.
- Issue-missing guard (currently `console.warn(...); return;`) → after the existing `console.warn`, `return { status: 'guard-failed', reason: 'issue-missing' };`.
- Idempotency guard (currently `console.warn(...); return;`) → after the existing `console.warn`, `return { status: 'already-archived' };`.
- End of the try block, after the (already-existing, unmodified) inner `try { execSync('git push', ...) } catch { console.warn(...) }` — add `return { status: 'archived' };` as the final statement of the try block. This is reached whether or not the push succeeded, matching the existing "push failure → warn only" contract.
- Outer `catch (err)` block (currently `console.warn(...)` with no return) → after the existing `console.warn`, add `return { status: 'error', message: String(err) };`.

**Verification:** `run_typecheck` on `packages/server` — no type errors. Manually re-read the diff to confirm none of the five `console.warn` call sites' string arguments changed and none of the existing guard `if` conditions changed.

### Step 2 — Update unit tests for the new return contract

**Files:** `packages/server/src/__tests__/planningArchiver.test.ts`

**Action:**
The ten existing numbered tests under `describe('archiveStoryArtifacts', ...)` assert on side effects (warnings, file moves, `execSyncMock` calls) and do not inspect the return value — they require no changes and must continue to pass unmodified. Add four new tests to the same `describe` block (placed after test 10, before the closing `});`):

- `archiveStoryArtifacts` returns `{ status: 'guard-failed', reason: 'summary-missing' }` when the completion summary is missing (reuse the `setupFixtures(tmpDir, { hasSummary: false })` fixture from test 1).
- `archiveStoryArtifacts` returns `{ status: 'guard-failed', reason: 'issue-missing' }` when the issue file is missing (reuse the `setupFixtures(tmpDir, { hasIssue: false })` fixture from test 2).
- `archiveStoryArtifacts` returns `{ status: 'already-archived' }` when the destination already exists (reuse the `setupFixtures(tmpDir, { prearchived: true })` fixture from test 3).
- `archiveStoryArtifacts` returns `{ status: 'archived' }` when all files are present and archiving completes (reuse the default `setupFixtures(tmpDir)` fixture from test 4).

**Verification:** `run_tests` on `packages/server` — all 10 existing plus 4 new tests in `planningArchiver.test.ts` pass.

### Step 3 — Route `guard-failed` outcomes to the escalation checkpoint in `laneRunner.ts`

**Files:** `packages/server/src/runner/laneRunner.ts`

**Action:**
Add `setStoryLifecycleState` to the existing `import { writeSidecar } from '../state/sidecar.js';` line (`import { setStoryLifecycleState, writeSidecar } from '../state/sidecar.js';`).

Replace the existing ARC-1370 end-of-lane block (the `if (arcKey !== undefined) { ... archiveStoryArtifacts(...); }` block at the very end of `runLane`, after `laneStatusMap.set(runbookPath, 'done');`) so that it captures the return value and, on `status === 'guard-failed'`, performs the escalation routing. The new block's logic, in order:

1. Call `archiveStoryArtifacts(doneState.repoRoot, arcKey, laneFromRunbookPath(runbookPath))` and capture the result (rename the local variable from the discarded call to e.g. `archiveResult`).
2. If `archiveResult.status !== 'guard-failed'`, fall through with no further action (existing behavior for `'archived'`, `'already-archived'`, and `'error'` outcomes is unchanged — lane stays `'done'`).
3. If `archiveResult.status === 'guard-failed'`:
   - Read `getState()` again as `current` (mirroring the exact pattern used by the retries-exhausted branch and by `triggerPlanningArtifactPR`'s unexpected-throw handler).
   - Build a human-readable reason string: `'completion summary file is missing'` when `archiveResult.reason === 'summary-missing'`, else `'issue file is missing'`.
   - Build a `CheckpointQueueEntry` with: `stepId: `${arcKey}__archive-guard`` , `runbookPath`, `checkpointLevel: 'high'`, `hitAt: new Date().toISOString()`, `lastAgentMessage: `Archive failed for ${arcKey}: ${reasonText}. Fix the issue, then click Resume to retry.`` — no `prUrl`, no `planningArtifact` (deliberately, so the generic resume path in `routes/checkpoints.ts` applies unmodified, per AC2 mapping in Scope & Alignment).
   - Call `setStoryLifecycleState(current, arcKey, 'awaiting_escalation_checkpoint')` to get an updated `SidecarState`, then spread the new `checkpointQueue` entry onto it (mirroring the two-step "get updated state from a pure helper, then append to `checkpointQueue`" composition already used nowhere else verbatim in this file, but directly composable from the two independent existing patterns: `setStoryLifecycleState`'s pure-transform signature from ARC-1374, and the `{ ...current, checkpointQueue: [...current.checkpointQueue, entry] }` spread pattern used at every other checkpoint-push site in this file).
   - `setState(updated)`; `writeSidecar(updated.repoRoot, updated)`.
   - `laneStatusMap.set(runbookPath, 'paused');` (overriding the `'done'` set earlier in the function, matching the existing convention where every checkpoint/failure branch in `runLane` overrides an earlier optimistic status).
   - `return;`

**Verification:** `run_typecheck` on `packages/server` — no type errors, in particular confirming `archiveResult.reason` is only accessed when `archiveResult.status === 'guard-failed'` (TypeScript's discriminated-union narrowing must accept this without a cast).

### Step 4 — Test coverage for escalation routing and the resume/re-attempt path

**Files:** `packages/server/src/__tests__/laneRunner.test.ts`

**Action:**
First, update the shared mock default: in the `beforeEach` block, change `archiveStoryArtifactsMock = vi.fn();` to `archiveStoryArtifactsMock = vi.fn().mockReturnValue({ status: 'archived' });` so the five existing tests in `describe('runLane — auto-archive on lane completion (ARC-1370)', ...)` (which call `runLane` and expect `getLaneState(RUNBOOK_PATH)` to be `'done'`) continue to receive a well-typed, non-guard-failed outcome and keep passing unmodified.

Add a new `describe('runLane — archive-guard failure routes to escalation checkpoint (ARC-1380)', ...)` block with these tests:

- Guard failure (`archiveStoryArtifactsMock.mockReturnValueOnce({ status: 'guard-failed', reason: 'summary-missing' })`) on an otherwise-complete lane → `getLaneState(RUNBOOK_PATH)` is `'paused'` (not `'done'`); `getState().checkpointQueue` has exactly one entry with `stepId` `` `${arcKey}__archive-guard` ``, `checkpointLevel: 'high'`, no `prUrl`, no `planningArtifact`, and `lastAgentMessage` containing `'completion summary'`.
- Same as above but `reason: 'issue-missing'` → `lastAgentMessage` contains `'issue file'` instead.
- After a guard-failed run, `getState().storyLifecycle?.[arcKey]?.state` equals `'awaiting_escalation_checkpoint'`.
- `'already-archived'` and `'error'` outcomes do **not** push a checkpoint entry and leave the lane `'done'` (regression coverage proving the deliberate non-escalating scope boundary from Scope & Alignment).
- **Re-attempt / resume simulation (AC2):** run `runLane` once with `archiveStoryArtifactsMock` returning `guard-failed`; assert `paused` and one checkpoint entry as above. Then, without resetting mocks, reconfigure `archiveStoryArtifactsMock` to return `{ status: 'archived' }` and call `runLane` a second time with the same steps array but every step's `checked` set to `true` (simulating the on-disk state after the generic `routes/checkpoints.ts` resume handler re-parses the runbook). Assert `archiveStoryArtifactsMock` was called exactly twice total, the second `runLane` call results in `getLaneState(RUNBOOK_PATH)` `'done'`, and no second checkpoint entry was added to `checkpointQueue` beyond the one already present from the first run (proving the retry reaches the archive block again and succeeds, without needing any special-case resume logic).

**Verification:** `run_tests` on `packages/server` — all new and existing tests in `laneRunner.test.ts` pass.

### Step 5 — Full verification pass

**Files:** N/A (verification only)

**Action:** Run the full `packages/server` test suite, typecheck, and lint to confirm zero regressions in any file that references `archiveStoryArtifacts`, `CheckpointQueueEntry`, or `SidecarState.storyLifecycle` (in particular `laneRunner.test.ts` in its entirety, not just the two describe blocks touched above, plus `checkpoints.test.ts`, which exercises the generic resume path this story relies on without modifying).

**Verification:** `run_typecheck` returns clean; `run_tests` returns all green; `run_lint` reports no new violations in `planningArchiver.ts` or `laneRunner.ts`.

## Testing & Validation

- `run_typecheck` on `packages/server` — confirms `ArchiveOutcome`'s discriminated union narrows correctly at the one call site in `laneRunner.ts`, and that no other file constructs or destructures the old `void` return shape.
- `run_tests` on `packages/server` — confirms:
  - `planningArchiver.test.ts`: all 10 existing tests pass unmodified (proving the guard conditions and warn-message text are untouched, satisfying the issue's Out of Scope); 4 new tests prove the return-value contract for all four outcomes.
  - `laneRunner.test.ts`: all pre-existing tests in the ARC-1370 archive describe block pass unmodified after the mock-default update; new ARC-1380 describe block proves AC1 (guard failure → paused + checkpoint + `awaiting_escalation_checkpoint` lifecycle state) and AC2 (re-attempt on resume reaches the archive block again and succeeds).
  - `checkpoints.test.ts` — unmodified; its existing "no prUrl" generic resume tests are the direct evidence that the new checkpoint type needs zero special-case code in `routes/checkpoints.ts`.
- `run_lint` — confirms no style violations in the two edited source files.
- Manual/integration confidence (not automated in this story, noted for completeness): `ResumeCard.tsx` requires no code change because it already renders `lastAgentMessage`, `checkpointLevel`, and the generic `[Resume]` button for any `CheckpointQueueEntry` shape.

## Risks & Open Questions

- **Deliberate scope boundary — outer `catch` (unexpected errors) is not escalated.** The issue's AC and Goal both specifically name "guard check fails (missing required file)" as the trigger condition. A `readdirSync`/`mkdirSync`/`renameSync`/git-command failure inside the try block (caught by the outer `catch`) is a different failure class (infrastructure/IO failure vs. a precondition check) and is left with its pre-existing "warn only, never throws" behavior. If a future story wants uniform escalation for *all* `archiveStoryArtifacts` failure modes, it can extend the `laneRunner.ts` branch to also match `status === 'error'` — the `ArchiveOutcome` type already carries a `message` field ready for that extension, so no further type change would be needed.
- **Deliberate scope boundary — idempotent "already archived" skip is not escalated.** This is existing, correct, intentional behavior (resuming a lane that already fully archived should be a silent no-op, not a checkpoint) and is preserved unchanged.
- **First real caller of `setStoryLifecycleState`.** ARC-1374 added the type/persistence plumbing but wired no call sites. This story is the first to call it from `laneRunner.ts`. No other code reads `getStoryLifecycleState` yet (ARC-1381's rollup-derivation story is what will eventually consume it for `LaneStatus`/`RunbookStatus` computation), so this story's state write is inert with respect to any other currently-shipped behavior — it only becomes observable once a future story reads it back. This is expected, matching the incremental-wiring approach ARC-1374 documented.
- **Checkpoint composite `stepId` is synthetic, not a real runbook step.** `${arcKey}__archive-guard` does not correspond to any `RunbookStep.id` produced by `parseRunbook`. This matches the existing precedent of `${step.id}__checkpoint` (sub-step gate) and `${storyKey}__plan-pr`/`${storyKey}__decision-pr` (ARC-1371 planning-artifact gates), none of which correspond to real parsed steps either — the `checkpointQueue` already supports and displays synthetic composite IDs generically via `ResumeCard`/`deriveRunbookSummary`'s `runbookPath`-based matching, not `stepId`-based matching against real steps.
