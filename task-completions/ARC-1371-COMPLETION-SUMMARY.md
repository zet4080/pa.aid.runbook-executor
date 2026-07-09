# ARC-1371 Conductor: Open PR for Implementation Plans and Decision Documents as Approval Gate — Completion Summary

**Issue:** `issues/checkpoint-management/ARC-1371-conductor-pr-approval-gate-for-plans-and-decisions.md`
**Implementation Plan:** `implementation_plans/checkpoint-management/ARC-1371-implementation-plan.md`
**Epic:** ARC-1295 — ARC: Checkpoint Management
**Repo:** `pa.aid.conductor.ts`
**Branch:** `ARC-1371` (off `main` at `03efc9b`, includes merged ARC-1367–ARC-1370)
**Pull Request:** https://bitbucket.org/proalpha/pa.aid.conductor.ts/pull-requests/8 (open, not yet merged)
**Completed:** 2026-07-09
**Duration:** 1 session
**Cost:** —

## Acceptance Criteria Verification

| # | Criterion | Status | Evidence |
|---|-----------|--------|----------|
| AC1 | Given the conductor writes an implementation plan file for a HIGH story, when the file is ready to commit, then it creates branch `plan/{lane}/{KEY}`, commits the file there, opens a Bitbucket PR titled `Plan review: {KEY} — {story title}` targeting `main`, and pauses the lane at a 🔴 checkpoint | Passed | `createPlanPR` in `checkpointGit.ts` creates the `plan/{lane}/{storyKey}` branch, commits the artifact, and opens the PR with the specified title. `triggerPlanningArtifactPR` in `laneRunner.ts` pushes a `CheckpointQueueEntry` and sets `laneStatusMap` to `paused`. Covered by unit tests in `checkpointGit.test.ts` (branch naming, PR title/body) and `laneRunner.test.ts` (pause + queue entry). |
| AC2 | Given the conductor writes a decision document file, when the file is ready to commit, then it creates branch `decision/{KEY}-{type}`, commits the file there, opens a Bitbucket PR titled `Decision: {KEY} — {type}` targeting `main`, and pauses the lane at a 🔴 checkpoint | Passed | `createDecisionPR` mirrors `createPlanPR` with `decision/{storyKey}-{decisionType}` branch naming and the `Decision: {KEY} — {type}` PR title. Same pause mechanism reused. Covered by `checkpointGit.test.ts` and `laneRunner.test.ts` decision-artifact tests. |
| AC3 | Given a plan or decision PR is open and the supervisor merges it, when the conductor detects the merge (via poll), then the lane resumes; for decision documents the conductor reads the merged file from `main` to extract `selected_approach` | Passed | `isPRMerged` + `startPlanningPRPoller` implement `setTimeout`-based polling (default interval `prPollIntervalMs: 30000`, added to `SessionConfig`/`DEFAULT_SESSION_CONFIG`). On merge, `resumeLaneFromPlanningPR` dequeues the checkpoint and re-dispatches `runLane`; for decisions it reads the merged file's `status`/`selected_approach` before continuing. Covered by fake-timer poller tests in `checkpointGit.test.ts` and resume-path tests in `laneRunner.test.ts`. |
| AC4 | Given a plan PR is merged, when the conductor resumes the lane, then the implementation plan file is now on `main` and the agent proceeds to execute the plan | Passed | `resumeLaneFromPlanningPR` re-parses the runbook and re-dispatches the still-unchecked agent step via `runLane`; the merged plan file is present on `main`'s working tree for the agent to read. Covered by `laneRunner.test.ts` resume test asserting `runLane` is invoked after dequeue. |
| AC5 | Given a decision PR is merged, when the conductor resumes the lane, then the decision document's `status` is read from the merged file; `answered` continues with `selected_approach`, `pending` re-pauses and logs a warning | Passed | `resumeLaneFromPlanningPR` reads the merged decision file for the literal `status: answered` line; if absent, it re-pushes a `CheckpointQueueEntry` with the same `planningArtifact` metadata and re-pauses the lane without calling `runLane`, logging a warning. Covered by a dedicated `laneRunner.test.ts` case for `status: pending` re-pause vs. `status: answered` continuation. |
| AC6 | Given PR creation fails (Bitbucket unavailable), when the conductor tries to open the PR, then the artifact is committed directly to `main` as a fallback, the checkpoint is still raised as a 🔴 pause, and the supervisor is notified via the checkpoint queue that no PR was created | Passed (after review fix) | `createPlanPR`/`createDecisionPR` catch PR-creation failures and fall back to a direct commit on `main`. A **BLOCKER** was found and fixed during local code review (see below) — the fallback commit initially used an incorrect git pathspec and silently no-op'd. After the fix, the fallback path is verified via `checkpointGit.test.ts` (fallback commit assertions) and `laneRunner.test.ts` (checkpoint entry pushed with no `prUrl`, no poller started). |
| AC7 | Given the feature is implemented, when an agent writes a plan/decision document, then the `write-implementation-plan` and `start-execution-session` skills no longer instruct manual commit or `status: answered` editing | Passed | Both skill files (`~/.config/opencode/skills/write-implementation-plan/SKILL.md`, `~/.config/opencode/skills/start-execution-session/SKILL.md`) updated to describe the automatic PR-based flow instead of manual commit/push/status-edit instructions. Verified by direct read of both files post-edit. |

## Implementation Summary

