| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Parent Epic | [ARC-1295](https://proalpha.atlassian.net/browse/ARC-1295) ARC: Checkpoint Management |
| Jira | [ARC-1304](https://proalpha.atlassian.net/browse/ARC-1304) |
| Created | 2026-06-27 |

# ARC-1304: Read PR comment threads on Resume and address each as a required action item

## Goal
When supervisor clicks [Resume], executor reads all unresolved comment threads on the checkpoint PR via Bitbucket MCP. Every unresolved comment is treated as a required action item. Executor addresses each comment, commits the result, replies on the PR thread with the commit SHA, resolves the thread, then re-sends the [Resume] card. Lane proceeds only when [Resume] is clicked with zero unresolved threads.

## Acceptance Criteria
1. Given [Resume] clicked with unresolved PR comment threads, then executor reads all unresolved threads via Bitbucket MCP.
2. Given unresolved comment thread, then executor treats it as a required action item, addresses it, and commits the result.
3. Given comment addressed, then executor replies on the PR thread with "Addressed in commit {sha}" and resolves the thread via Bitbucket MCP.
4. Given all comments addressed, then executor re-sends the [Resume] card with message "Agent addressed N comments — [view changes]" and waits for [Resume] again.
5. Given [Resume] clicked with zero unresolved comment threads, then lane proceeds to next step.

## In Scope
- Bitbucket MCP comment thread reading, thread resolution, commit-and-reply flow, re-notify loop

## Out of Scope
- Auto-proceeding without re-notify after addressing comments, partial comment addressing (all unresolved threads must be addressed)

## Dependencies
- ARC-1302 (PR creation and [Resume] card)
- ARC-1305 ([Resume] button and click handler)