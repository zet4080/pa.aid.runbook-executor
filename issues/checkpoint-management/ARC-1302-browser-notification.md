| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Parent Epic | [ARC-1295](https://proalpha.atlassian.net/browse/ARC-1295) ARC: Checkpoint Management |
| Jira | [ARC-1302](https://proalpha.atlassian.net/browse/ARC-1302) |
| Created | 2026-06-27 |

# ARC-1302: Create Bitbucket PR and display persistent [Resume] card on checkpoint hit

## Goal
When a lane hits a checkpoint, the executor creates a git branch, pushes relevant artifacts, opens a Bitbucket Pull Request via the Bitbucket MCP endpoint, and displays a persistent [Resume] card in the session UI — all within 5 seconds of the checkpoint hit. The [Resume] card shows the PR link, checkpoint type, and runbook/lane context.

## Acceptance Criteria
1. Given lane hits checkpoint, within 5 seconds: git branch created, relevant artifacts pushed, and Bitbucket PR opened via Bitbucket MCP.
2. Given PR created, then [Resume] card displayed in session UI with: PR URL, checkpoint type (🔴/🟡/🟢), runbook name, and lane identifier.
3. Given [Resume] card displayed, then card remains visible in session UI until supervisor clicks [Resume] or lane is manually cancelled.

## In Scope
- Bitbucket MCP PR creation, git branch creation, artifact push, [Resume] card display in session UI

## Out of Scope
- Browser Notification API (separate story if needed), email notifications, Slack notifications

## Dependencies
- ARC-1301 (checkpoint detection and lane pause)
- Bitbucket MCP endpoint available as `bitbucket`