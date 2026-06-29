| Field | Value |
|-------|-------|
| Type | Story |
| Priority | Medium |
| Status | Completed |\n| Last Updated | 2026-06-29 |
| Parent Epic | [ARC-1289](https://proalpha.atlassian.net/browse/ARC-1289) ARC: Session Setup & Runbook Selection |
| Jira | [ARC-1314](https://proalpha.atlassian.net/browse/ARC-1314) |
| Created | 2026-06-27 |

# ARC-1314: Expand runbook detail on demand

## Goal
Supervisor expands any runbook card to see full progress tree: all waves, stories, steps with checkbox state, next pending step highlighted. Collapsing returns to summary view.

## Acceptance Criteria
1. Given collapsed runbook card, when supervisor clicks expand, then all waves and steps shown with checkbox state.
2. Given expanded runbook, when supervisor collapses, then returns to summary view without losing state.

## In Scope
- Expand/collapse UI, full step tree rendering, next-step highlighting

## Out of Scope
- Editing step content, inline step execution from dashboard
