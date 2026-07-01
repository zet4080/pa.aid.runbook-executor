| Field | Value |
|-------|-------|
| Type | Story |
| Priority | Low |
| Status | To Do |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-01 |
| Parent Epic | [ARC-1344](https://proalpha.atlassian.net/browse/ARC-1344) ARC: Agent Workflow Improvements |
| Jira | [ARC-1345](https://proalpha.atlassian.net/browse/ARC-1345) |

# ARC-1345: Fix runbook-commit workflow — no PR for runbook artifacts

## Description

Skills currently imply or require PRs for runbook changes (claim, checkoff, wave gate). These changes commit directly to `main`. Only code and generated artifacts need PRs. Remove or correct all skill language that implies PRs for runbook artifacts.

## In Scope

- `start-execution-session` skill — remove any PR implication from claim/checkoff commit steps
- `write-completion-summary` skill — remove any PR implication from completion summary commit steps
- `HOW-THIS-WORKS.md` in any planning repo — clarify the runbook artifact lifecycle

## Out of Scope

- Changes to how code artifacts (implementation plans, application code) are handled
- Any non-planning-workflow skills

## Acceptance Criteria

1. Given the `start-execution-session` skill, when it describes committing a claim or checkoff, then no PR creation is mentioned or implied.
2. Given the `write-completion-summary` skill, when it describes committing a completion summary to the planning repo, then no PR is required for that commit.
3. Given `HOW-THIS-WORKS.md`, when it describes the runbook artifact lifecycle, then it clearly distinguishes: runbook changes → commit to main directly; code artifacts → PR.

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-workflow/ARC-1345-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1345-COMPLETION-SUMMARY.md` | TBD |
