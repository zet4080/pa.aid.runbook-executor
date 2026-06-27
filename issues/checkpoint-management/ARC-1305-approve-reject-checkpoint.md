| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Parent Epic | [ARC-1295](https://proalpha.atlassian.net/browse/ARC-1295) ARC: Checkpoint Management |
| Jira | [ARC-1305](https://proalpha.atlassian.net/browse/ARC-1305) |
| Created | 2026-06-27 |

# ARC-1305: Approve or reject checkpoint inline with feedback

## Goal
Supervisor approves or rejects checkpoint in UI. Approval resumes lane. Rejection captures free-text feedback and halts lane — supervisor must explicitly restart or requeue.

## Acceptance Criteria
1. Given pending checkpoint, when supervisor clicks Approve, then lane resumes from next step.
2. Given pending checkpoint, when supervisor clicks Reject and enters feedback, then lane halts and feedback stored.
3. Given rejected checkpoint, when supervisor has not restarted lane, then lane remains halted.

## In Scope
- Approve/reject UI, free-text feedback, halt-on-rejection, explicit restart requirement

## Out of Scope
- Auto-requeue on rejection, partial approval
