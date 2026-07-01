| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-01 |
| Parent Epic | [ARC-1344](https://proalpha.atlassian.net/browse/ARC-1344) ARC: Agent Workflow Improvements |
| Jira | [ARC-1346](https://proalpha.atlassian.net/browse/ARC-1346) |

# ARC-1346: Introduce decision document pattern for async human decisions

## Description

When an agent needs a human decision (approach selection, concept confirmation, scope clarification), it currently stops with an inline question. In non-interactive sessions this blocks progress. Instead the agent must write a structured `decisions/{KEY}-decision.md` file, commit and push it, then surface it as a checkpoint. The human fills in the document and re-triggers the session. On resume, the agent reads the answered document and continues.

## In Scope

- Define the `decisions/` folder convention and document template
- Define the resume protocol: how the agent detects an answered decision document and extracts the decision
- Update `write-implementation-plan` skill (Phase 1 approach selection is the primary trigger)
- Define archival: answered decisions move to `done/decisions/`

## Out of Scope

- Updating all skills that could theoretically need a decision (that is ARC-1347). This story only defines the pattern and updates `write-implementation-plan`.

## Acceptance Criteria

1. Given a HIGH risk story with multiple valid approaches, when the agent reaches Phase 1 of `write-implementation-plan`, then it writes a `decisions/{KEY}-approach-decision.md` file with the options, pros/cons, and recommendation, commits and pushes it, and stops at a checkpoint presenting the file path to the human.
2. Given a decision document exists and is filled in by the human, when the agent resumes the session, then it reads the document, extracts the selected approach, and proceeds to write the implementation plan without asking interactively.
3. Given the `decisions/` folder, when a decision is acted on and the implementation plan is written, then the decision document is moved to `done/decisions/`.

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-workflow/ARC-1346-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1346-COMPLETION-SUMMARY.md` | TBD |
