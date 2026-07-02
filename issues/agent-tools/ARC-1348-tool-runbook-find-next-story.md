| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-02 |
| Parent Epic | ARC-1348-epic: OpenCode Agent Tools |
| Jira | ARC-1348 |

# ARC-1348: Implement `runbook_find_next_story` OpenCode tool

## Goal

Create a custom OpenCode tool at `~/.config/opencode/tools/runbook_find_next_story.ts` that parses a runbook markdown file and returns the first unchecked, unclaimed story in the active wave — eliminating manual markdown parsing by the agent.

## Problem

The `start-execution-session` skill currently requires the agent to manually parse runbook markdown to find the next story. This involves:
- Locating the active wave section
- Scanning for `- [ ]` story checkboxes
- Checking the `🔒 Claimed:` sub-item to distinguish unclaimed from in-progress stories
- Extracting the story KEY, title, risk level, and line number

Agents frequently make mistakes in this parsing — picking the wrong story, misreading claimed status, or failing to identify the correct wave section.

## Goal

A single tool call `runbook_find_next_story({ runbook_path })` replaces all of that. The tool reads the file, parses the structure, and returns a structured result the agent can use directly.

## Acceptance Criteria

**Given** a runbook file with multiple waves and stories,
**When** `runbook_find_next_story` is called with the runbook path,
**Then** it returns the first story whose checkbox is `[ ]` AND whose `🔒 Claimed:` sub-item is also `[ ]` (or absent), in wave order.

---

**Given** a story that has `- [x] 🔒 Claimed:` sub-item,
**When** `runbook_find_next_story` is called,
**Then** that story is skipped — it is considered in-progress.

---

**Given** all stories in all waves are either checked `[x]` or claimed,
**When** `runbook_find_next_story` is called,
**Then** the tool returns `null` with a message indicating no unclaimed work remains.

---

**Given** the runbook file does not exist at the given path,
**When** `runbook_find_next_story` is called,
**Then** the tool returns an error with the missing path.

## In Scope

- TypeScript tool file at `~/.config/opencode/tools/runbook_find_next_story.ts`
- Parses runbook markdown: wave headers, story checkboxes, Claimed sub-items
- Returns: `{ storyKey: string, title: string, wave: number, risk: string, lineNumber: number }` or `null`
- Works with the runbook format used in `pa.aid.runbook-executor/docs/plans/runbook-*.md`

## Out of Scope

- Writing back to the runbook file (that is `runbook_claim_story`)
- Parsing cross-lane dependency blocks
- Jira integration

## Constraints

- Tool is placed in `~/.config/opencode/tools/` (global, available in all sessions)
- Uses `@opencode-ai/plugin` `tool()` helper
- Input: `runbook_path` — absolute path to runbook markdown file
- Output must be JSON-serialisable

## Dependencies

- OpenCode custom tools API (`@opencode-ai/plugin`)
- Runbook markdown format (see `docs/plans/runbook-*.md` in `pa.aid.runbook-executor`)

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-tools/ARC-1348-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1348-COMPLETION-SUMMARY.md` | TBD |
