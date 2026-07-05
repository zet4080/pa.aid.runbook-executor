# ARC-1360 Implementation Plan — `archive_issue` OpenCode Tool

**Issue:** `issues/agent-tools/ARC-1360-tool-archive-issue.md`
**Completion Summary:** `task-completions/ARC-1360-COMPLETION-SUMMARY.md` (TBD)
**Approach:** Single approach — pre-flight validation, atomic move, README append, single git commit
**Owner:** agent-tools
**Date:** 2026-07-05

---

## Scope & Alignment

Create `~/.config/opencode/tools/archive_issue.ts` — atomically moves all three issue artifacts to their respective `done/` subdirectories, updates `done/README.md`, and commits + pushes in one operation. Replaces the six-step manual close sequence in the `close-issue` skill.

Covers all 5 acceptance criteria: full move, missing-plan skip, README creation, pre-flight error guard, and push-failure recovery.

---

## Phase 0 — Exploration Findings

| Question | Answer |
|---|---|
| `archive_issue.ts` already exists? | ❌ No |
| Allowlist configured? | ✅ Already in `opencode-sync-config.sh` and `opencode.json` for `build` + `senior-coder` |
| `done/README.md` format? | ✅ Exists — 4 columns: `Key \| Title \| Completed \| Issue File`, 6 existing rows |
| `done/issues/` structure? | `done/issues/{epic}/` subdirectories |
| `done/implementation_plans/` structure? | `done/implementation_plans/{epic}/` subdirectories |
| `done/task-completions/` structure? | `done/task-completions/{epic}/` subdirectories (epic subdir used) |
| Source `task-completions/` | Flat: `task-completions/{KEY}-COMPLETION-SUMMARY.md` (no epic subdir in source) |
| Source `implementation_plans/` | `implementation_plans/{epic}/{KEY}-*.md` |
| Source `issues/` | `issues/{epic}/{KEY}-*.md` |
| Test runner | `node --experimental-strip-types --test` |

---

## Files

| Action | Path |
|---|---|
| CREATE | `~/.config/opencode/tools/archive_issue.ts` |
| CREATE | `~/.config/opencode/tools/tests/archive_issue.test.ts` |
| ALREADY DONE | `opencode-sync-config.sh` — `archive_issue` already in `build` + `senior-coder` allowlists |
| ALREADY DONE | `~/.config/opencode/opencode.json` — `archive_issue` already in `build` + `senior-coder` allowlists |

---

## Return Type

```typescript
export interface ArchiveIssueResult {
  moved: string[];           // relative paths of files moved
  readmeUpdated: boolean;
  hash: string;              // git commit hash
  pushed: boolean;
  error?: string;
}
```

---

## Exported Helpers (for testability)

