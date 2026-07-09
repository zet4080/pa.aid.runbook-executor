# ARC-1371 Implementation Plan: Conductor — Open PR for Implementation Plans and Decision Documents as Approval Gate

**Issue:** `issues/checkpoint-management/ARC-1371-conductor-pr-approval-gate-for-plans-and-decisions.md`
**Completion Summary:** `task-completions/ARC-1371-COMPLETION-SUMMARY.md` (TBD)
**Approach:** single approach — pattern established (extend `checkpointGit.ts`, no new module)
**Lane:** checkpoint-management
**Epic:** ARC-1295 — ARC: Checkpoint Management
**Owner:** build agent
**Date:** 2026-07-09 (rewritten against merged ARC-1367–ARC-1370 code; originally written 2026-07-07)

## Rewrite Note

This plan was originally written speculatively, before ARC-1367, ARC-1368, ARC-1369, and ARC-1370 merged into `main`. It has been rewritten from a fresh read of the actual merged code in `packages/server/src/git/checkpointGit.ts`, `packages/server/src/runner/laneRunner.ts`, `packages/server/src/routes/checkpoints.ts`, `packages/server/src/state/types.ts`, and `packages/server/src/index.ts` at commit `03efc9b` (post-ARC-1370 merge). Every function signature, call site, and control-flow reference below is verified against that code, not assumed.

## Scope & Alignment

