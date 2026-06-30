# ARC-1302 Completion Summary

| Key | Value |
| --- | --- |
| Story | ARC-1302 |
| Status | Completed |
| Branch | ARC-1302 in /repos/pa.aid.wsl-setup.sh |
| Commit | ba4da5a

## Acceptance Criteria Evidence

### AC 1 — git branch + Bitbucket PR

- **File**: `packages/server/src/git/checkpointGit.ts`
  - `createCheckpointBranchAndPR()` calls `execSync('git checkout -b checkpoint/{lane}/{stepId}')`, `execSync('git push origin …')`, then `execSync('git checkout -')`, then POSTs to `https://api.bitbucket.org/2.0/repositories/{workspace}/{slug}/pullrequests`
  - Uses a 10s timeout on all git commands
  - PR creation failure is non-blocking (try/catch → `prUrl: undefined`)
- **File**: `packages/server/src/runner/laneRunner.ts`
  - Wired at both checkpoint pause sites (sub-step gate and top-level checkpoint step)
  - Fire-and-forget via `void createCheckpointBranchAndPR(...).then(...).catch(...)`
- **Test file**: `packages/server/src/git/checkpointGit.test.ts`
  - `calls execSync for git checkout -b, git push, and git checkout -`
  - `returns branch name and prUrl on success`
  - `calls fetch with correct Bitbucket API URL and Basic auth header`

### AC 2 — ResumeCard displayed with PR URL, emoji, runbook name

- **File**: `packages/ui/src/components/ResumeCard.tsx`
  - Renders checkpoint level emoji (🔴/🟡/🟢) via `role="img" aria-label="checkpoint level: {level}"`
  - Renders runbook label derived from runbookPath (strips `runbook-` prefix and `.md` suffix)
  - Renders `<a href={prUrl}>View PR</a>` when prUrl present; `<span>PR pending</span>` otherwise
  - Left border coloured: `border-red-500` (high), `border-amber-500` (medium), `border-green-500` (low)
- **File**: `packages/ui/src/hooks/useCheckpointQueue.ts`
  - Polls `GET /api/state` every 2s, extracts `checkpointQueue`, exposes `resumeCheckpoint(stepId)`
- **File**: `packages/ui/src/pages/Dashboard.tsx`
  - Integrates `useCheckpointQueue`; renders `<ResumeCard>` for each entry above lane panels
- **Test file**: `packages/ui/src/components/ResumeCard.test.tsx`
  - `renders 🔴 for high checkpoint level`
  - `renders a "View PR" link when prUrl is present`
  - `renders "PR pending" when prUrl is undefined`
  - `strips runbook- prefix and.md suffix from runbookPath`

### AC 3 — Card remains until Resume clicked

- **File**: `packages/ui/src/pages/Dashboard.tsx`
  - Cards rendered from `checkpointQueue` state; entry only removed from queue when `POST /api/checkpoints/:stepId/resume` succeeds
- **File**: `packages/server/src/routes/checkpoints.ts`
  - `POST /api/checkpoints/:stepId/resume` removes entry from queue + persists + fires `runLane`
  - 404 returned if stepId not found
- **Test file**: `packages/server/src/__tests__/checkpoints.test.ts`
  - `returns 200 { ok: true } when stepId exists in queue`
  - `removes the entry from checkpointQueue on resume`
  - `returns 404 when stepId not in queue`

## Test Results

| Test Type | Status |
| --- | --- |
| Server | 261 tests pass (19 test files) |
| UI | 138 tests pass (16 test files) |
| Server typecheck (tsc --noEmit) | pass |
| UI typecheck (tsc --noEmit) | pass |

## Files Changed

- `packages/server/src/state/types.ts` — added `prUrl?: string` to `CheckpointQueueEntry`
- `packages/server/src/git/checkpointGit.ts` — new: `createCheckpointBranchAndPR()`
- `packages/server/src/git/checkpointGit.test.ts` — new: 8 tests
- `packages/server/src/runner/laneRunner.ts` — wired PR creation at both checkpoint pause sites
- `packages/server/src/__tests__/laneRunner.test.ts` — added mock + 5 ARC-1302 tests
- `packages/server/src/routes/checkpoints.ts` — new: POST /api/checkpoints/:stepId/resume
- `packages/server/src/__tests__/checkpoints.test.ts` — new: 8 tests
- `packages/server/src/index.ts` — registered checkpointsRouter
- `packages/ui/src/types/runbook.ts` — added `CheckpointQueueEntry` interface
- `packages/ui/src/hooks/useCheckpointQueue.ts` — new: polling hook
- `packages/ui/src/hooks/useCheckpointQueue.test.ts` — new: 10 tests
- `packages/ui/src/components/ResumeCard.tsx` — new: persistent card component
- `packages/ui/src/components/ResumeCard.test.tsx` — new: 15 tests
- `packages/ui/src/pages/Dashboard.tsx` — integrated ResumeCards above lane panels
