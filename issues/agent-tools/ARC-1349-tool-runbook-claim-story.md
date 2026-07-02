| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-02 |
| Parent Epic | ARC-1348-epic: OpenCode Agent Tools |
| Jira | ARC-1349 |

# ARC-1349: Implement `runbook_claim_story` OpenCode tool

## Goal

Create a custom OpenCode tool at `~/.config/opencode/tools/runbook_claim_story.ts` that marks a story as claimed in a runbook file — patching the `🔒 Claimed:` sub-item in-place and committing the change — so the agent never has to manually edit runbook markdown or issue git commands for this operation.

## Problem

The `start-execution-session` skill requires the agent to:
1. Find the `🔒 Claimed:` sub-item for the target story
2. Rewrite it from `- [ ] 🔒 Claimed:` to `- [x] 🔒 Claimed: {lane} / {YYYY-MM-DD HH:MM}`
3. `git add` the runbook file
4. `git commit -m "chore({lane}): claim {KEY}"`
5. `git push`

This is four steps with specific formatting requirements. Agents frequently produce incorrect timestamps, wrong commit message format, or forget to push.

## Acceptance Criteria

**Given** a runbook file and a story KEY with an unchecked `🔒 Claimed:` sub-item,
**When** `runbook_claim_story({ runbook_path, story_key, lane, repo_path })` is called,
**Then** the sub-item is updated to `- [x] 🔒 Claimed: {lane} / {YYYY-MM-DD HH:MM}` using the current UTC timestamp, the file is staged, committed, and pushed.

---

**Given** the story's `🔒 Claimed:` sub-item is already `[x]` (already claimed),
**When** `runbook_claim_story` is called,
**Then** the tool returns an error: `"Story {KEY} is already claimed"` and makes no changes.

---

**Given** no `🔒 Claimed:` sub-item exists for the story,
**When** `runbook_claim_story` is called,
**Then** the tool returns an error: `"No Claimed sub-item found for story {KEY}"` and makes no changes.

---

**Given** the git push fails (e.g. network issue),
**When** `runbook_claim_story` is called,
**Then** the file change and local commit are preserved, and the tool returns an error indicating push failure with instructions to retry.

## In Scope

- TypeScript tool file at `~/.config/opencode/tools/runbook_claim_story.ts`
- In-place patch of the `🔒 Claimed:` sub-item for the named story
- Timestamp format: `YYYY-MM-DD HH:MM` (UTC)
- `git add`, `git commit`, `git push` in the repo at `repo_path`
- Commit message format: `chore({lane}): claim {KEY}`

## Out of Scope

- Finding which story to claim (that is `runbook_find_next_story`)
- Checking cross-lane dependencies before claiming
- Jira integration

## Constraints

- Tool placed in `~/.config/opencode/tools/` (global)
- Uses `@opencode-ai/plugin` `tool()` helper
- Inputs: `runbook_path` (absolute), `story_key` (e.g. `ARC-1285`), `lane` (e.g. `core-infrastructure`), `repo_path` (absolute path to git repo root)

## Dependencies

- OpenCode custom tools API (`@opencode-ai/plugin`)
- `runbook_find_next_story` (ARC-1348) — typically called before this tool

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-tools/ARC-1349-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1349-COMPLETION-SUMMARY.md` | TBD |
