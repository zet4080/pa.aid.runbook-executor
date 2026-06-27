---
name: local-code-review
description: Use when implementation is complete and code needs local review before PR creation — runs tests, lint, type-check, reviews diff against acceptance criteria, classifies findings, and loops fix→retest→re-review until clean or escalates to human after 3 iterations
---

# Local Code Review

## Purpose

Pre-PR automated review loop: runs tests, lint, type checks, and AC coverage validation, then loops fix→retest→re-review until clean or escalates after 3 iterations. Catches blockers before PR creation.

## When to Use

- After `execute-implementation-plan` completes all steps
- Before `finishing-a-development-branch` (called automatically by that skill)
- Manually when resuming a branch mid-way and unsure of quality state

---

## The Loop

```
Run quality checks
       ↓
Classify findings
       ↓
BLOCKER or ISSUE found?
  ├─ yes → Fix → increment iteration → re-run from top
  └─ no  → Exit clean → proceed to finishing-a-development-branch
```

**Max iterations: 3.** If iteration 3 still has BLOCKER or ISSUE findings → STOP, escalate to human.

---

## Step 1: Load Issue File

Find the issue file for the current branch:

```bash
BRANCH=$(git branch --show-current)
ISSUE_ID=$(echo "$BRANCH" | grep -oE '^[A-Z]+-[0-9]+')
```

Search `issues/` recursively for `${ISSUE_ID}-*.md`. Read it. Extract acceptance criteria.

If no issue file found: note it, continue without AC validation.

---

## Step 2: Run Quality Checks

Run all applicable checks. Collect all output before classifying.

### 2a. Tests
```bash
npm test 2>&1         # Node/TS
cargo test 2>&1       # Rust
pytest -v 2>&1        # Python
go test ./... 2>&1    # Go
```

### 2b. Lint
```bash
npm run lint 2>&1
ruff check . 2>&1
cargo clippy 2>&1
golangci-lint run 2>&1
```

### 2c. Type check
```bash
npx tsc --noEmit 2>&1
mypy . 2>&1
```

### 2d. Diff review
```bash
git diff $(git merge-base HEAD main 2>/dev/null || git merge-base HEAD master)...HEAD
```

Review the diff against the acceptance criteria extracted in Step 1.

**Skip checks that have no corresponding tool in the project.** Do not fail for missing tools.

---

## Step 3: Classify Findings

Classify every finding using this rubric:

| Severity | Definition | Examples |
|----------|-----------|---------|
| **BLOCKER** | Must fix before any PR | Test failure, type error, broken functionality, security vulnerability, data corruption risk |
| **ISSUE** | Should fix before PR | Missing AC coverage, incorrect assumption, unhandled edge case, maintainability problem |
| **SUGGESTION** | Optional improvement | Cleaner implementation, alternative design |
| **NIT** | Minor style | Naming preference, formatting |

**Stop condition:** Loop continues only while BLOCKER or ISSUE findings remain.
SUGGESTION and NIT may remain open — they do not block progression.

---

## Step 4: Report Findings

```
=== Local Code Review — Iteration {N}/3 ===

BLOCKERS ({count}):
  [B1] {file}:{line} — {description}

ISSUES ({count}):
  [I1] {file}:{line} — {description}

SUGGESTIONS ({count}):
  [S1] {description}

NITS ({count}):
  [N1] {description}

AC Coverage:
  ✅ AC1: {criterion} — covered by {test/code}
  ❌ AC2: {criterion} — NOT covered → ISSUE

Status: {CLEAN | NEEDS_FIXES}
```

---

## Step 5: Fix or Exit

### If CLEAN (no BLOCKER, no ISSUE):

```
✅ Local review clean (iteration {N}/3).
No blockers. No issues. {K} suggestions / {M} nits remain (non-blocking).
Proceeding to finishing-a-development-branch.
```

Call `finishing-a-development-branch`.

### If NEEDS_FIXES and iteration < 3:

Fix all BLOCKER and ISSUE findings. Then re-run from Step 2.

Commit fixes:
```bash
git add -A
git commit -m "{TRACKER-ID}: fix review findings (iteration {N})"
```

### If NEEDS_FIXES and iteration = 3:

**STOP. Escalate to human.**

```
🔴 ESCALATE — local review iteration 3/3 still has unresolved findings.

Remaining BLOCKERs:
  [B1] ...

Remaining ISSUEs:
  [I1] ...

Cannot auto-resolve. Human review required before proceeding.
Options:
  A) Accept findings as-is and proceed to PR (human takes responsibility)
  B) Provide guidance and continue fixing
  C) Discard branch
```

Wait for human decision. Do not proceed without explicit instruction.

---

## Quick Reference

| Situation | Action |
|-----------|--------|
| All checks pass, no findings | Exit clean immediately (iteration 1) |
| BLOCKER/ISSUE found, iter < 3 | Fix → re-run |
| BLOCKER/ISSUE found, iter = 3 | STOP, escalate |
| Only SUGGESTION/NIT remain | Exit clean, note non-blocking findings |
| No issue file found | Skip AC validation, run quality checks only |
| Tool not present in project | Skip that check, do not fail |

---

## Finding Severity Quick Reference

| Always BLOCKER | Always ISSUE | May be SUGGESTION |
|----------------|--------------|-------------------|
| Test failure | Uncovered AC | Performance improvement |
| Type error | Missing error handling | Alternative design |
| Security vulnerability | Incorrect assumption | Code simplification |
| Build failure | Unhandled edge case | Extract helper |
| Data corruption risk | Maintainability problem | |

---

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Skipping Step 2 checks because "tests already pass" | Always re-run fresh — state may have changed |
| Treating SUGGESTION as ISSUE | SUGGESTION does not block progression |
| Fixing and re-running without committing | Commit each fix iteration so history is clean |
| Proceeding after 3 failed iterations | STOP — escalate. Never bypass the limit. |
| Skipping AC validation because no issue file | Note the absence; run quality checks regardless |

---

## Red Flags

**Never:**
- Proceed to PR with unresolved BLOCKER or ISSUE findings
- Skip iteration limit and keep auto-fixing beyond 3
- Mark findings as SUGGESTION to avoid fixing them (severity manipulation)
- Claim "clean" without running fresh checks

**Always:**
- Run all applicable checks fresh each iteration
- Commit fixes between iterations
- Escalate on iteration 3 with full finding list
- Report AC coverage explicitly