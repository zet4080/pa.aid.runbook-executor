# ARC-1345 Fix runbook-commit workflow: no PR for runbook artifacts — Implementation Plan
**Issue:** `issues/agent-workflow/ARC-1345-fix-runbook-commit-workflow.md`
**Completion Summary:** `task-completions/ARC-1345-COMPLETION-SUMMARY.md` (TBD)
**Approach:** Pre-branch commit of runbook files — single approach, pattern established
**Owner:** build agent
**Date:** 2026-07-01

---

## Scope & Alignment

This plan covers a single surgical change to `createCheckpointBranchAndPR` in
`packages/server/src/git/checkpointGit.ts` (`pa.aid.conductor.ts`).

| AC | Covered by step |
|----|----------------|
| Runbook files (`docs/plans/runbook-*.md`) are committed to base branch before `git checkout -b` | Step 1 |
| If no runbook changes are present, the pre-branch commit step is a no-op | Step 1 |
| Planning artifacts still appear in the checkpoint branch and PR | Step 1 (unchanged branching path) |
| Checkpoint PR diff contains only planning artifacts — no runbook files | Steps 1 + 2 |

Out of scope: PR creation logic, Bitbucket integration, agent skill instructions, planning artifact routing.

---

## Assumptions & Dependencies

- `REPO_ROOT` (passed as `opts.repoRoot`) is the absolute path to the planning repo on disk.
- The function is called while the planning repo is checked out on its base branch (e.g. `main`). Runbook files are present in the working tree with uncommitted changes.
- `execSync` (already imported from `child_process`) is the correct primitive to use — no new dependencies.
- `git add -- <pathspec>` with a glob pattern is safe to use inside `execSync`; if nothing matches the pattern, the command exits 0 with no changes staged.
- A `git commit` with nothing staged (when there are no runbook changes) exits with code 1. The pre-branch commit must therefore check for staged runbook changes before invoking `git commit`, to keep the no-op path clean.
- The existing tests count the exact number of `execSync` calls. Adding the pre-branch commit steps will change the call count; the test suite must be updated accordingly.

---

## Implementation Steps

### Step 1 — Insert pre-branch runbook commit in `createCheckpointBranchAndPR`

**File:** `packages/server/src/git/checkpointGit.ts`

**Action:**

Immediately before the `branchExistsLocally` check (the block that calls `git rev-parse --verify` and then `git checkout -b`), insert a pre-branch runbook commit block:

1. Run `git status --porcelain -- docs/plans/runbook-*.md` to detect whether any runbook files are modified (staged or unstaged).
2. If the output is non-empty (i.e. at least one matching file has changes):
   a. Run `git add -- docs/plans/runbook-*.md` to stage all modified runbook files.
   b. Run `git commit -m "docs(runbook): auto-commit runbook checkoffs before checkpoint branch"` to commit them directly to the current base branch.
3. If the output is empty, skip the add+commit — no-op.

The surrounding branch-creation and push logic is unchanged.

**Verification:** Read the modified function and confirm:
- The `git status --porcelain` call precedes `git rev-parse --verify`.
- The `git add` and `git commit` calls are inside an `if` guard.
- The `git checkout -b` / `git checkout` block is unchanged.

---

### Step 2 — Update tests in `checkpointGit.test.ts`

**File:** `packages/server/src/git/checkpointGit.test.ts`

**Action:**

The existing success-path test `'calls execSync for git checkout -b, git push, and git checkout -'` asserts `execSyncMock` is called exactly 3 times. After Step 1 the pre-branch path adds one `execSync` call (`git status --porcelain`), changing the minimum call count.

Update the test suite:

1. **Existing `'calls execSync for git checkout -b, git push, and git checkout -'` test:**
   - Change the `toHaveBeenCalledTimes(3)` assertion to `toHaveBeenCalledTimes(4)` (adding the `git status --porcelain` call).
   - Retain the existing `toHaveBeenNthCalledWith` assertions but shift their indices: `git status --porcelain` is now call 1; `git checkout -b` is call 2; `git push` is call 3; `git checkout -` is call 4.

2. **Add a new test: `'commits runbook files to base branch before creating checkpoint branch'`:**
   - Scenario: `execSyncMock` returns non-empty output (e.g. `'M docs/plans/runbook-core.md'`) for the `git status --porcelain` call and `''` for all others.
   - Verify that `execSyncMock` was called with `git add -- docs/plans/runbook-*.md` and then with the auto-commit message, both before the `git checkout -b` call.

3. **Add a new test: `'skips runbook commit when no runbook files are modified'`:**
   - Scenario: `execSyncMock` returns `''` for the `git status --porcelain` call (no runbook changes).
   - Verify that `git add` and `git commit` are NOT called for runbook files.
   - Verify that `git checkout -b` is still called (normal path proceeds).

**Verification:** Run `pnpm --filter server test` (or the equivalent test command for the project) and confirm all tests pass with no failures.

---

## Testing & Validation

### Unit tests (Step 2)

Run the server package tests:
```
pnpm --filter server test
```
Expected: all existing tests plus two new tests pass.

### Manual smoke scenario (conceptual)

Scenario: planning repo has `docs/plans/runbook-core-infrastructure.md` with an uncommitted change AND an uncommitted `implementation_plans/agent-workflow/ARC-1345-implementation-plan.md`.

On `createCheckpointBranchAndPR` call:
- `git log --oneline -1` on the base branch should show the auto-commit of the runbook file.
- `git diff main...checkpoint/{lane}/{stepId} --name-only` should list only `implementation_plans/agent-workflow/ARC-1345-implementation-plan.md` — not any `docs/plans/runbook-*.md` file.

### Edge case: no runbook changes

Scenario: only a planning artifact is uncommitted, no runbook files modified.
- The `git status --porcelain` returns empty output.
- No `git add` or `git commit` for runbook files occurs.
- Checkpoint branch and PR are created normally.

---

## Risks & Open Questions

- **`execSync` call-count coupling in tests:** Existing tests assert exact call counts. The test updates in Step 2 are load-bearing — if Step 2 is incomplete the test suite will fail on call-count assertions. Steps must be executed together.
- **Shell glob expansion in `execSync`:** The glob `docs/plans/runbook-*.md` is passed as a shell argument string inside `execSync`. Verify the shell expands the glob correctly in the test environment. If `execSync` is invoked with `shell: false`, the glob will not be expanded by the OS shell — the existing `execSync` calls in the file use the two-argument form (command string, options), which invokes the shell by default on POSIX, so glob expansion should work.
- **Commit identity:** `git commit` requires `user.name` and `user.email` to be configured in the repo or globally. In CI/agent environments this is typically configured. No change needed here; if it fails, the error will surface through the existing un-caught `execSync` path (git failures are not swallowed — consistent with existing behavior).
- **Concurrent checkpoint calls:** Not addressed here. The issue is scoped to the single-call sequential flow.
