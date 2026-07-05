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
- `"ARC-1285"` → `"ARC-1285"` ✅
- `"ARC-1285-some-description"` → `"ARC-1285"` ✅
- `"main"`, `"HEAD"`, `"feature/ARC-1285"` → `null` (key must START the branch name)

### `findIssueFile(planningRepoPath: string, issueId: string): string | null`
- Recursively walks `{planningRepoPath}/issues/` using `readdirSync` + `statSync`
- Matches files where filename starts with `${issueId}-` and ends with `.md`
- Returns absolute path of first match, `null` if not found
- Returns `null` on any filesystem error (no throw)

### `parseAcceptanceCriteria(content: string): string[]`
- Finds line matching `/^## Acceptance Criteria/`
- Collects non-empty lines until next `/^## /` heading or end of file
- Excludes `---` dividers and blank lines
- Returns `[]` if section not found

### `validatePlanningRepo(path: string): string | null`
- `existsSync(path)` false → returns `"Planning repo path does not exist: ${path}"`
- `existsSync(join(path, '.git'))` false → returns `"Path is not a git repository: ${path}"`
- Returns `null` on success

---

## Implementation Steps

### Step 1: Create `~/.config/opencode/tools/get_current_issue.ts`

Structure (follows standard tool pattern):

```
Header docblock → imports → Types → helper functions → execute() → export default tool({...})
```

**`execute(args, context)` logic:**
1. Call `validatePlanningRepo(args.planning_repo_path)` — if non-null, return `JSON.stringify({ error: errorMsg })`
2. Run `Bun.$\`git -C ${context.worktree} branch --show-current\`` — capture `stdout.trim()` as `branchName`
3. Call `extractIssueId(branchName)` — if null, return result with `issueId: null`, `filePath: null`, `acceptanceCriteria: []`, `warning: "Branch '${branchName}' does not match [A-Z]+-[0-9]+ pattern"`
4. Call `findIssueFile(args.planning_repo_path, issueId)` — if null, return result with `issueId`, `filePath: null`, `acceptanceCriteria: []`, `warning: "No issue file found for ${issueId} in ${args.planning_repo_path}/issues/"`
5. Read file: `const content = await Bun.file(filePath).text()`
6. Call `parseAcceptanceCriteria(content)`
7. Return `JSON.stringify({ issueId, filePath, acceptanceCriteria })`

**Tool descriptor:**

```typescript
export default tool({
  description: 'Resolve the current issue from the active git branch, find the issue file in the planning repo, and return the issue ID, file path, and parsed acceptance criteria.',
  args: {
    planning_repo_path: tool.schema.string().describe('Absolute path to the planning repo, e.g. /repos/pa.aid.runbook-executor'),
  },
  async execute(args, context) { ... },
});
```

### Step 2: Create `~/.config/opencode/tools/tests/get_current_issue.test.ts`

8 test scenarios using `node:test`. Imports helpers directly from `../get_current_issue.ts`.

| # | Function | Scenario | Expected |
|---|---|---|---|
| 1 | `extractIssueId` | `"ARC-1285"` | `"ARC-1285"` |
| 2 | `extractIssueId` | `"ARC-1285-some-description"` | `"ARC-1285"` |
| 3 | `extractIssueId` | `"main"` | `null` |
| 4 | `extractIssueId` | `"feature/ARC-1285"` | `null` |
| 5 | `parseAcceptanceCriteria` | Issue content with AC section | Non-empty array |
| 6 | `parseAcceptanceCriteria` | Content with no AC heading | `[]` |
| 7 | `findIssueFile` | Temp dir with matching file | Returns absolute path |
| 8 | `findIssueFile` | Temp dir with no matching file | `null` |

Fixture helper: `writeTmpIssueDir(files: Record<string, string>): string` — writes temporary `issues/subfolder/` structure.

### Step 3: Verify allowlist

```bash
grep get_current_issue ~/.config/opencode/opencode.json
```

Expected: 3 matches (`build`, `senior-coder`, `plan` agents). No edit needed.

### Step 4: Commit to `pa.aid.config.md`

```bash
git -C ~/.config/opencode commit -am "feat(agent-tools): implement get_current_issue OpenCode tool (ARC-1353)"
```

### Step 5: Write completion summary and check off runbook

- Write `task-completions/ARC-1353-COMPLETION-SUMMARY.md`
- Check off all sub-steps and story checkbox in `docs/plans/runbook-agent-tools.md`

---

## Testing & Validation

```bash
# Unit tests (8 scenarios)
node --experimental-strip-types --test ~/.config/opencode/tools/tests/get_current_issue.test.ts

# Type check
node --experimental-strip-types --check ~/.config/opencode/tools/get_current_issue.ts

# Allowlist verification
grep get_current_issue ~/.config/opencode/opencode.json
```

---

## Acceptance Criteria Mapping

| AC | Implementing Step |
|---|---|
| AC1: Valid branch + file found → `{ issueId, filePath, acceptanceCriteria[] }` | `execute()` happy path; test scenarios 1, 5, 7 |
| AC2: Valid branch + no file → `filePath: null` + warning | `execute()` null-file branch; test scenario 8 |
| AC3: Branch doesn't match pattern → `issueId: null` + warning | `extractIssueId()` null; test scenarios 3, 4 |
| AC4: Planning repo path invalid → error with clear message | `validatePlanningRepo()` error; covered by unit test |

---

## Commit Messages

| Repo | Message |
|---|---|
| `pa.aid.config.md` | `feat(agent-tools): implement get_current_issue OpenCode tool (ARC-1353)` |
| `pa.aid.runbook-executor` | `docs(agent-tools): add ARC-1353 implementation plan` |
| `pa.aid.runbook-executor` | `chore(agent-tools): check off ARC-1353` |

---

## Risks

| Risk | Severity | Mitigation |
|---|---|---|
| `context.worktree` returns unexpected path when agent runs in planning repo | Low | Document in tool description; callers must use in correct git context |
| `git branch --show-current` fails in detached HEAD state | Low | Returns `null` issueId with warning — same AC3 path |
| AC section formatting deviates in future issue files | Low | `parseAcceptanceCriteria` returns `[]` safely — graceful AC2 path |
