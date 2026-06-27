| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Parent Epic | [ARC-1300](https://proalpha.atlassian.net/browse/ARC-1300) ARC: Failure Handling & Escalation |
| Jira | [ARC-1306](https://proalpha.atlassian.net/browse/ARC-1306) |
| Created | 2026-06-27 |

# ARC-1306: Retry failed agent steps up to configurable limit

## Goal
When agent step fails, tool automatically retries up to configured limit. Each retry logged in lane output. After limit exhausted, escalation flow triggered.

## Acceptance Criteria
1. Given step fails and retry count > 0 remaining, when tool retries, then new opencode subprocess spawned for same step.
2. Given retry count reaches configured limit, when last retry fails, then no further auto-retries and escalation triggered.

## In Scope
- Automatic retry, retry counter, retry logging in UI

## Out of Scope
- Manual retry triggered by supervisor

## Dependencies
- ARC-1288 (opencode CLI subprocess integration)
