```markdown
# ARC-1355 Completion Summary — `run_lint` OpenCode Tool

| Field | Value |
|-------|-------|
| Story | ARC-1355 |
| Title | Implement `run_lint` OpenCode tool |
| Date | 2026-07-05 |
| Lane | agent-tools |
| Status | ✅ Complete |

## Files Created

| File | Repo | Commit |
|------|------|--------|
| `tools/run_lint.ts` | `pa.aid.config.md` | `06c5910` |
| `tools/tests/run_lint.test.ts` | `pa.aid.config.md` | `06c5910` |

Branch: `ARC-1355` in `pa.aid.config.md`

## Test Results

```
node --experimental-strip-types --test ~/.config/opencode/tools/tests/run_lint.test.ts

▶ detectLinter
  ✔ 1: package.json with scripts.lint → eslint / npm run lint (1.78ms)
  ✔ 2: package.json with scripts but no lint key → error shape (0.21ms)
  ✔ 3: package.json with no scripts key → error shape (0.13ms)
  ✔ 4: pyproject.toml only → ruff check. (0.17ms)
  ✔ 5: ruff.toml only → ruff check. (0.14ms)
  ✔ 6: Cargo.toml → cargo clippy -- -D warnings (0.16ms)
  ✔ 7: go.mod → golangci-lint run (0.13ms)
  ✔ 8: empty directory → null (0.17ms)
  ✔ 9: package.json (no lint) + pyproject.toml → package.json early return wins (0.21ms)
  ✔ 10: malformed package.json → error shape (0.30ms)
✔ detectLinter (4.23ms)
▶ buildLintResult
  ✔ 11: passed=true, linter='eslint', output='ok' — no error key (0.14ms)
  ✔ 12: passed=null, linter=null, output='', error present (0.06ms)
✔ buildLintResult (0.28ms)

tests 12 | pass 12 | fail 0
```

## Allowlist Verification

```
grep run_lint ~/.config/opencode/opencode.json
→ 3 hits (debug, build, senior-coder agents)
```

## Acceptance Criteria

| AC | Description | Evidence |
|----|-------------|----------|
| AC1 | `package.json` with lint → `npm run lint`, `linter: "eslint"` | Test 1; `detectLinter` returns `{ linter: 'eslint', command: 'npm run lint' }` |
| AC2 | `pyproject.toml`/`ruff.toml` → `ruff check.`, `linter: "ruff"` | Tests 4+5 |
| AC3 | `Cargo.toml` → `cargo clippy -- -D warnings`, `linter: "clippy"` | Test 6 |
| AC4 | `go.mod` → `golangci-lint run`, `linter: "golangci-lint"` | Test 7 |
| AC5 | `package.json` no lint script → `passed: null` + error | Tests 2+3+9+10; `execute()` linter-null branch |
| AC6 | No linter → `passed: null`, `output: """ + error | Test 8; `execute()` null branch |

## Implementation Notes

- `detectLinter()` has three return shapes (not two): happy path / package.json-no-lint error / null
- `package.json` with no `scripts.lint` blocks Python/Rust/Go detection (early return)
- Malformed `package.json` treated same as missing `scripts.lint` (try/catch in JSON.parse)
- `linter` field hardcoded to `'eslint'` for npm path per AC spec
- Tool never throws — all subprocess errors caught and returned as `passed: false`
```