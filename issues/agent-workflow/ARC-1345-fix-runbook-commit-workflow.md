| Field | Value |
|-------|-------|
| Type | Story |
| Priority | Low |
| Status | To Do |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-01 |
| Parent Epic | [ARC-1344](https://proalpha.atlassian.net/browse/ARC-1344) ARC: Agent Workflow Improvements |
| Jira | [ARC-1345](https://proalpha.atlassian.net/browse/ARC-1345) |

# ARC-1345: Clarify checkpoint-branch/PR scope vs. runbook artifact commit flow

## Description

The original issue assumed that skills were incorrectly implying PRs for runbook artifacts (claim, checkoff, wave-gate commits). Investigation showed this assumption was wrong: no skill ever implied PRs for runbook artifacts. The agent always commits runbook artifacts directly to `main` in `pa.aid.runbook-executor` — no PR, no branch.

The real behavior at checkpoint is different: when `pa.aid.conductor.ts` reaches a CHECKPOINT/GATE step, `laneRunner.ts` calls `createCheckpointBranchAndPR` in `packages/server/src/git/checkpointGit.ts`. This function creates a branch named `checkpoint/{lane}/{stepId}` in the **application repo** (`pa.aid.conductor.ts`), pushes it to remote, and non-blockingly attempts to open a Bitbucket PR for that branch. This checkpoint branch captures all application-code changes in `pa.aid.conductor.ts` up to that point in the lane. It has no connection to `pa.aid.runbook-executor` or its artifacts.

The two flows are entirely independent:

- **Runbook artifacts** (claim, checkoff, wave-gate, completion summary) → committed directly to `main` in `pa.aid.runbook-executor`; no branch, no PR
- **Application code changes** at checkpoint → `checkpoint/{lane}/{stepId}` branch + PR created by the conductor system in `pa.aid.conductor.ts`

The issue is to document this distinction clearly in `docs/plans/HOW-THIS-WORKS.md` so that agents and developers have an accurate mental model of both flows.

## In Scope

- `docs/plans/HOW-THIS-WORKS.md` — add or update a section that explains: (a) runbook artifacts commit directly to `main` in `pa.aid.runbook-executor`, no PR; (b) at checkpoint, the conductor system creates a `checkpoint/{lane}/{stepId}` branch + PR in `pa.aid.conductor.ts` for application code changes; (c) these two flows are independent and do not interfere with each other
- Optionally: add a brief note to `start-execution-session` or `execute-implementation-plan` skills if either currently lacks any mention of the checkpoint branch mechanism

## Out of Scope

- Changes to `checkpointGit.ts` or any conductor application logic
- Changes to how application code PRs are created or merged
- Any Jira workflow changes

## Acceptance Criteria

1. Given `docs/plans/HOW-THIS-WORKS.md`, when it describes runbook artifact commits, then it states explicitly that claim, checkoff, wave-gate, and completion summary artifacts commit directly to `main` in `pa.aid.runbook-executor` with no PR or branch.
2. Given `docs/plans/HOW-THIS-WORKS.md`, when it describes the checkpoint mechanism, then it explains that at a CHECKPOINT/GATE step the conductor system creates a `checkpoint/{lane}/{stepId}` branch and opens a Bitbucket PR in `pa.aid.conductor.ts` for all application-code changes up to that point.
3. Given both descriptions exist in `HOW-THIS-WORKS.md`, when read together, then it is unambiguous that the two flows (runbook artifact commits vs. conductor checkpoint branches) are independent and never interleave.

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-workflow/ARC-1345-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1345-COMPLETION-SUMMARY.md` | TBD |
