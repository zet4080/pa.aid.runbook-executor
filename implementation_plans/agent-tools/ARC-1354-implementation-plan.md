Content to write (verbatim):

---
# ARC-1354 Implementation Plan — `run_tests` OpenCode Tool

**Issue:** `issues/agent-tools/ARC-1354-tool-run-tests.md`
**Completion Summary:** `task-completions/ARC-1354-COMPLETION-SUMMARY.md` (TBD)
**Approach:** Single approach — pattern established (mirrors `get_current_issue.ts`)
**Owner:** agent-tools
**Date:** 2026-07-05

---

## Scope & Alignment

Create `~/.config/opencode/tools/run_tests.ts` — a self-contained OpenCode tool that auto-detects the test framework in a project directory and runs its test suite. Returns `{ passed, framework, output, error? }`.

Covers all 6 acceptance criteria: four framework-specific happy paths (npm, cargo, pytest, go), the no-framework error path, and the non-zero-exit `passed: false` path.

---

## Phase 0 — Exploration Findings

| Question | Answer |
|---|---|
| `run_tests.ts` already exists? | ❌ No |
| Test file already exists? | ❌ No |
| Allowlist already configured? | ✅ Yes — `run_tests` already in `opencode-sync-config.sh` for `debug`, `build`, `senior-coder` agents |
| `opencode.json` update needed? | No |
| `lib/` imports needed? | ❌ No — self-contained |
| Test runner | `node --experimental-strip-types --test` |
| Subprocess pattern | `Bun.$` with `.cwd()` fluent method |
| `context.worktree` usage | Default directory when `args.directory` omitted |

---

## Assumptions & Dependencies

- `Bun.$` is available in the OpenCode runtime (confirmed by existing tools)
- `Bun.$` supports `.cwd(dir)` fluent method for setting working directory
- `existsSync` from Node `fs` is available for framework detection
- `run_tests` is already in agent allowlists — no allowlist edit required
- Framework detection by file *presence* only — no parsing of file content required

---

## Files

| Action | Path |
|---|---|
| CREATE | `~/.config/opencode/tools/run_tests.ts` |
| CREATE | `~/.config/opencode/tools/tests/run_tests.test.ts` |
| VERIFY (no edit) | `opencode-sync-config.sh` — `run_tests` already allowlisted |

---

## Return Type

```typescript
export interface RunTestsResult {
  passed: boolean | null;
  framework: string | null;
  output: string;
  error?: string;
}
```

---

## Exported Helpers (for testability)

### `detectFramework(directory: string): { framework: string; command: string } | null`

Checks for framework indicator files in `directory` in priority order:

| Priority | File checked | Returns |
|---|---|---|
| 1 | `package.json` | `{ framework: 'npm', command: 'npm test' }` |
| 2 | `Cargo.toml` | `{ framework: 'cargo', command: 'cargo test' }` |
| 3 | `pyproject.toml` | `{ framework: 'pytest', command: 'pytest -v' }` |
| 3b | `setup.py` | `{ framework: 'pytest', command: 'pytest -v' }` |
| 4 | `go.mod` | `{ framework: 'go', command: 'go test./...' }` |
| — | None found | `null` |

Uses `existsSync(join(directory, filename))` for each check. Returns on first match.

### `buildResult(passed: boolean | null, framework: string | null, output: string, error?: string): RunTestsResult`

Pure constructor for the result object. Omits the `error` key when not provided.

---

## Implementation Steps

### Step 1: Create `~/.config/opencode/tools/run_tests.ts`

**File structure:**
```
Header docblock → imports → RunTestsResult interface →
detectFramework() → buildResult() → execute() → export default tool({...})
```

**`execute(args, context)` logic:**
1. Resolve `directory`: `args.directory?? context.worktree`
2. Call `detectFramework(directory)` — if `null`, return `JSON.stringify(buildResult(null, null, '', 'No supported test framework detected in ${directory}'))`
3. Run the detected command in `directory` using `Bun.$` with `.cwd(directory)` and `.quiet()`; catch errors; capture `exitCode`, `stdout`, `stderr`
4. Combine stdout and stderr into a single `output` string
5. `exitCode === 0` → `passed = true`; non-zero → `passed = false`
6. Return `JSON.stringify(buildResult(passed, framework, output), null, 2)`

**Error handling:** Wrap `Bun.$` call in try/catch. If subprocess throws (e.g. command not found), return `buildResult(false, framework, errorMessage)` — the tool never throws.

**Tool descriptor:**
```typescript
export default tool({
  description: 'Auto-detect the test framework in a project directory and run its test suite. Returns { passed, framework, output, error? }.',
  args: {
    directory: tool.schema
     .string()
     .optional()
     .describe('Absolute path to project root. Defaults to context.worktree if omitted.'),
  },
  async execute(args, context) {... },
});
```

