| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Parent Epic | [ARC-1284](https://proalpha.atlassian.net/browse/ARC-1284) ARC: Core Infrastructure |
| Jira | [ARC-1288](https://proalpha.atlassian.net/browse/ARC-1288) |
| Created | 2026-06-27 |

# ARC-1288: Integrate opencode CLI subprocess with JSON event stream

## Goal
Invokes `opencode run --format json` as subprocess per agent step. Reads JSON event stream from stdout, detects completion/failure via exit code and event fields. Fresh session per call.

## Acceptance Criteria
1. Given agent step, when opencode run invoked, then stdout consumed as JSON event stream and rendered live.
2. Given non-zero exit or error event, then step marked failed and failure flow triggered.
3. Given zero exit code, then step marked succeeded.

## In Scope
- Subprocess spawning, JSON stream consumption, exit code detection, fresh-session-per-call

## Out of Scope
- opencode serve mode, persistent sessions, credential management (inherited from environment)

## Constraints
- Agent invocation via opencode CLI subprocess — fresh session per agent call
- Uses global opencode config (~/.config/opencode/) — no per-repo override
- Credentials inherited from setup.env
