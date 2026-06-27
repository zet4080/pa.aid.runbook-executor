| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Parent Epic | [ARC-1284](https://proalpha.atlassian.net/browse/ARC-1284) ARC: Core Infrastructure |
| Jira | [ARC-1287](https://proalpha.atlassian.net/browse/ARC-1287) |
| Created | 2026-06-27 |

# ARC-1287: Implement sidecar state file

## Goal
Maintains a sidecar JSON state file for session persistence across restarts. Runbook markdown is always ground truth — sidecar reconciled against markdown on startup.

## Acceptance Criteria
1. Given active session, when tool restarts, then state restored from sidecar (runbooks, progress, queue).
2. Given sidecar and markdown disagree on checkbox state, when tool starts, then markdown state takes precedence.

## In Scope
- Sidecar file read/write, startup reconciliation with markdown ground truth

## Out of Scope
- Multi-session state, cloud sync, external state stores
