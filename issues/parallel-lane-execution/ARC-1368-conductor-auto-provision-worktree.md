| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-06 |
| Parent Epic | ARC-1290 — ARC: Parallel Lane Execution |
| Jira | ARC-1368 |

# ARC-1368: Conductor: auto-provision feature git worktree for HIGH stories

## Goal

When the conductor starts executing a HIGH-priority story it should automatically create a git worktree at `/repos/<KEY>` on branch `<KEY>` (branched from `main` in `pa.aid.conductor.ts`) and inject the worktree path into the agent's prompt context. This eliminates manual branch-creation and worktree-setup ceremony from agent workflows entirely.

## Problem

Agents currently must run `git worktree list`, check for existing worktrees, and call `git worktree add /repos/<KEY> -b <KEY> main` themselves at the start of every HIGH story. The conductor already knows the story KEY and repo path before it calls the agent — these are fully deterministic operations that should never reach the agent.

## Acceptance Criteria

**Given** The conductor identifies a HIGH-priority story with key `ARC-XXXX`,
**When** It is about to dispatch the first step to the agent,
**Then** A git worktree exists at `/repos/ARC-XXXX` on branch `ARC-XXXX` (created from `main`) before the agent prompt is sent.

---

**Given** A worktree for `/repos/ARC-XXXX` already exists (resume scenario),
**When** The conductor tries to provision it,
**Then** The existing worktree is reused without error; no duplicate branch is created.

---

**Given** The worktree has been provisioned,
**When** The agent prompt is built,
**Then** The prompt context includes the worktree path so the agent knows where to write code.

---

**Given** The auto-provisioning is implemented,
**When** An agent session starts on a HIGH story,
**Then** The `start-execution-session` skill no longer instructs the agent to run `git worktree list` or `git worktree add` as ceremony.

## In Scope

- Detect HIGH story type from the parsed runbook step (`checkpointLevel === 'high'`)
- Execute `git worktree add /repos/<KEY> -b <KEY> main` in the `pa.aid.conductor.ts` repo
- Handle already-exists: detect via `git worktree list` output or catch the specific git error, then reuse
- Inject worktree path as a variable in the agent prompt template (e.g., `WORKTREE_PATH=/repos/<KEY>`)
- Update `start-execution-session` skill to remove manual worktree setup instructions
- Unit test: worktree is created before the agent prompt is built; resume path reuses existing worktree

## Out of Scope

- MEDIUM/LOW batch stories — they operate in the main checkout, no worktree needed
- Worktree pruning or cleanup after story completion (separate concern)
- Creating the PR — that is already handled by `checkpointGit.ts`

## Constraints

- Branch name must equal the story key exactly — no prefixes, no suffixes (e.g., `ARC-1368`, never `feature/ARC-1368`)
- Worktree base branch is always `main` in `pa.aid.conductor.ts`
- Failure to create worktree is a hard error — the story cannot proceed without it

## Dependencies

- ARC-1367 (auto-claim must fire before worktree creation to mark ownership first)

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-tools/ARC-1368-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1368-COMPLETION-SUMMARY.md` | TBD |