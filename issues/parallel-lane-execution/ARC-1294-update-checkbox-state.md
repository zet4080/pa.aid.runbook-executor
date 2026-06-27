| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Parent Epic | [ARC-1290](https://proalpha.atlassian.net/browse/ARC-1290) ARC: Parallel Lane Execution |
| Jira | [ARC-1294](https://proalpha.atlassian.net/browse/ARC-1294) |
| Created | 2026-06-27 |

# ARC-1294: Update runbook checkbox state in real time

## Goal
As steps complete, tool updates corresponding checkboxes in runbook markdown immediately. Markdown always ground truth — checkbox state reflects actual execution state at all times.

## Acceptance Criteria
1. Given step completes, when tool updates state, then markdown checkbox updated from [ ] to [x] within 1 second.
2. Given checkpoint hit, when lane pauses, then markdown reflects paused state (checkbox not yet checked).
3. Given tool crashes mid-execution, when restarted, then markdown accurately reflects completed steps.

## In Scope
- Real-time markdown checkbox updates, crash-safe state writing

## Out of Scope
- Modifying runbook structure beyond checkboxes
