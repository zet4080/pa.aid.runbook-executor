# ARC-1355 Implementation Plan — `run_lint` OpenCode Tool

**Issue:** `issues/agent-tools/ARC-1355-tool-run-lint.md`
**Completion Summary:** `task-completions/ARC-1355-COMPLETION-SUMMARY.md` (TBD)
**Approach:** Single approach — mirrors `run_tests.ts` pattern with lint-specific detection logic
**Owner:** agent-tools
**Date:** 2026-07-05

---

## Scope & Alignment

Create `~/.config/opencode/tools/run_lint.ts` — a self-contained OpenCode tool that auto-detects the linter in a project directory and runs it. Returns `{ passed, linter, output, error? }`.

Covers all 6 acceptance criteria: four linter-specific happy paths (eslint/npm, ruff, clippy, golangci-lint), the package.json-without-lint-script error path, and the no-linter-detected error path.

---

## Phase 0 — Exploration Findings

| Question | Answer |
|---|---|
| `run_lint.ts` already exists? | ❌ No |
| Test file already exists? | ❌ No |
| Allowlist already configured? | ✅ Yes — `run_lint` present in `opencode-sync-config.sh` for `debug`, `build`, `senior-coder` |
| `opencode.json` update needed? | No |
| `lib/` imports needed? | ❌ No — self-contained |
| Test runner | `node --experimental-strip-types --test` |
| Subprocess pattern | `Bun.$` with `.cwd(dir).quiet().nothrow()` — confirmed from `run_tests.ts` |
| `context.worktree` usage | Default directory when `args.directory` omitted |
| `run_tests.ts` `detectFramework` reusable? | ❌ No — see ARC-1354 shared-detection decision below |

---

## ARC-1354 Shared-Detection Decision

`run_tests.ts` exports `detectFramework()` which uses file-presence only. `run_lint.ts` requires different logic: parsing `package.json` contents for `scripts.lint`, and detecting Python via `pyproject.toml` OR `ruff.toml`. Importing `detectFramework` from `run_tests.ts` would create a cross-tool dependency and would not satisfy the lint-specific detection requirements. Decision: keep `run_lint.ts` fully self-contained with its own `detectLinter()` helper.

---

## Files

| Action | Path |
|---|---|
| CREATE | `~/.config/opencode/tools/run_lint.ts` |
| CREATE | `~/.config/opencode/tools/tests/run_lint.test.ts` |
| VERIFY (no edit) | `opencode-sync-config.sh` — `run_lint` already allowlisted |

---

## Return Type

```typescript
export interface RunLintResult {
  passed: boolean | null;
  linter: string | null;
  output: string;
  error?: string;
}
```

---

## Exported Helpers (for testability)

### `detectLinter(directory: string): { linter: string; command: string } | { linter: null; error: string } | null`

Three possible return shapes:

| Shape | When |
|---|---|
| `{ linter: string; command: string }` | Linter detected — happy path |
| `{ linter: null; error: string }` | `package.json` found but no `scripts.lint` key |
| `null` | No recognised linter detected at all |

**Detection logic — evaluated in priority order:**

| Priority | Condition | Returns |
|---|---|---|
| 1 | `package.json` exists AND `parsed.scripts.lint` defined | `{ linter: 'eslint', command: 'npm run lint' }` |
| 2 | `package.json` exists but no `scripts.lint` (or malformed JSON) | `{ linter: null, error: 'No lint script found in package.json' }` ← early return |
| 3 | `pyproject.toml` OR `ruff.toml` exists | `{ linter: 'ruff', command: 'ruff check.' }` |
| 4 | `Cargo.toml` exists | `{ linter: 'clippy', command: 'cargo clippy -- -D warnings' }` |
| 5 | `go.mod` exists | `{ linter: 'golangci-lint', command: 'golangci-lint run' }` |
| — | None | `null` |

**`package.json` parsing detail:**
- `readFileSync(join(directory, 'package.json'), 'utf8')` wrapped in `try/catch`
- `JSON.parse()` in try/catch — malformed JSON → treat as no lint script
- Guard: `parsed?.scripts?.lint!== undefined`

**Python detection:** `existsSync(join(directory, 'pyproject.toml')) || existsSync(join(directory, 'ruff.toml'))`

**`linter` name for npm path:** Always `'eslint'` per AC spec regardless of what the script actually invokes.

### `buildLintResult(passed: boolean | null, linter: string | null, output: string, error?: string): RunLintResult`

Pure result constructor — omits `error` key when not provided:

```typescript
export function buildLintResult(
  passed: boolean | null,
  linter: string | null,
  output: string,
  error?: string,
): RunLintResult {
  const result: RunLintResult = { passed, linter, output };
  if (error!== undefined) result.error = error;
  return result;
}
```

---

## Implementation Steps

### Step 1: Create `~/.config/opencode/tools/run_lint.ts`

**File structure:**
```
Header docblock → imports → RunLintResult interface →
detectLinter() → buildLintResult() → execute() → export default tool({...})
```

**`execute(args, context)` logic:**
1. Resolve `directory`: `args.directory?? context.worktree`
2. Call `detectLinter(directory)`
3. If result is `null` (no linter detected): return `JSON.stringify(buildLintResult(null, null, '', 'No linter detected in ${directory}'), null, 2)`
4. If result has `linter: null` (package.json without lint script): return `JSON.stringify(buildLintResult(null, null, '', result.error), null, 2)`
5. Happy path — run `Bun.$\`sh -c ${command}\`.cwd(directory).quiet().nothrow()\