### `findArtifact(dir: string, keyPrefix: string): string | null`
- Calls `readdirSync(dir)` (returns `null` if dir doesn't exist)
- Returns absolute path of first entry starting with `${keyPrefix}-` and ending with `.md`
- Returns `null` if no match

### `extractTitle(content: string): string`
- Scans lines for first line matching `/^# /`
- Returns text after `# ` (trimmed)
- Falls back to `"(no title)"` if no heading found

### `ensureReadmeTable(readmePath: string): void`
- Reads file if it exists; creates from scratch if missing
- If no `| Key |` table header line present, appends:
  ```
  | Key | Title | Completed | Issue File |
  |-----|-------|-----------|------------|
  ```
- No-op if table already present

### `appendReadmeRow(readmePath: string, key: string, title: string, date: string, issuePath: string): void`
- Appends: `| {key} | {title} | {date} | \`{issuePath}\` |`

---

## Implementation Steps

### Step 1: Create `~/.config/opencode/tools/archive_issue.ts`

**File structure:**
```
Header docblock → imports → ArchiveIssueResult interface →
findArtifact() → extractTitle() → ensureReadmeTable() → appendReadmeRow() →
execute() → export default tool({...})
```

**`execute(args, context)` logic:**

1. **Resolve paths:**
   - `issueFile = findArtifact(join(args.repo_path, 'issues', args.epic), args.issue_key)`
   - `planFile = findArtifact(join(args.repo_path, 'implementation_plans', args.epic), args.issue_key)` (optional)
   - `summaryFile = findArtifact(join(args.repo_path, 'task-completions'), args.issue_key)`

2. **Pre-flight validation (AC4):** Collect missing required files (`issueFile`, `summaryFile`). If any missing, return `{ error: "Missing required files: ..." }` — no side effects.

3. **Create destination directories:**
   ```
   done/issues/{epic}/
   done/implementation_plans/{epic}/   (only if planFile exists)
   done/task-completions/{epic}/
   ```

4. **Move files** using `fs.renameSync`:
   - Issue → `done/issues/{epic}/{filename}`
   - Plan → `done/implementation_plans/{epic}/{filename}` (AC2: skip if null)
   - Summary → `done/task-completions/{epic}/{filename}`
   - Track moved paths in `moved[]`

5. **Update README:**
   - `ensureReadmeTable(done/README.md)` (AC3)
   - `extractTitle(issueFileContent)` for the title
   - `appendReadmeRow(...)` with today's date

6. **Git commit:**
   ```bash
   git -C {repo_path} add -A
   git -C {repo_path} commit -m "docs(close): archive {issue_key}"
   ```
   Extract hash from commit stdout.

7. **Git push (separate try/catch — AC5):**
   - On success: `pushed: true`
   - On failure: `pushed: false, error: "Push failed: ..."` — commit is preserved

8. Return `{ moved, readmeUpdated: true, hash, pushed }`

**Tool descriptor:**
```typescript
export default tool({
  description: 'Archive a completed issue: move issue file, implementation plan, and completion summary to done/, update done/README.md, and commit+push in one operation.',
  args: {
    repo_path: tool.schema.string().describe('Absolute path to the planning repo, e.g. /repos/pa.aid.runbook-executor'),
    issue_key: tool.schema.string().describe('Issue key, e.g. ARC-1285'),
    epic: tool.schema.string().describe('Epic folder name, e.g. core-infrastructure'),
  },
  async execute(args, _context) { ... },
});
```

### Step 2: Create `~/.config/opencode/tools/tests/archive_issue.test.ts`

10 test scenarios using `node:test` + `assert/strict`. Tests helpers directly; `execute()` integration scenarios use temp dirs.

| # | Scenario | AC |
|---|---|---|
| 1 | `findArtifact` — matching file found | helper |
| 2 | `findArtifact` — dir missing → `null` | helper |
| 3 | `findArtifact` — no matching file → `null` | helper |
| 4 | `extractTitle` — extracts first `# ` heading | helper |
| 5 | `extractTitle` — no heading → `"(no title)"` | helper |
| 6 | `ensureReadmeTable` — creates header when file missing | AC3 |
| 7 | `ensureReadmeTable` — no-op when table already present | AC3 |
| 8 | `execute()` — issue file missing → error, no moves | AC4 |
| 9 | `execute()` — plan missing → skipped, issue+summary moved | AC2 |
| 10 | `execute()` — all 3 exist → all moved, README updated | AC1 |

Note: Git push (AC5) is tested at the `execute()` level but a real git repo is set up in the temp dir for test 10.

### Step 3: Update allowlist

Already done — `archive_issue` is already in both `opencode-sync-config.sh` and `~/.config/opencode/opencode.json` for `build` and `senior-coder`.

### Step 4: Commit to `pa.aid.config.md`
```
feat(agent-tools): implement archive_issue OpenCode tool (ARC-1360)
```

### Step 5: Write completion summary and check off runbook

---

## Testing & Validation

```bash
node --experimental-strip-types --test ~/.config/opencode/tools/tests/archive_issue.test.ts
grep archive_issue ~/.config/opencode/opencode.json
```

All 10 tests pass. Allowlist grep returns hits in `build` and `senior-coder` sections.

---

## Acceptance Criteria Mapping

| AC | Covered by |
|---|---|
| AC1: All 3 artifacts moved + README + commit+push | `execute()` happy path; test 10 |
| AC2: Missing plan skipped silently | `planFile` null check; test 9 |
| AC3: README table created if missing | `ensureReadmeTable()`; tests 6+7 |
| AC4: Missing required file → error before any changes | pre-flight validation; tests 8 |
| AC5: Push failure → preserve commit, return `pushed: false` | separate push try/catch; described in plan |

---

## Commit Messages

| Repo | Message |
|---|---|
| `pa.aid.config.md` | `feat(agent-tools): implement archive_issue OpenCode tool (ARC-1360)` |
| `pa.aid.runbook-executor` | `docs(agent-tools): add ARC-1360 implementation plan` |
| `pa.aid.runbook-executor` | `chore(agent-tools): complete ARC-1360 archive_issue tool` |

---

## Risks

| Risk | Severity | Mitigation |
|---|---|---|
| `fs.renameSync` fails across filesystems | Low | All moves within same repo — no cross-device |
| `done/task-completions/` source is flat (no epic subdir) | Low | Handled: `findArtifact` called on flat `task-completions/` dir |
| Git commit hash extraction from stdout | Low | Parse `git commit` stdout for short hash; fallback to empty string |
| Push failure in CI/CD context | Low | AC5 path: commit preserved, error reported |
