| Field | Value |
|-------|-------|
| Type | Story |
| Priority | Medium |
| Status | To Do |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-02 |
| Parent Epic | ARC-1348-epic: OpenCode Agent Tools |
| Jira | ARC-1352 |

# ARC-1352: Implement `runbook_check_dependencies` OpenCode tool

## Goal

Create a custom OpenCode tool at `~/.config/opencode/tools/runbook_check_dependencies.ts` that reads a runbook file and returns all unmet cross-lane dependency gate checkboxes that must be satisfied before a given story can begin — replacing manual inspection by the agent.

## Problem

The `start-execution-session` skill requires the agent to scan the runbook for dependency gate checkboxes that appear before the target story. These are cross-lane prerequisites formatted as checked or unchecked items. Agents often skip this check or misread the gate status, leading to starting work on stories that are blocked.

## Acceptance Criteria

**Given** a runbook with dependency gate checkboxes before a story,
**When** `runbook_check_dependencies({ runbook_path, story_key })` is called,
**Then** the tool returns a list of all dependency items above that story's section, indicating whether each is checked (`[x]`) or unchecked (`[ ]`).

---

**Given** all dependency gate checkboxes above the story are `[x]`,
**When** `runbook_check_dependencies` is called,
**Then** the tool returns `{ blocked: false, unmet: [] }`.

---

**Given** one or more dependency gate checkboxes are `[ ]`,
**When** `runbook_check_dependencies` is called,
**Then** the tool returns `{ blocked: true, unmet: ["ARC-XXXX must be merged before starting"] }`.

---

**Given** the story has no dependency gates before it,
**When** `runbook_check_dependencies` is called,
**Then** the tool returns `{ blocked: false, unmet: [] }`.

## In Scope

- TypeScript tool file at `~/.config/opencode/tools/runbook_check_dependencies.ts`
- Read-only: no file modifications, no git operations
- Parse dependency gate items (lines containing `must be merged`, `pre-check`, or similar gate language) above the target story section
- Returns structured result: `{ blocked: boolean, unmet: string[] }`

## Out of Scope

- Modifying dependency gate checkboxes
- Resolving or merging dependent stories
- Cross-repo dependency tracking

## Constraints

- Tool placed in `~/.config/opencode/tools/` (global)
- Uses `@opencode-ai/plugin` `tool()` helper
- Inputs: `runbook_path` (absolute), `story_key`
- Read-only — no side effects

## Dependencies

- OpenCode custom tools API (`@opencode-ai/plugin`)
- `runbook_find_next_story` (ARC-1348) — typically called before this tool

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-tools/ARC-1352-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1352-COMPLETION-SUMMARY.md` | TBD |