This plan extends `checkpointGit.ts` with plan/decision PR-opening and PR-merge-detection functions, and rewires the two `laneRunner.ts` call sites that currently commit planning artifacts and completion artifacts directly to `main` (`commitPlanningArtifacts`, wired at the successful-agent-step tail; `archiveStoryArtifacts`, wired at lane completion) so that **implementation plans and decision documents** go through a branch+PR approval gate instead. Completion summaries and story archival (ARC-1370's `archiveStoryArtifacts`) are explicitly out of scope per the issue and continue to commit straight to `main`.

Acceptance criteria mapping:

| AC | Addressed by |
|----|--------------|
| AC1 — plan PR: branch `plan/{lane}/{KEY}`, PR titled `Plan review: {KEY} — {title}`, pause at 🔴 | Steps 2, 3, 6 |
| AC2 — decision PR: branch `decision/{KEY}-{type}`, PR titled `Decision: {KEY} — {type}`, pause at 🔴 | Steps 2, 4, 6 |
| AC3 — merge detected, lane resumes; decision merge reads `selected_approach` from `main` | Steps 5, 7 |
| AC4 — plan PR merge → plan file now on `main`, agent executes it | Step 7 (poller triggers `runLane`, which re-parses the runbook and re-dispatches the agent step — the merged plan file is on disk for the agent to read) |
| AC5 — decision PR merge → read `status`; `answered` continues, `pending` re-pauses with warning | Step 7 |
| AC6 — PR creation failure → fallback commit to `main`, checkpoint raised, supervisor notified via queue | Steps 2, 3, 4 |
| AC7 — `write-implementation-plan` / `start-execution-session` skills no longer instruct manual commit / `status: answered` editing | Steps 9, 10 |

## Assumptions & Dependencies

| # | Assumption | Verified against |
|---|------------|-------------------|
| 1 | `commitPlanningArtifacts(repoRoot: string, lane: string): void` exists in `checkpointGit.ts`, is synchronous (not async), and is called once — after `markStepCheckedInMarkdown` — in the success tail of the `type === 'agent'` branch of `runLane`, only when no pending CHECKPOINT/GATE sub-step was found | Read `checkpointGit.ts` lines ~357–399 and `laneRunner.ts` lines ~456–461 |
| 2 | `archiveStoryArtifacts(repoRoot: string, storyKey: string, lane: string): void` exists in `packages/server/src/git/planningArchiver.ts` (its own module, not `checkpointGit.ts`), is synchronous, and is called once at the very end of `runLane` after `laneStatusMap.set(runbookPath, 'done')`, gated on deriving an `ARC-\d+` key from the completed steps | Read `planningArchiver.ts` lines ~150–243 and `laneRunner.ts` lines ~509–518 |
| 3 | `createCheckpointBranchAndPR`, `listUnresolvedPRComments`, `replyToThread`, `resolveThread`, `fetchCommitSha` are the only currently-exported PR/git functions from `checkpointGit.ts`; the private helpers `parseRemoteUrl`, `getBitbucketCredentials`, `parsePrId`, `parsePrUrlRepo`, `createBitbucketPR` are unexported and reusable in-module only | Read `checkpointGit.ts` in full (428 lines) |
| 4 | There is **no existing PR-merge-polling mechanism** anywhere in the codebase. ARC-1366 (`get_checkpoint_context` / `send_checkpoint_notification`) is an OpenCode *tool* pair for supervisor-instruction handoff at a checkpoint — it does not poll Bitbucket and has no `resume_checkpoint` endpoint or PR-state polling. The route `POST /api/checkpoints/:stepId/resume` (in `routes/checkpoints.ts`) is a manual, supervisor-triggered HTTP call; it addresses unresolved PR comments (ARC-1304 `listUnresolvedPRComments`/`addressPRComments`) before dequeuing and calling `runLane`, but never checks whether a PR is merged. **This plan must build the PR-merge poller from scratch** — there is nothing to "reuse" beyond the existing Bitbucket REST call pattern in `checkpointGit.ts`. | Read `routes/checkpoints.ts` in full (220 lines); searched for `resume_checkpoint`, `isPRMerged`, `MERGED`, poll* across `packages/server/src` — zero matches; read `issues/agent-tools/ARC-1366-tool-resume-checkpoint.md` in full |
| 5 | `SessionConfig` (in `state/types.ts`) currently has exactly four fields: `maxConcurrentLanes`, `retryLimit`, `queuePolicy`, `escalationSequence`. No `prPollIntervalMs` or any PR-related field exists yet. `PATCH /api/state` already supports partial `config` merges via `routes/state.ts`, so adding a new optional field requires no route change. | Read `state/types.ts` lines 1–24 and `routes/state.ts` in full |
| 6 | `CheckpointQueueEntry` (in `state/types.ts`) currently has: `stepId`, `runbookPath`, `checkpointLevel`, `hitAt`, `prUrl?`, `renotifyMessage?`, `lastAgentMessage?`. No `planningArtifact` discriminator exists yet. | Read `state/types.ts` lines 26–46 |
| 7 | `laneFromRunbookPath`, `isNonExecutableSubStep`, `checkpointLevelFromSubStepLabel`, `isHighStory`, `extractLastAgentMessage` are private (unexported) functions local to `laneRunner.ts`. `laneStatusMap`, `laneActiveStep`, `pushedSubStepCheckpoints` are private module-level `Map`/`Set` state. | Read `laneRunner.ts` in full (519 lines) |
| 8 | The agent step's **only** signal that a plan or decision file was written is the markdown runbook step label plus the fact that the agent, per skill instructions, writes the file to a fixed conventional path (`implementation_plans/{lane}/{KEY}-implementation-plan.md` or `decisions/{KEY}-approach-decision.md`). There is **no structured "artifact written" event type** in `runner/types.ts` (`OpenCodeEvent` is `StepStartEvent \| ToolUseEvent \| TextEvent \| StepFinishEvent \| ErrorEvent \| UnknownEvent` — no artifact-specific variant). Detection must be done via `git status --porcelain` on the conventional directories, exactly as `commitPlanningArtifacts` already does. | Read `runner/types.ts` in full; read `commitPlanningArtifacts` implementation |
| 9 | ARC-1366 is a **planning-repo tool pair**, not an application-server feature. It has no bearing on `pa.aid.conductor.ts` server code and is not a dependency this plan needs to call into. The issue's "Dependencies" line referencing ARC-1366 is about *conceptual* precedent (checkpoint + supervisor instruction), not a shared function. | Read `issues/agent-tools/ARC-1366-tool-resume-checkpoint.md` in full |
| 10 | Decision documents live in `decisions/{KEY}-approach-decision.md` in the planning repo (per `decisions/_template.md`), with frontmatter fields `key`, `story`, `status` (`pending`/`answered`), `selected_approach`, `decided_by`, `decided_at`. The `write-implementation-plan` skill's Phase 1 (`decisions/_template.md`) and `start-execution-session` skill both currently instruct manual commit-and-push of the decision file and manual `status: answered` editing by the human. | Read `decisions/_template.md`; read both skill files (`~/.config/opencode/skills/write-implementation-plan/SKILL.md`, `~/.config/opencode/skills/start-execution-session/SKILL.md`) |
| 11 | `BITBUCKET_USERNAME`, `BITBUCKET_PASSWORD`, optionally `BITBUCKET_WORKSPACE`, `BITBUCKET_DEFAULT_BRANCH` env vars are loaded via `dotenv.config()` in `index.ts` from the monorepo root `.env` | Read `index.ts` lines 30–38; existing `getBitbucketCredentials()` pattern in `checkpointGit.ts` |
| 12 | Branch must be created in the planning repo (`pa.aid.runbook-executor`), reached via `getState().repoRoot` — the same `repoRoot` value `commitPlanningArtifacts` and `archiveStoryArtifacts` already use — not `pa.aid.conductor.ts` (constraint from the issue) | Constraint restated from issue; `repoRoot` usage confirmed in `laneRunner.ts` |

## Architecture Decision

**Decision:** Extend `checkpointGit.ts` with two new exported functions — `createPlanPR` (branch `plan/{lane}/{KEY}`) and `createDecisionPR` (branch `decision/{KEY}-{type}`) — following the exact structural pattern of the existing `createCheckpointBranchAndPR`: create/reuse branch → commit → push (skip if already on remote) → return to previous ref → fire-and-forget Bitbucket POST → return `{ branch, prUrl? }`. Do **not** introduce a separate `planningArtifactGit.ts` module — this satisfies the issue constraint "must reuse the existing `checkpointGit.ts` PR-creation logic" and keeps the private helpers (`getBitbucketCredentials`, `parseRemoteUrl`, `parsePrId`, `parsePrUrlRepo`) co-located and unexported.

**Decision:** No polling mechanism exists to reuse (see Assumption 4). A new exported function `isPRMerged(prUrl: string): Promise<boolean>` and a new `startPlanningPRPoller(opts): { stop: () => void }` are added to `checkpointGit.ts`, following the same private-helper-reuse pattern (`parsePrId`, `parsePrUrlRepo`, `getBitbucketCredentials`) as the existing `listUnresolvedPRComments`.

**Decision:** `commitPlanningArtifacts` is **not removed**. It remains the auto-commit path for any planning-artifact directory changes that are *not* plans or decisions (defensive: if a future artifact type lands in `implementation_plans/` or `task-completions/` without going through the new PR path, it still gets committed). The new logic in `laneRunner.ts` intercepts **only** plan and decision files *before* `commitPlanningArtifacts` runs, diverting them to the PR path; anything else `commitPlanningArtifacts` picks up (e.g. `task-completions/`) is untouched, matching the issue's explicit "Out of Scope: completion summaries continue to be committed directly to main (ARC-1369)."

**Decision:** `archiveStoryArtifacts` (ARC-1370) is untouched. It runs at lane completion regardless of this story — plan/decision PR merges happen mid-lane, well before `archiveStoryArtifacts` fires.

## Implementation Steps

### Step 1 — Add `prPollIntervalMs` to `SessionConfig`

**File:** `packages/server/src/state/types.ts`

Add an optional field `prPollIntervalMs?: number` to the `SessionConfig` interface (alongside `maxConcurrentLanes`, `retryLimit`, `queuePolicy`, `escalationSequence`), documented as the polling interval in milliseconds for Bitbucket PR-merge checks. Add `prPollIntervalMs: 30000` to `DEFAULT_SESSION_CONFIG`. No changes to `routes/state.ts` are needed — its `PATCH /api/state` handler already merges any `body.config` fields into `current.config` verbatim.

**Verification:** `DEFAULT_SESSION_CONFIG.prPollIntervalMs === 30000`; existing `reconcile.test.ts` fixtures that construct `SessionConfig` literals without the new field still typecheck (field is optional).

### Step 2 — Add planning-artifact PR types to `checkpointGit.ts`

**File:** `packages/server/src/git/checkpointGit.ts`

Add, near `CreateCheckpointBranchAndPRResult`:
- `CreatePlanPROptions { repoRoot: string; lane: string; storyKey: string; storyTitle: string; artifactPath: string }`
- `CreateDecisionPROptions { repoRoot: string; storyKey: string; decisionType: string; artifactPath: string }`
- `CreatePlanningArtifactPRResult { branch: string; prUrl?: string }`

**Verification:** `npx tsc --noEmit` in `packages/server` reports no new errors.

### Step 3 — Implement `createPlanPR` in `checkpointGit.ts`

**File:** `packages/server/src/git/checkpointGit.ts`

Add `export async function createPlanPR(opts: CreatePlanPROptions): Promise<CreatePlanningArtifactPRResult>`, mirroring `createCheckpointBranchAndPR`'s git sequence but scoped to a single artifact file instead of the whole runbook-checkoff diff:
1. Branch name: `` `plan/${lane}/${storyKey}` ``.
2. Create or reuse the branch (same `rev-parse --verify` / `checkout -b` / `checkout` pattern as `createCheckpointBranchAndPR`).
3. `git add` the single `artifactPath` (relative to `repoRoot`), commit with message `` `docs(${lane}): add ${storyKey} implementation plan for review` ``.
4. Push only if not already on remote (same `ls-remote --heads` guard).
5. Return to the previous ref (`git checkout -`, matching `createCheckpointBranchAndPR`'s pattern rather than a hardcoded `git checkout main` — this is safer if the caller was on a non-`main` ref, and avoids the detached-HEAD risk called out in the risk table).
6. Attempt Bitbucket PR creation using the existing private helpers `getBitbucketCredentials`, `parseRemoteUrl` (via `git -C {repoRoot} remote get-url origin`), reading `BITBUCKET_WORKSPACE`/`BITBUCKET_DEFAULT_BRANCH` overrides exactly as `createBitbucketPR` does. PR title: `` `Plan review: ${storyKey} — ${storyTitle}` ``. PR description includes artifact path, lane, story key, and a one-line reviewer instruction. On any failure, catch, `console.error` with an `[ARC-1371]` prefix, and return `{ branch, prUrl: undefined }` — never throw.
7. **Fallback commit to `main` on PR failure (AC6):** if the Bitbucket POST fails, the artifact is already committed on the new branch (step 3), not on `main`. To satisfy AC6's "artifact is committed directly to main as a fallback," add an explicit fallback path: after the PR-creation `catch` block, if `prUrl` is `undefined`, cherry-pick or re-apply the same commit onto `main` (`git checkout main && git cherry-pick {branch}` or simpler: re-run the add+commit sequence against `main` directly, since the working tree still has the file). Given the branch is disposable in this failure case, the simplest safe approach is: checkout `main`, `git add`/`git commit` the same artifact file with the same message, push. Document this explicitly in code comments so a future reader does not mistake the branch-commit for the fallback.

**Verification:** unit test asserts branch naming, PR title/body content via the mocked `fetch` call, and that on `fetch` failure the artifact ends up committed on `main` (assert `execSync` was called with `git checkout main` and a `git commit` after the PR attempt failed).

### Step 4 — Implement `createDecisionPR` in `checkpointGit.ts`

**File:** `packages/server/src/git/checkpointGit.ts`

Add `export async function createDecisionPR(opts: CreateDecisionPROptions): Promise<CreatePlanningArtifactPRResult>`, structurally identical to `createPlanPR` (Step 3) but:
- Branch name: `` `decision/${storyKey}-${decisionType}` ``.
- Commit message: `` `docs: add ${storyKey} ${decisionType} decision document for review` ``.
- PR title: `` `Decision: ${storyKey} — ${decisionType}` ``.
- PR description instructs the supervisor to edit `status: answered` and `selected_approach: Option N` in the file (in the PR's diff or by pushing an additional commit to the PR branch) before merging.
- Same fallback-to-`main` behavior as Step 3 on PR-creation failure.

**Verification:** unit test asserts branch naming (`decision/ARC-1371-approach-decision`), PR title, and fallback-to-main path.

### Step 5 — Add `isPRMerged` and `startPlanningPRPoller` to `checkpointGit.ts`

**File:** `packages/server/src/git/checkpointGit.ts`

Since no polling mechanism exists anywhere in the codebase (Assumption 4), add both from scratch:

- `export async function isPRMerged(prUrl: string): Promise<boolean>` — reuses the private `parsePrId` and `parsePrUrlRepo` helpers (already used by `listUnresolvedPRComments`/`replyToThread`/`resolveThread`) and `getBitbucketCredentials`. Issues a `GET` to `https://api.bitbucket.org/2.0/repositories/{workspace}/{repoSlug}/pullrequests/{prId}`, returns `true` iff the response body's `state === 'MERGED'`. Throws on non-2xx or network error — caller must catch.
- `export function startPlanningPRPoller(opts: { prUrl: string; intervalMs: number; onMerged: () => void }): { stop: () => void }` — a `setTimeout`-based loop (not `setInterval`, to avoid overlapping in-flight checks) that calls `isPRMerged` every `intervalMs`, invokes `onMerged` exactly once and stops on the first `true`, logs and continues on individual poll errors, and exposes a `stop()` closure to cancel.

**Verification:** unit tests — `isPRMerged` returns true/false correctly per mocked `fetch` response and throws on non-2xx; `startPlanningPRPoller` calls `onMerged` once when merged, never calls it after `stop()`, and survives one thrown poll error before succeeding on a later tick (use `vi.useFakeTimers()` and advance timers, consistent with existing test patterns in `checkpointGit.test.ts`).

### Step 6 — Wire plan/decision-artifact detection and PR triggering into `laneRunner.ts`

**File:** `packages/server/src/runner/laneRunner.ts`

**Current control flow being modified** (verified in the actual file): inside the `for (const step of steps)` loop, `type === 'agent'` branch, after `executeStep` succeeds and the `pendingGate` (CHECKPOINT/GATE sub-step) check finds no pending gate, the code calls `markStepCheckedInMarkdown(runbookPath, step.id)` and then unconditionally `commitPlanningArtifacts(stepState.repoRoot, laneFromRunbookPath(runbookPath))` before falling through to the next loop iteration.

**New logic**, inserted immediately before the existing `commitPlanningArtifacts` call (i.e. still inside the "no pending gate" branch, after `markStepCheckedInMarkdown`):

1. Run `git status --porcelain -- implementation_plans/ decisions/` against `stepState.repoRoot` (mirroring the exact `git status --porcelain` pattern `commitPlanningArtifacts` already uses, but scoped to `implementation_plans/` and `decisions/` instead of `implementation_plans/` + `task-completions/`).
2. If the status output contains any line under `implementation_plans/`, treat it as a plan artifact: derive `storyKey` from the changed filename (matches the `{KEY}-implementation-plan.md` convention) and `lane` from `laneFromRunbookPath(runbookPath)`; derive `storyTitle` from the step's `label` (strip any leading `ARC-\d+ — ` prefix or ARC key match already extracted by `buildPromptFromStep`'s `arcKeyMatch` logic — reuse that same regex).
3. If the status output contains any line under `decisions/` matching `{KEY}-approach-decision.md`, treat it as a decision artifact: derive `storyKey` and `decisionType` (`approach-decision`, the only decision type this codebase's `_template.md` currently produces) from the filename.
4. For each detected artifact, call a new local async helper `triggerPlanningArtifactPR(...)` (Step 7) instead of proceeding to `commitPlanningArtifacts` for that file. If **any** plan or decision artifact was detected and dispatched, `return` from `runLane` immediately after `triggerPlanningArtifactPR` resolves (the lane is now paused; do not fall through to `commitPlanningArtifacts` or the next loop iteration).
5. If no plan/decision artifact is detected, fall through unchanged to the existing `commitPlanningArtifacts(stepState.repoRoot, laneFromRunbookPath(runbookPath))` call — this preserves the existing behavior for completion summaries and any other `task-completions/`/`implementation_plans/` changes that are not plan/decision files (though in practice a step either writes a plan or doesn't; this fallthrough mainly protects the completion-summary and archived-plan-move scenarios already covered by ARC-1369/ARC-1370).

**Why detect via `git status` rather than a new event type:** confirmed in Assumption 8 — `OpenCodeEvent` has no artifact-write signal, and `commitPlanningArtifacts` already establishes the git-status-scan pattern as this codebase's convention for artifact detection.

**Verification:** unit test in `laneRunner.test.ts` mocks `execSync`-backed git status (via the same `checkpointGit.js` module mock already used for `commitPlanningArtifacts`) to report a changed `implementation_plans/checkpoint-management/ARC-9999-implementation-plan.md` file, and asserts `triggerPlanningArtifactPR` (mocked) is called with `artifactType: 'plan'` and that `commitPlanningArtifacts` is NOT called for that step; a second test asserts the decision-file path.

### Step 7 — Implement `triggerPlanningArtifactPR` and `resumeLaneFromPlanningPR` in `laneRunner.ts`

**File:** `packages/server/src/runner/laneRunner.ts`

**7a. Imports** — extend the existing `checkpointGit.js` import (currently `{ createCheckpointBranchAndPR, commitPlanningArtifacts }`) to add `createPlanPR, createDecisionPR, startPlanningPRPoller`.

**7b. Module-level poller registry** — add `const planningPRPollers = new Map<string, { stop: () => void }>()`, following the existing pattern of `laneStatusMap`/`laneActiveStep`/`pushedSubStepCheckpoints` as private module state; reset it in `_resetLaneStateForTesting()` alongside the other three.

**7c. `triggerPlanningArtifactPR`** — signature: `async function triggerPlanningArtifactPR(opts: { runbookPath: string; artifactType: 'plan' | 'decision'; lane: string; storyKey: string; storyTitle?: string; decisionType?: string; artifactPath: string; repoRoot: string }): Promise<void>`. Behavior:
1. Calls `createPlanPR` or `createDecisionPR` based on `artifactType`.
2. Constructs a `CheckpointQueueEntry` with `checkpointLevel: 'high'`, `hitAt`, `prUrl: result.prUrl`, and the new `planningArtifact: { artifactType, storyKey, branch: result.branch }` field (Step 8), pushes it into `getState().checkpointQueue`, persists via `setState`/`writeSidecar`, exactly matching the existing push pattern used for sub-step and top-level checkpoints elsewhere in `runLane`.
3. Sets `laneStatusMap.set(runbookPath, 'paused')`.
4. If `result.prUrl` is defined, starts a poller via `startPlanningPRPoller({ prUrl, intervalMs: getState().config.prPollIntervalMs ?? 30000, onMerged: () => { planningPRPollers.delete(result.prUrl!); resumeLaneFromPlanningPR(...) } })` and stores it in `planningPRPollers`.
5. If `result.prUrl` is undefined (PR creation failed and the artifact was committed to `main` as a fallback per Step 3/4), log a warning; the checkpoint entry stays in the queue with no `prUrl`, and per the existing `routes/checkpoints.ts` resume route, a supervisor can resume it manually via `POST /api/checkpoints/:stepId/resume` (that route's no-`prUrl` branch already dequeues and calls `runLane` — no route change needed for this fallback case).

**7d. `resumeLaneFromPlanningPR`** — signature: `export function resumeLaneFromPlanningPR(compositeId: string, runbookPath: string, artifactType: 'plan' | 'decision', storyKey: string): void` (exported, not module-private, so `index.ts` startup recovery in Step 8 can call it). Behavior:
1. Dequeue the matching `checkpointQueue` entry by `stepId === compositeId`, persist via `setState`/`writeSidecar` — same filter-by-stepId pattern already used in `routes/checkpoints.ts`'s `resumeLane` helper (not `runLane` itself) to avoid stale-index bugs.
2. If `artifactType === 'decision'`: no special extraction is done here — the merged decision file is on `main` at this point; the next agent invocation (triggered by `runLane` re-dispatching the same step, since `step.checked` is still `false` for a plan/decision-writing step that hasn't reached its `markStepCheckedInMarkdown` call) will re-read the file from disk via its prompt, which already references `decisions/{KEY}-approach-decision.md` by convention. Log an info message noting the merge.

   **Correction to AC5's re-pause requirement:** AC5 requires that if `status` is still `pending` after merge (supervisor merged without answering), the conductor re-pauses and logs a warning rather than proceeding. Implement this by reading the merged file (`readFileSync` on the resolved `decisions/{KEY}-approach-decision.md` path under `repoRoot`) and checking for the literal line `status: answered`. If absent, re-push a `CheckpointQueueEntry` with the same `planningArtifact` metadata (no new PR — the existing merged PR already serves as the review surface) and `laneStatusMap.set(runbookPath, 'paused')`, then `return` without calling `runLane`. If present, proceed to step 3 below.
3. Otherwise (plan artifact, or decision confirmed answered): re-parse the runbook via the existing `parseRunbook` import — this must be a **static top-level import** (`import { parseRunbook } from '../runbook/parser.js'`) added to `laneRunner.ts`'s import block, not a `require()` call, since `laneRunner.ts` is a pure ESM module and no other function in this file uses `require`. Then call `void runLane(runbookPath, steps).catch(...)`, matching the exact resume pattern already used in `routes/checkpoints.ts`'s `resumeLane` helper.

**Verification:** unit tests mock `createPlanPR`/`createDecisionPR`/`startPlanningPRPoller` and assert: lane pauses and checkpoint entry is pushed with `planningArtifact` set; poller is started with the configured `intervalMs`; fallback (`prUrl` undefined) path pauses without starting a poller; `resumeLaneFromPlanningPR` dequeues the entry and calls `runLane`; decision resume with `status: pending` re-pauses without calling `runLane`.

### Step 8 — Add `planningArtifact` discriminator to `CheckpointQueueEntry`

**File:** `packages/server/src/state/types.ts`

Add an optional field to `CheckpointQueueEntry` (alongside `prUrl?`, `renotifyMessage?`, `lastAgentMessage?`):

```typescript
planningArtifact?: { artifactType: 'plan' | 'decision'; storyKey: string; branch: string };
```

This is purely additive — existing entries without the field remain valid, and no other reader of `CheckpointQueueEntry` needs to change (the UI's `ResumeCard`/`CheckpointQueuePanel` components read `prUrl`, `renotifyMessage`, `lastAgentMessage` today and can be extended in a follow-up UI story if display polish is wanted; this plan's issue does not require a UI change).

**Verification:** `npx tsc --noEmit` passes; existing `CheckpointQueueEntry` literals in tests that omit the field still typecheck.

### Step 9 — Guard the resume route against manual resume of planning-PR checkpoints

**File:** `packages/server/src/routes/checkpoints.ts`

The current `POST /api/checkpoints/:stepId/resume` handler (verified: lines 123–186) branches on `entry.prUrl` truthy/falsy to decide whether to run the ARC-1304 comment-address flow before dequeuing. For a planning-artifact checkpoint (`entry.planningArtifact` set) **with** a `prUrl`, that comment-address flow is not the intended resume path — PR merge (via the Step 5/7 poller) is. Add a guard as the **first check** inside the try block, immediately after resolving `entry` (before the existing `if (entry.prUrl)` branch): if `entry.planningArtifact !== undefined && entry.prUrl !== undefined`, respond `409` with `{ error: '...merge the PR to resume automatically...', prUrl: entry.prUrl }` and `return` without touching the queue. If `entry.planningArtifact` is set but `entry.prUrl` is undefined (the AC6 fallback-to-`main` case), fall through to the existing no-`prUrl` branch unchanged — that path already dequeues and calls `runLane`, which is the correct manual-resume behavior for the fallback case.

**Verification:** unit test in `checkpoints.test.ts` — entry with both `planningArtifact` and `prUrl` set returns 409 with `prUrl` in the body and does not call `runLane`; entry with `planningArtifact` set but no `prUrl` resumes normally (existing no-`prUrl` path).

### Step 10 — Startup recovery for in-flight planning-PR pollers

**File:** `packages/server/src/index.ts`

Pollers live only in `laneRunner.ts`'s in-memory `planningPRPollers` map (Step 7b) and are lost on server restart. The current startup sequence (verified: lines 30–111) already loads `initialState` via `loadOrCreateState`, then reconstructs `laneStatusMap` from `initialState.checkpointQueue` via `restoreLaneStatus`. Add, immediately after that existing `if (initialState.sessionStartedAt) { ... }` block: iterate `initialState.checkpointQueue`, and for each entry where `entry.planningArtifact !== undefined && entry.prUrl !== undefined`, call `startPlanningPRPoller({ prUrl: entry.prUrl, intervalMs: initialState.config.prPollIntervalMs ?? 30000, onMerged: () => resumeLaneFromPlanningPR(entry.stepId, entry.runbookPath, entry.planningArtifact!.artifactType, entry.planningArtifact!.storyKey) })`. This requires importing `startPlanningPRPoller` from `../git/checkpointGit.js` and `resumeLaneFromPlanningPR` from `../runner/laneRunner.js` (already exported per Step 7d) into `index.ts`, alongside the existing `restoreLaneStatus` import.

**Verification:** integration-style test (or manual smoke test) — seed a sidecar file with a `checkpointQueue` entry containing `planningArtifact` and `prUrl`, start the server, mock `isPRMerged` to resolve `true`, confirm `resumeLaneFromPlanningPR` fires and the lane transitions out of `paused`.

### Step 11 — Update `write-implementation-plan` skill

**File:** `~/.config/opencode/skills/write-implementation-plan/SKILL.md` (and mirrored copy at `opencode-config/skills/write-implementation-plan/SKILL.md` in this planning repo, if present — verify at execution time and update both if the mirror exists)

Remove the instruction (already partially updated by a prior story — verify current wording at execution time) that implies the agent commits the plan file manually. Replace with: the conductor's `runLane` now detects the new plan file via git status, opens a PR, and pauses the lane automatically (Steps 3, 6, 7 of this plan) — the agent's step is complete once the file is written to `implementation_plans/{lane}/{KEY}-implementation-plan.md`; no manual `git add`/`commit`/`push` and no manual stop-and-wait phrasing is needed, since the pause now happens inside `runLane` itself, not via agent-authored instructions.

**Verification:** read the file after edit and confirm no remaining instruction tells the agent to manually commit/push the plan file or the decision document.

### Step 12 — Update `start-execution-session` skill

**File:** `~/.config/opencode/skills/start-execution-session/SKILL.md` (and mirrored copy at `opencode-config/skills/start-execution-session/SKILL.md` in this planning repo, if present)

Remove instructions telling the agent to commit the decision document manually or to poll/wait for `status: answered` to be edited directly on `main`. Replace with: the conductor opens a PR for the decision document automatically (Step 4, 6, 7); the supervisor edits `status: answered` and `selected_approach: Option N` in the file within the PR (either via the Bitbucket web editor on the PR branch, or by pushing a commit to the PR branch) and merges; the conductor detects the merge (Step 5 poller) and resumes the lane, re-reading the merged file from `main` (Step 7d).

**Verification:** read the file after edit and confirm no remaining instruction tells the human/agent to edit `status: answered` directly on `main` outside a PR, or to manually push the decision file.

## Testing & Validation

### Unit — `packages/server/src/git/checkpointGit.test.ts`

- `createPlanPR` — branch naming (`plan/{lane}/{storyKey}`), PR title/body content, commit-on-branch before push, fallback-to-`main` commit when `fetch` fails.
- `createDecisionPR` — branch naming (`decision/{storyKey}-{decisionType}`), PR title/body content, fallback-to-`main` commit.
- `isPRMerged` — `true` when `state === 'MERGED'`, `false` for `OPEN`/other, throws on non-2xx.
- `startPlanningPRPoller` — calls `onMerged` once on first `true` tick; `stop()` prevents further calls; survives a thrown poll error and succeeds on a later tick (fake timers).

### Unit — `packages/server/src/__tests__/laneRunner.test.ts`

- Plan-artifact detection after successful agent step triggers `triggerPlanningArtifactPR` with `artifactType: 'plan'`, not `commitPlanningArtifacts`.
- Decision-artifact detection triggers `artifactType: 'decision'`.
- No plan/decision artifact present → existing `commitPlanningArtifacts` call is unchanged (regression guard).
- `triggerPlanningArtifactPR` pauses the lane, pushes a `CheckpointQueueEntry` with `planningArtifact` set, and starts a poller when `prUrl` is present.
- `triggerPlanningArtifactPR` fallback path (no `prUrl`) pauses without starting a poller and logs a warning.
- `resumeLaneFromPlanningPR` dequeues by `stepId` and calls `runLane`.
- `resumeLaneFromPlanningPR` for a decision artifact with `status: pending` in the merged file re-pauses without calling `runLane`.

### Unit — `packages/server/src/__tests__/checkpoints.test.ts`

- `POST /checkpoints/:stepId/resume` for an entry with both `planningArtifact` and `prUrl` returns 409 with `prUrl` in the body.
- Same route for an entry with `planningArtifact` set but no `prUrl` resumes normally (existing fallback path, unchanged).

### Integration smoke test

1. Start the server against a test planning repo with a pushed decision-document branch and an open PR recorded in `checkpointQueue` (`planningArtifact` + `prUrl` set).
2. `GET /api/state` shows the entry with `planningArtifact` populated.
3. Mock `isPRMerged` to resolve `true`.
4. Confirm the lane's status (via `GET /api/state`'s `laneState`) transitions from `paused` to `running`/`done` once the poller fires.

## Risks & Open Questions

| Risk | Likelihood | Mitigation |
|------|-----------|------------|
| Git-status-based artifact detection (Step 6) misclassifies a file — e.g. an unrelated file lands in `implementation_plans/` or `decisions/` during the same step — as a plan/decision artifact | Low | Filename pattern match (`{KEY}-implementation-plan.md`, `{KEY}-approach-decision.md`) is strict; log and skip files that don't match either pattern rather than guessing |
| `git checkout -` (return-to-previous-ref) in `createPlanPR`/`createDecisionPR` fails if the repo was already in a detached HEAD or mid-rebase state | Low | Existing `createCheckpointBranchAndPR` already uses `git checkout -` successfully in production; no new risk introduced beyond what's already accepted for checkpoint PRs |
| Fallback-to-`main` commit (Step 3/4, AC6) races with a concurrent `commitPlanningArtifacts` call from a different lane's step completing at nearly the same time, since both mutate `main` via `execSync` synchronously | Medium | Node.js is single-threaded for synchronous `execSync` calls — no two `runLane` invocations can interleave `execSync` calls mid-sequence; the existing `commitPlanningArtifacts`/`archiveStoryArtifacts` already rely on this same single-thread guarantee, so no new risk class is introduced, but this should be called out in code review since it's easy to misjudge as unsafe |
| Poller (`setTimeout` loop) accumulates unbounded if a PR is opened but the server is restarted many times without the PR ever merging — each restart's Step 10 recovery starts a new poller for the same `prUrl` without checking whether one is already conceptually "in flight" from a crashed prior process | Low | This is safe in practice because `planningPRPollers` is an in-memory `Map` that does not survive restart — only one poller per `prUrl` can exist within a single process lifetime; document this explicitly so a future multi-process deployment doesn't silently double-poll |
| Decision-document `status: pending` re-pause (Step 7d) creates a duplicate `CheckpointQueueEntry` if `compositeId`/`stepId` reuse collides with an entry already re-added by a different resume attempt | Low | Reuse the same `stepId` as the original entry (not a new composite) so a second re-push simply replaces/matches the same key; add a defensive check (find-and-replace by `stepId`, not blind push) when re-pausing |
| `planningArtifact` field absent on checkpoint entries persisted by server versions running before this story | Low | Field is optional; all new guard checks (`entry.planningArtifact !== undefined`) treat absence as "not a planning-artifact checkpoint," falling through cleanly to existing resume logic |
| Two skill files may exist in two locations (`~/.config/opencode/skills/...` and this planning repo's `opencode-config/skills/...` mirror) and Step 11/12 could update one but miss the other, leaving stale instructions in whichever copy agents actually load at runtime | Medium | Step 11/12 explicitly instruct checking both locations at execution time; confirm which path OpenCode actually loads from before editing (per `agent-tools-technical-context.md`, tools/skills are loaded from `~/.config/opencode/`, so that copy is authoritative — the planning-repo mirror, if present, is likely a tracked backup and should be kept in sync but is not itself loaded by the agent) |

## Commit Message Template

```
feat(checkpoint-management): open Bitbucket PR for planning artifacts as approval gate (ARC-1371)

- Add createPlanPR, createDecisionPR, isPRMerged, startPlanningPRPoller to checkpointGit.ts
- Add triggerPlanningArtifactPR and resumeLaneFromPlanningPR to laneRunner.ts
- Detect plan/decision artifacts via git status before falling through to commitPlanningArtifacts
- Add planningArtifact discriminator field to CheckpointQueueEntry
- Add prPollIntervalMs to SessionConfig (default 30 s)
- Guard POST /checkpoints/:stepId/resume from manual resume of planning PR checkpoints
- Add startup recovery for in-flight planning PR pollers in index.ts
- Update write-implementation-plan and start-execution-session skills to remove
  manual commit and status:answered instructions
```
