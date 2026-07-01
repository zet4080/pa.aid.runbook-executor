| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-01 |
| Parent Epic | [ARC-1344](https://proalpha.atlassian.net/browse/ARC-1344) ARC: Agent Workflow Improvements |
| Jira | [ARC-1347](https://proalpha.atlassian.net/browse/ARC-1347) |

# ARC-1347: Update planning skills for non-interactive checkpoint model

## Description

With the decision document pattern established (ARC-1346), all planning skills that currently use inline "STOP — answer these questions" patterns must be updated to use the decision document pattern instead. This includes `start-execution-session` (all STOP points), `execute-implementation-plan` (scope change, architectural decision triggers), and the Human Escalation Triggers section of `HOW-THIS-WORKS.md` in planning repos.

## In Scope

- `start-execution-session` skill — all STOP points replaced with decision document pattern
- `execute-implementation-plan` skill — scope change and architectural decision triggers
- `HOW-THIS-WORKS.md` Human Escalation Triggers section — update to reference decision document protocol
- `write-completion-summary` skill — if it contains any inline STOP patterns

## Out of Scope

- `write-implementation-plan` skill (handled in ARC-1346)
- Domain-specific skills outside the planning workflow (e.g., `ansible-local-test`, `pr-review`)

## Dependencies

- ARC-1346 (decision document pattern) must be complete before this story begins

## Acceptance Criteria

1. Given any planning skill with a "STOP — ask human" pattern, when the agent hits a decision point during a non-interactive session, then it writes a decision document, commits it, surfaces the file path as a checkpoint, and does not emit an inline question.
2. Given `HOW-THIS-WORKS.md` Human Escalation Triggers table, when any trigger fires, then the documented response is to write a decision document and stop at a checkpoint, not to ask interactively.
3. Given all updated skills, when a human reviews them, then no skill contains inline "STOP — answer X, Y, Z" question patterns.

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-workflow/ARC-1347-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1347-COMPLETION-SUMMARY.md` | TBD |