Implemented the PR-based approval gate for planning artifacts as an extension of the existing `checkpointGit.ts` PR-creation pattern (no new module, per the issue's constraint):

- **`checkpointGit.ts`**: added `createPlanPR`, `createDecisionPR`, `isPRMerged`, `startPlanningPRPoller`, reusing the existing private helpers (`getBitbucketCredentials`, `parseRemoteUrl`, `parsePrId`, `parsePrUrlRepo`) and the same branch → commit → push → PR-POST sequence as `createCheckpointBranchAndPR`.
- **`laneRunner.ts`**: added git-status-based detection of plan/decision artifacts (scoped to `implementation_plans/` and `decisions/`) inserted before the existing `commitPlanningArtifacts` call; added `triggerPlanningArtifactPR` (creates the PR, pushes a `CheckpointQueueEntry`, pauses the lane, starts a poller) and `resumeLaneFromPlanningPR` (dequeues on merge, re-dispatches the lane, handles the decision `status: pending` re-pause case).
- **`state/types.ts`**: added `prPollIntervalMs` to `SessionConfig` (default 30s) and a `planningArtifact` discriminator field to `CheckpointQueueEntry`.
- **`routes/checkpoints.ts`**: guarded the manual `POST /checkpoints/:stepId/resume` route so planning-PR checkpoints with an open PR cannot be manually resumed (returns 409) — only PR merge or the AC6 fallback (no `prUrl`) path can resume them.
- **`index.ts`**: added startup recovery that re-starts in-flight PR pollers for any `checkpointQueue` entries with `planningArtifact` + `prUrl` set, since pollers are in-memory only.
- **Skills**: updated `write-implementation-plan` and `start-execution-session` to remove manual-commit and `status: answered`-editing instructions, replacing them with a description of the automatic PR/checkpoint/poll/resume flow.

### Local Code Review Findings and Fixes (commit `f569b5d`)

Local code review (per the `local-code-review` skill) surfaced two findings, both fixed on the branch before requesting integration:

1. **BLOCKER — AC6 fallback-to-`main` commit used an incorrect git-add pathspec.** After `git checkout main` in the fallback path, the artifact file did not exist in `main`'s working tree (it had only been added on the disposable `plan/`/`decision/` branch), so `git add -- {artifactPath}` failed with "pathspec did not match any files," causing the fallback commit to silently no-op on every real PR-creation failure — directly breaking AC6. Fixed by adding `git checkout {branch} -- {artifactPath}` before `git add`, which pulls the file's content from the disposable branch into `main`'s working tree (and stages it) prior to commit. Verified against a real git repro before and after the fix. Regression test added in `checkpointGit.test.ts`.

2. **ISSUE — unhandled git-level throw in `triggerPlanningArtifactPR`.** A raw git-level throw from `createPlanPR`/`createDecisionPR` (e.g. a stale lock file or dirty working tree, outside the function's own internal Bitbucket-API try/catch) previously propagated unhandled, leaving the lane stuck in `running` with no checkpoint entry and no recovery path. Fixed by wrapping the call in `laneRunner.ts` so any such failure still raises a checkpoint (`planningArtifact` set, empty branch, no `prUrl`) and pauses the lane, matching the existing `createCheckpointBranchAndPR` resilience pattern used elsewhere in the file. Regression test added in `laneRunner.test.ts`.

## Verification Steps

```bash
# In worktree /worktrees/ARC-1371
cd /worktrees/ARC-1371
npm test   # 380 server tests + 192 UI tests, all passing
npx tsc --noEmit -p packages/server   # clean, no errors
```

## Tests Added/Modified

- `packages/server/src/git/checkpointGit.test.ts` — `createPlanPR`/`createDecisionPR` branch naming, PR title/body, fallback-to-`main` commit (including the pathspec regression fix); `isPRMerged` true/false/throw cases; `startPlanningPRPoller` merge-detection, `stop()`, and poll-error survival (fake timers).
- `packages/server/src/__tests__/laneRunner.test.ts` — plan/decision artifact detection routing to `triggerPlanningArtifactPR` instead of `commitPlanningArtifacts`; checkpoint pause + queue entry with `planningArtifact`; poller start on `prUrl` present, no poller on fallback; `resumeLaneFromPlanningPR` dequeue-and-resume; decision `status: pending` re-pause vs. `status: answered` continuation; regression test for the unhandled-throw fix.
- `packages/server/src/__tests__/checkpoints.test.ts` — `POST /checkpoints/:stepId/resume` returns 409 for planning-PR checkpoints with an open PR; resumes normally for the no-`prUrl` fallback case.

## Rollback Notes

Changes are additive: new exported functions in `checkpointGit.ts`, new optional fields in `state/types.ts` (`prPollIntervalMs`, `planningArtifact`), new branch/route guard in `checkpoints.ts`, and new startup recovery logic in `index.ts`. `commitPlanningArtifacts` and `archiveStoryArtifacts` are untouched and continue to handle completion summaries and non-plan/decision artifacts directly to `main`. To roll back, revert commits `66aefd5` and `f569b5d` on the `ARC-1371` branch before merge; no data migration is required since `planningArtifact`/`prPollIntervalMs` are optional fields.

## Next Steps

- PR #8 (https://bitbucket.org/proalpha/pa.aid.conductor.ts/pull-requests/8) is open against `main` and awaiting supervisor/human review and merge.
- Once merged: archive this issue (`archive_issue`) and update runbook to check off the final "Archive issue" step — not done yet, per workflow (archival only happens after PR merge).
