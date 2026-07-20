| Field | Value |
|-------|-------|
| Type | Story |
| Priority | Medium |
| Status | To Do |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-20 |
| Parent Epic | ARC-1284 — ARC: Core Infrastructure |
| Jira | ARC-1373 |

# ARC-1373: Change worktree base path from /repos to /worktrees

## Goal

Change the hardcoded worktree base path from `/repos/{storyKey}` to `/worktrees/{storyKey}` in all source files and test assertions to avoid conflicts with existing repository checkouts and provide clearer separation between persistent repositories and ephemeral worktrees.

## Acceptance Criteria

**Given** The worktree path construction logic in worktreeGit.ts line 21,
**When** A story worktree is created,
**Then** The path is `/worktrees/{storyKey}`, not `/repos/{storyKey}`.

---

**Given** The worktree path construction logic in laneRunner.ts line 566,
**When** A story worktree is created,
**Then** The path is `/worktrees/{storyKey}`, not `/repos/{storyKey}`.

---

**Given** Test assertions in worktreeGit.test.ts lines 56 and 82,
**When** Worktree tests run,
**Then** All assertions expect `/worktrees/{storyKey}` paths, not `/repos/{storyKey}`.

---

**Given** Test assertions in laneRunner.test.ts line 1644,
**When** Lane runner tests run,
**Then** All assertions expect `/worktrees/{storyKey}` paths, not `/repos/{storyKey}`.

---

**Given** All changes are complete,
**When** The full test suite runs,
**Then** All tests pass with the new `/worktrees/` base path.

## In Scope

- Change hardcoded path from `/repos/{storyKey}` to `/worktrees/{storyKey}` in worktreeGit.ts line 21
- Change hardcoded path from `/repos/{storyKey}` to `/worktrees/{storyKey}` in laneRunner.ts line 566
- Update test assertions in worktreeGit.test.ts lines 56 and 82 to expect `/worktrees/` paths
- Update test assertions in laneRunner.test.ts line 1644 to expect `/worktrees/` paths
- Verify all tests pass after changes

## Out of Scope

- Changes to any other path configuration
- Changes to repository cloning logic
- Changes to git operations beyond worktree path construction
- Documentation updates (will be handled separately if needed)
- Migration of existing worktrees from old to new paths

## Constraints

- Current hardcoded worktree base path is `/repos/{storyKey}` in worktreeGit.ts:21 and laneRunner.ts:566
- Test assertions currently expect `/repos/{storyKey}` in worktreeGit.test.ts:56,82 and laneRunner.test.ts:1644
- All changes must be confined to these four locations plus any additional references discovered during implementation
- No other configuration or environment variables control this path currently

## Dependencies

- None

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-tools/ARC-1373-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1373-COMPLETION-SUMMARY.md` | TBD |