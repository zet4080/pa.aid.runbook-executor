| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-01 |
| Parent Epic | [ARC-1344](https://proalpha.atlassian.net/browse/ARC-1344) ARC: Agent Workflow Improvements |
| Jira | [ARC-1345](https://proalpha.atlassian.net/browse/ARC-1345) |

# ARC-1345: Fix checkpoint workflow — runbook file changes must not appear in checkpoint PR

## Goal

Fix `createCheckpointBranchAndPR` in `packages/server/src/git/checkpointGit.ts` (`pa.aid.conductor.ts`) so that runbook file changes in the planning repo are committed directly to the base branch **before** the checkpoint branch is created, ensuring they never appear in the checkpoint PR.

## Problem

The planning repo (configured via `REPO_ROOT`) contains two categories of files the agent modifies:

| Category | Examples | Should end up in checkpoint PR? |
|----------|----------|----------------------------------|
| Planning artifacts | implementation plans, decision docs, design specs | ✅ YES |
| Runbook files | `docs/plans/runbook-*.md` checkoffs, wave-gate marks, claim marks | ❌ NO — must go directly to `main` |

**What currently happens at a checkpoint:**

1. The agent has been working — it has modified runbook files (checkoffs, wave-gate marks, etc.) AND created planning artifacts (implementation plans, etc.); both are uncommitted in the planning repo working tree.
2. `createCheckpointBranchAndPR` fires in `checkpointGit.ts`.
3. `git checkout -b checkpoint/{lane}/{stepId}` runs — because git carries uncommitted working-tree changes to the new branch, **both** the runbook changes and the planning artifacts land on the checkpoint branch.
4. The branch is pushed and a PR is opened.
5. **Result:** runbook checkoffs and wave-gate marks appear in the checkpoint PR — **wrong**.

**What should happen:**

1. Before `git checkout -b`, the checkpoint logic detects any modified files matching `docs/plans/runbook-*.md` in the planning repo.
2. Those changes are staged and committed directly to the base branch (e.g. `main`).
3. Then `git checkout -b` runs — only the planning artifacts follow to the checkpoint branch.
4. The branch is pushed and a PR is opened containing only planning artifacts.
5. Runbook changes are already safely committed to `main`.

## In Scope

- `createCheckpointBranchAndPR` in `pa.aid.conductor.ts` — `packages/server/src/git/checkpointGit.ts`:
  - Before branching, detect uncommitted changes to files matching `docs/plans/runbook-*.md` in the planning repo (`REPO_ROOT`)
  - Stage and commit those changes directly to the current base branch
  - Proceed with `git checkout -b` as usual — only non-runbook changes (planning artifacts) will follow

## Out of Scope

- Changing how planning artifacts flow into the checkpoint branch and PR
- Changing the PR creation logic or Bitbucket integration
- Changing agent skill instructions or runbook authoring conventions

## Acceptance Criteria

**Given** the agent has modified runbook files AND created planning artifacts (both uncommitted in the planning repo),
**When** a checkpoint step is reached,
**Then** runbook file changes (`docs/plans/runbook-*.md`) are committed directly to the base branch and do NOT appear in the checkpoint PR.

---

**Given** the agent has created planning artifacts (uncommitted) in the planning repo,
**When** a checkpoint step is reached,
**Then** those planning artifacts appear in the checkpoint branch and PR as expected.

---

**Given** the checkpoint branch has been created and pushed,
**When** the PR is opened,
**Then** the diff contains only planning artifacts — no runbook file changes.

## Constraints

- Runbook files are identifiable by path pattern `docs/plans/runbook-*.md` within the planning repo root (`REPO_ROOT`).
- The commit of runbook changes to the base branch must happen atomically before the `git checkout -b` so that no runbook changes are present in the working tree when the branch is created.
- If there are no runbook changes, the pre-branch commit step is a no-op.

## Dependencies

- `pa.aid.conductor.ts` — `packages/server/src/git/checkpointGit.ts`

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-workflow/ARC-1345-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1345-COMPLETION-SUMMARY.md` | TBD |
