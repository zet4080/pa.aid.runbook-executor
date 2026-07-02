| Field | Value |
|-------|-------|
| Type | Story |
| Priority | Medium |
| Status | To Do |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-02 |
| Parent Epic | ARC-1348-epic: OpenCode Agent Tools |
| Jira | ARC-1357 |

# ARC-1357: Implement `get_branch_diff` OpenCode tool

## Goal

Create a custom OpenCode tool at `~/.config/opencode/tools/get_branch_diff.ts` that returns the unified diff of all changes on the current branch relative to the merge-base with `main` (or `master`) — used in `local-code-review` for the diff review step.

## Problem

The `local-code-review` skill runs:
```bash
git diff $(git merge-base HEAD main 2>/dev/null || git merge-base HEAD master)...HEAD
```

This command is subtle — agents often get the syntax wrong, use `HEAD..main` instead of `main...HEAD`, or fail silently when neither `main` nor `master` exists. The result is a diff review against the wrong baseline.

## Acceptance Criteria

**Given** a feature branch with commits since branching from `main`,
**When** `get_branch_diff({ directory })` is called,
**Then** the tool returns the unified diff from merge-base to HEAD as a string.

---

**Given** the base branch is `master` instead of `main`,
**When** `get_branch_diff` is called,
**Then** the tool auto-detects `master` as the base and returns the correct diff.

---

**Given** there are no changes since the merge-base,
**When** `get_branch_diff` is called,
**Then** the tool returns `{ diff: "", changed_files: [] }`.

---

**Given** the repository has no `main` or `master` branch,
**When** `get_branch_diff` is called,
**Then** the tool returns an error: `"Cannot determine base branch — neither main nor master found"`.

## In Scope

- TypeScript tool file at `~/.config/opencode/tools/get_branch_diff.ts`
- Auto-detects base branch (`main` preferred, `master` fallback)
- Returns: `{ diff: string, changed_files: string[], base_branch: string }`

## Out of Scope

- Applying or reverting the diff
- Filtering by file type or path

## Constraints

- Tool placed in `~/.config/opencode/tools/` (global)
- Uses `@opencode-ai/plugin` `tool()` helper and `Bun.$`
- Input: `directory` (absolute path, defaults to `context.worktree`)
- Read-only — no side effects

## Dependencies

- OpenCode custom tools API (`@opencode-ai/plugin`)

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-tools/ARC-1357-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1357-COMPLETION-SUMMARY.md` | TBD |
