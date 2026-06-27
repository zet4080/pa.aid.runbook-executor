| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Parent Epic | [ARC-1300](https://proalpha.atlassian.net/browse/ARC-1300) ARC: Failure Handling & Escalation |
| Jira | [ARC-1307](https://proalpha.atlassian.net/browse/ARC-1307) |
| Created | 2026-06-27 |

# ARC-1307: Escalate agent level on each retry

## Goal
Each retry uses next agent in configured escalation sequence (e.g. build → senior-coder). If sequence exhausted before retry limit, last agent reused.

## Acceptance Criteria
1. Given escalation sequence [build, senior-coder] and failing step, when first retry runs, then senior-coder used instead of build.
2. Given sequence exhausted but retries remain, when subsequent retries run, then last agent in sequence used.

## In Scope
- Per-retry agent escalation, sequence exhaustion fallback

## Out of Scope
- Dynamic escalation changes mid-session

## Dependencies
- ARC-1306 (retry mechanism must exist)
