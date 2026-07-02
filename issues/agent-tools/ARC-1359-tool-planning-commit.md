| Field | Value |
|-------|-------|
| Type | Story |
| Priority | Medium |
| Status | To Do |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-02 |
| Parent Epic | ARC-1348-epic: OpenCode Agent Tools |
| Jira | ARC-1359 |

# ARC-1359: Implement `planning_commit` OpenCode tool

## Goal

Create a custom OpenCode tool at `~/.config/opencode/tools/planning_commit.ts` that performs `git add <files>`, `git commit -m <message>`, and `git push` in the planning repo as a single atomic operation — used throughout the agent workflow wherever planning artifacts need to be committed.

## Problem

Throughout the skill pipeline, the agent repeats the same three git commands:
1. `git add <file(s)>`
2. `git commit -m "{type}({lane}): {message}"`
3. `git push`

This is done after claiming a story, after writing a plan, after each implementation step, after writing a completion summary, and after closing an issue. The commit message format must follow `{type}({lane}): {summary}`. Agents frequently forget to push, use wrong message format, or stage unintended files.

## Acceptance Criteria

**Given** a list of files and a commit message,
**When** `planning_commit({ repo_path, files, message })` is called,
**Then** only those files are staged, committed with the given message, and pushed — returning the commit hash.

---

**Given** one of the specified files does not exist or has no changes,
**When** `planning_commit` is called,
**Then** the tool returns an error listing which files could not be staged.

---

**Given** the push fails due to a network error,
**When** `planning_commit` is called,
**Then** the local commit is preserved and the tool returns an error: `"Committed locally ({hash}) but push failed: {reason}"`.

---

**Given** `files` is `["."]` (stage all),
**When** `planning_commit` is called,
**Then** all modified tracked files are staged and committed.

## In Scope

- TypeScript tool file at `~/.config/opencode/tools/planning_commit.ts`
- `git add`, `git commit`, `git push` in the repo at `repo_path`
- Returns: `{ hash: string, pushed: boolean, error?: string }`

## Out of Scope

- Branch creation or switching
- Merge operations
- Rebasing

## Constraints

- Tool placed in `~/.config/opencode/tools/` (global)
- Uses `@opencode-ai/plugin` `tool()` helper and `Bun.$`
- Inputs: `repo_path` (absolute), `files` (string array), `message` (string)

## Dependencies

- OpenCode custom tools API (`@opencode-ai/plugin`)

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-tools/ARC-1359-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1359-COMPLETION-SUMMARY.md` | TBD |
