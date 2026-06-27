| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Parent Epic | [ARC-1289](https://proalpha.atlassian.net/browse/ARC-1289) ARC: Session Setup & Runbook Selection |
| Jira | [ARC-1315](https://proalpha.atlassian.net/browse/ARC-1315) |
| Created | 2026-06-27 |

# ARC-1315: Select runbooks to include in session

## Goal
Supervisor selects which runbooks to automate in current session. Unselected runbooks remain visible but not executed. Selection changeable before session starts.

## Acceptance Criteria
1. Given dashboard, when supervisor selects subset and starts session, then only selected runbooks execute.
2. Given deselected runbook, when session starts, then that runbook remains idle.

## In Scope
- Multi-select UI, session-scoped runbook list

## Out of Scope
- Adding/removing runbooks from repo, mid-session selection changes