### Step 2: Create `~/.config/opencode/tools/tests/run_tests.test.ts`

10 test scenarios using `node:test` + `assert/strict`. Imports helpers from `../run_tests.ts`. Uses `mkdtempSync`/`writeFileSync`/`rmSync` for temp dir fixtures.

| # | Function | Scenario | Expected |
|---|---|---|---|
| 1 | `detectFramework` | Dir with `package.json` | `{ framework: 'npm', command: 'npm test' }` |
| 2 | `detectFramework` | Dir with `Cargo.toml` | `{ framework: 'cargo', command: 'cargo test' }` |
| 3 | `detectFramework` | Dir with `pyproject.toml` | `{ framework: 'pytest', command: 'pytest -v' }` |
| 4 | `detectFramework` | Dir with `setup.py` | `{ framework: 'pytest', command: 'pytest -v' }` |
| 5 | `detectFramework` | Dir with `go.mod` | `{ framework: 'go', command: 'go test./...' }` |
| 6 | `detectFramework` | Empty dir | `null` |
| 7 | `detectFramework` | Dir with `package.json` AND `Cargo.toml` | `npm` wins (priority) |
| 8 | `buildResult` | `passed=true`, `framework='npm'`, `output='ok'` | `{ passed: true, framework: 'npm', output: 'ok' }`, no `error` key |
| 9 | `buildResult` | `passed=false`, `framework='cargo'`, `output='FAILED'` | `{ passed: false, framework: 'cargo', output: 'FAILED' }` |
| 10 | `buildResult` | `passed=null`, `framework=null`, `output=''`, `error='No supported...'` | `error` field present |

### Step 3: Verify allowlist
```bash
grep run_tests ~/.config/opencode/opencode.json
```
Expected: matches in `debug`, `build`, `senior-coder` sections. No edit needed.

### Step 4: Commit to `pa.aid.config.md`
```
feat(agent-tools): implement run_tests OpenCode tool (ARC-1354)
```

### Step 5: Write completion summary and check off runbook

---

## Testing & Validation

```bash
node --experimental-strip-types --test ~/.config/opencode/tools/tests/run_tests.test.ts
grep run_tests ~/.config/opencode/opencode.json
```

---

## Acceptance Criteria Mapping

| AC | Covered by |
|---|---|
| AC1: `package.json` → `npm test` | `detectFramework` test 1; `execute()` happy path |
| AC2: `Cargo.toml` → `cargo test` | `detectFramework` test 2 |
| AC3: `pyproject.toml`/`setup.py` → `pytest -v` | `detectFramework` tests 3+4 |
| AC4: `go.mod` → `go test./...` | `detectFramework` test 5 |
| AC5: No framework → `passed: null` + error | `detectFramework` test 6; `execute()` null path |
| AC6: Non-zero exit → `passed: false` | `buildResult` test 9; `execute()` non-zero exit path |

---

## Commit Messages

| Repo | Message |
|---|---|
| `pa.aid.config.md` | `feat(agent-tools): implement run_tests OpenCode tool (ARC-1354)` |
| `pa.aid.runbook-executor` | `docs(agent-tools): add ARC-1354 implementation plan` |
| `pa.aid.runbook-executor` | `chore(agent-tools): check off ARC-1354` |

---

## Risks

| Risk | Severity | Mitigation |
|---|---|---|
| `Bun.$` `.cwd()` not available | Low | Fallback: `Bun.$\`cd ${dir} && command\`` |
| `npm test` exits non-zero when no test script configured | Low | AC6 path handles it |
| `context.worktree` undefined in edge cases | Low | `args.directory` param is explicit override |

---

## After Writing this Plan

🔴 HIGH story — supervisor approval required before any code is written.

---

## After Approval — Steps to Execute

1. Create `run_tests.ts` (Step 1)
2. Create `run_tests.test.ts` (Step 2)
3. Run tests — all 10 must pass
4. Verify allowlist (Step 3)
5. Commit to `pa.aid.config.md` (Step 4)
6. Write completion summary
7. Check off all runbook steps + story checkbox

---

## Steps After Writing the File

1. Commit and push:
```bash
git -C /repos/pa.aid.runbook-executor add implementation_plans/agent-tools/ARC-1354-implementation-plan.md
git -C /repos/pa.aid.runbook-executor commit -m "docs(agent-tools): add ARC-1354 implementation plan"
git -C /repos/pa.aid.runbook-executor push
```

2. Check off step 2 in the runbook:
   - `runbook_check_step`: runbook_path=`/repos/pa.aid.runbook-executor/docs/plans/runbook-agent-tools.md`, story_key=`ARC-1354`, step_number=`2`, lane=`agent-tools`, repo_path=`/repos/pa.aid.runbook-executor`

3. Return confirmation that file was written, committed, pushed. Do NOT write any code.

Return: confirmation that the file was written successfully with its path.