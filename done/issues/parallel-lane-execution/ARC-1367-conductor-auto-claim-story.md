| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | Completed |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-06 |
| Last Updated | 2026-07-07 |
| Parent Epic | ARC-1290 — ARC: Parallel Lane Execution |
| Jira | ARC-1367 |

# ARC-1367: Conductor: auto-claim story in runbook before dispatching agent

## Goal

When the conductor's `laneRunner` begins executing a story it should automatically mark the story's `🔒 Claimed:` sub-item as done in the runbook markdown and commit the change to the planning repo `main` branch — before the agent subprocess is dispatched. This eliminates the need for agents to call `runbook_claim_story` as a manual ceremony step and prevents parallel sessions from picking up the same story.

## Problem

Agents currently must call `runbook_claim_story` explicitly at the start of every story. This is pure mechanical ceremony: the conductor already knows which story it is about to execute, so it can perform the claim atomically as part of story dispatch — with no agent involvement required.

## Acceptance Criteria

**Given** The conductor `laneRunner` identifies the next unclaimed story in a runbook,
**When** It is about to dispatch the agent subprocess for that story,
**Then** The story's `🔒 Claimed:` checkbox is set to `[x]` with an ISO-8601 timestamp and the lane name in the planning repo runbook before the agent process starts.

---

**Given** The claim has been written to the runbook markdown,
**When** The claim write completes,
**Then** The change is committed and pushed to `main` in the planning repo using the existing auto-commit pattern from `checkpointGit.ts`.

---

**Given** A story is already claimed (e.g., conductor resuming after a crash),
**When** The lane runner encounters the story again,
**Then** The claim step is skipped without error (idempotent).

---

**Given** The auto-claim is implemented,
**When** An agent session starts,
**Then** The `start-execution-session` skill no longer instructs the agent to call `runbook_claim_story` as a ceremony step.

## In Scope

- Modify `laneRunner.ts` to invoke claim logic (using the same markdown-writer pattern as `markStepCheckedInMarkdown`) at story execution start
- Timestamp format: ISO-8601 datetime string
- Commit message: `chore({lane}): claim {KEY}` pushed to planning repo `main`
- Idempotency: no-op if claim row already checked
- Update `start-execution-session` skill to remove manual `runbook_claim_story` instruction
- Unit test: auto-claim fires before first agent prompt is built

## Out of Scope

- Changes to the `runbook_claim_story` MCP tool itself (it remains available for edge-case manual use)
- Claiming batch (MEDIUM/LOW) stories differently — same logic applies
- Any changes to the runbook file format

## Constraints

- Must use the existing `markdownWriter` utilities — do not duplicate markdown-manipulation logic
- Commit must land on planning repo `main`, never on the feature branch
- Must not delay agent dispatch: the commit is fire-and-forget if push fails (log warning, continue)

## Dependencies

- ARC-1349 (runbook_claim_story tool — reference implementation of claim logic)

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-tools/ARC-1367-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1367-COMPLETION-SUMMARY.md` | TBD |