```
markdown
# ARC-1353 Implementation Plan — `get_current_issue` OpenCode Tool

**Issue:** `issues/agent-tools/ARC-1353-tool-get-current-issue.md`
**Lane:** agent-tools
**Date:** 2026-07-05

---

## Goal

Create a read-only OpenCode tool at `~/.config/opencode/tools/get_current_issue.ts` that:
1. Reads the current branch name from the active session's git root (`context.worktree`)
2. Extracts an issue key matching `[A-Z]+-[0-9]+` from the branch name
3. Searches the planning repo (`planning_repo_path` param) for a matching issue file
4. Returns parsed acceptance criteria from that file

Eliminates silent AC validation failures in `local-code-review`.

---

## Phase 0: Exploration Findings

| Question | Answer |
|---|---|
| `get_current_issue.ts` already exists? | ❌ No — not in `~/.config/opencode/tools/` |
| Allowlist already configured? | ✅ Yes — `get_current_issue` already in `opencode-sync-config.sh` AND live `opencode.json` for `build`, `senior-coder`, `plan` agents |
| `lib/parser.ts` needed? | ❌ No — tool reads issue files, not runbooks |
| Test runner? | `node --experimental-strip-types --test` |
| Issue file AC format? | Lines under `## Acceptance Criteria` until next `## ` heading |
| `context.worktree` for git branch? | ✅ Confirmed — `git -C ${context.worktree} branch --show-current` |

---

## Files

| Action | File |
|---|---|
| CREATE | `~/.config/opencode/tools/get_current_issue.ts` |
| CREATE | `~/.config/opencode/tools/tests/get_current_issue.test.ts` |
| VERIFY (no edit) | `~/.config/opencode/opencode.json` — allowlist already contains `get_current_issue` |

---

## Return Type

```typescript
export interface GetCurrentIssueResult {
  issueId: string | null;
  filePath: string | null;
  acceptanceCriteria: string[];
  warning?: string;
}
```

---

## Exported Helpers (for testability)

### `extractIssueId(branchName: string): string | null`
- Applies regex `/^([A-Z]+-\d+)/` to extract issue key from the start of branch name
- Returns captured group if matched, `null` otherwise
- `