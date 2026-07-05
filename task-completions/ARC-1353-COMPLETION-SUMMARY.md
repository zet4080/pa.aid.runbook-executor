# ARC-1353 Completion Summary — `get_current_issue` OpenCode Tool

**Story:** ARC-1353 — Implement `get_current_issue` OpenCode tool
**Lane:** agent-tools
**Date:** 2026-07-05
**Status:** Complete

---

## What Was Built

A read-only OpenCode tool at `~/.config/opencode/tools/get_current_issue.ts` that:
1. Reads the current branch name from `context.worktree` via `git branch --show-current`
2. Extracts a Jira-style issue key (`[A-Z]+-[0-9]+`) from the start of the branch name
3. Recursively searches `{planning_repo_path}/issues/` for a matching `${issueId}-*.md` file
4. Reads and parses the `## Acceptance Criteria` section from the issue file
5. Returns `{ issueId, filePath, acceptanceCriteria[] }` — or graceful results with warnings for all error cases

A companion test file was created at `~/.config/opencode/tools/tests/get_current_issue.test.ts` with 8 unit test scenarios.

---

## Test Results

All 8 tests pass:

```
✔ 1. extractIssueId — exact key "ARC-1285" returns "ARC-1285" (1.188848ms)
✔ 2. extractIssueId — "ARC-1285-some-description" returns "ARC-1285" (0.117284ms)
✔ 3. extractIssueId — "main" returns null (0.064487ms)
✔ 4. extractIssueId — "feature/ARC-1285" returns null (key not at start) (0.058168ms)
✔ 5. parseAcceptanceCriteria — content with AC section returns non-empty array (0.254037ms)
✔ 6. parseAcceptanceCriteria — content with no AC heading returns [] (1.083511ms)
✔ 7. findIssueFile — temp dir with matching file returns absolute path (1.041722ms)
✔ 8. findIssueFile — temp dir with no matching file returns null (0.235999ms)
ℹ pass 8 / fail 0
```

---

## Acceptance Criteria Mapping

| AC | Evidence |
|---|---|
| AC1: Valid branch + file found → `{ issueId, filePath, acceptanceCriteria[] }` | `execute()` happy path (steps 2–7); tests 1, 5, 7 confirm each helper |
| AC2: Valid branch + no file found → `filePath: null` + warning | `execute()` null-file branch (step 4); test 8 confirms `findIssueFile` returns `null` |
| AC3: Branch doesn't match `[A-Z]+-[0-9]+` → `issueId: null` + warning | `extractIssueId()` returning `null` (step 3); tests 3, 4 confirm non-matching branches |
| AC4: Planning repo path invalid → error with clear message | `validatePlanningRepo()` error return (step 1); checks both path existence and `.git` presence |

---

## Files Created

| File | Purpose |
|---|---|
| `~/.config/opencode/tools/get_current_issue.ts` | Tool implementation |
| `~/.config/opencode/tools/tests/get_current_issue.test.ts` | 8 unit test scenarios |

---

## Allowlist Status

`get_current_issue` was already pre-configured in:
- `opencode-sync-config.sh` in `pa.aid.wsl-setup.sh`
- Live `~/.config/opencode/opencode.json` — 3 agents: `build`, `senior-coder`, `plan`

No allowlist changes were needed.

---

## Commits

| Repo | Branch | Commit | Message |
|---|---|---|---|
| `pa.aid.config.md` | `ARC-1353` | `714a1ed` | `feat(agent-tools): implement get_current_issue OpenCode tool (ARC-1353)` |
| `pa.aid.runbook-executor` | `main` | (see planning commits) | `docs(agent-tools): add ARC-1353 implementation plan` |

**PR:** https://bitbucket.org/proalpha/pa.aid.config.md/pull-requests/new?source=ARC-1353&t=1
