| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Parent Epic | [ARC-1295](https://proalpha.atlassian.net/browse/ARC-1295) ARC: Checkpoint Management |
| Jira | [ARC-1301](https://proalpha.atlassian.net/browse/ARC-1301) |
| Created | 2026-06-27 |

# ARC-1301: Detect checkpoint markers and pause lane execution

## Goal
When executor reaches a step with 🔴, 🟡, or 🟢 marker, the lane immediately pauses and transitions to blocked state, awaiting PR review and supervisor [Resume]. All three marker types trigger pause.

## Acceptance Criteria
1. Given step with 🔴, 🟡, or 🟢 marker, when executor reaches it, then lane pauses and transitions to blocked status.
2. Given blocked lane, when supervisor views dashboard, then lane shows blocked with checkpoint type indicated and a [Resume] card is expected to follow from the PR creation step (ARC-1302).

## In Scope
- Marker detection (all three types), lane pause, blocked status transition

## Out of Scope
- Auto-resolving checkpoints, inferring level from content

## Dependencies
- ARC-1291 (sequential step execution)