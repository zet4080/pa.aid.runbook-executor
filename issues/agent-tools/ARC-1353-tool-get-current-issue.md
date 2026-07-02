# ARC-1353: Implement `get_current_issue` OpenCode tool

## Goal

Create a custom OpenCode tool at `~/.config/opencode/tools/get_current_issue.ts` that resolves the current issue from the active git branch name, finds the corresponding issue file in the planning repo, and returns the issue ID, file path, and parsed acceptance criteria — used at the start of `local-code-review` to avoid manual branch parsing and file searching.

## Problem

The `local-code-review` skill requires the agent to:
1. Run `git branch --show-current`
2. Extract the issue ID with a regex (`grep -oE '^[A-Z]+-[0-9]+'`)
3. Search `issues/` recursively for a file matching `${ISSUE_ID}-*.md`
4. Read the file and extract acceptance criteria

This chain of bash commands and text parsing is done at the start of every review. If the branch name does not match, or the file is not found, the agent proceeds without AC validation — silently skipping a critical quality gate.

## Acceptance Criteria

**Given** the current git branch is named with a valid issue key prefix (e.g. `ARC-1285` or `ARC-1285-some-description`),
**When** `get_current_issue({ planning_repo_path })` is called,
**Then** the tool returns `{ issueId: "ARC-1285", filePath: "issues/core-infrastructure/ARC-1285-*.md", acceptanceCriteria: string[] }`.

---

**Given** no issue file exists for the extracted issue ID,
**When** `get_current_issue` is called,
**Then** the tool returns `{ issueId: "ARC-XXXX", filePath: null, acceptanceCriteria: [] }` with a warning that AC validation will be skipped.

---

**Given** the current branch name does not match the `[A-Z]+-[0-9]+` pattern,
**When** `get_current_issue` is called,
**Then** the tool returns `{ issueId: null, filePath: null, acceptanceCriteria: [] }` with a warning.

---

**Given** the planning repo path does not exist or is not a git repository,
**When** `get_current_issue` is called,
**Then** the tool returns an error with a clear message.

## In Scope

- TypeScript tool file at `~/.config/opencode/tools/get_current_issue.ts`
- Reads current branch from `context.worktree` git repo
- Searches `issues/` directory tree in the planning repo for matching file
- Parses acceptance criteria lines from the issue file (lines under `## Acceptance Criteria`)
- Returns: `{ issueId: string | null, filePath: string | null, acceptanceCriteria: string[] }`

## Out of Scope

- Modifying the issue file
- Running any quality checks (those are `run_tests`, `run_lint`, `run_typecheck`)
- Jira API integration

## Constraints

- Tool placed in `~/.config/opencode/tools/` (global)
- Uses `@opencode-ai/plugin` `tool()` helper
- Input: `planning_repo_path` (absolute path to planning repo, e.g. `/repos/pa.aid.runbook-executor`)
- Uses `context.worktree` for the application repo git root
- Read-only — no side effects

## Dependencies

- OpenCode custom tools API (`@opencode-ai/plugin`)
- Planning repo issue file format (see `issues/` in `pa.aid.runbook-executor`)

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-tools/ARC-1353-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1353-COMPLETION-SUMMARY.md` | TBD |
