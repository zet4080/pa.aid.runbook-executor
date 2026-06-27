| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Parent Epic | [ARC-1309](https://proalpha.atlassian.net/browse/ARC-1309) ARC: Session History |
| Jira | [ARC-1310](https://proalpha.atlassian.net/browse/ARC-1310) |
| Created | 2026-06-27 |

# ARC-1310: Persist active session state across restarts

## Goal
One active session per planning repo. Session state persisted continuously to sidecar file — restarting tool restores session without data loss.

## Acceptance Criteria
1. Given active session, when tool restarts, then session restored: runbooks, progress, checkpoints, config all present.
2. Given two simultaneous starts on same repo, when second starts, then it detects active session and warns supervisor.

## In Scope
- Session persistence, restart recovery, single-session-per-repo enforcement

## Out of Scope
- Multiple simultaneous sessions, cloud session sync

## Dependencies
- ARC-1287 (sidecar state file)
