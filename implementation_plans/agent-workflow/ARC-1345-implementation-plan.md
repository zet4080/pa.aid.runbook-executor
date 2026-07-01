# ARC-1345 Fix checkpoint workflow — runbook file changes must not appear in checkpoint PR - Implementation Plan

**Issue:** `issues/agent-workflow/ARC-1345-fix-runbook-commit-workflow.md`
**Completion Summary:** `task-completions/ARC-1345-COMPLETION-SUMMARY.md` (TBD)
**Approach:** Pre-branch runbook flush — detect and commit runbook-matching files to base branch before `git checkout -b`
**Owner:** agent
**Date:** 2026-07-01

---

## Scope & Alignment

This plan covers a single surgical change to `createCheckpointBranchAndPR` in
`packages/server/src/git/checkpointGit.ts` (`pa.aid.conductor.ts`).

Before the function creates the checkpoint branch it must:
1. Detect any uncommitted working-tree changes to files matching `docs/plans/runbook-*.md`
2. Stage and commit those files directly to the current base branch
3. Then proceed with branch creation as today

If no runbook files are modified, the new step is a no-op and existing behaviour is unchanged.

AC mapping:

| AC | Covered by |
|----|-----------|
| Runbook changes go to base branch, not PR | Step 1 (flush helper) + Step 2 (call site) |
| Planning artifacts still reach checkpoint PR | Step 2 (no change to post-flush branch logic) |
| PR diff contains only planning artifacts | Step 1 + Step 2 — runbook files committed before branch creation |
| No-op when no runbook files changed | Conditional in `flushRunbookChangesToBaseBranch` |

---

## Assumptions & Dependencies

- `execSync` is already imported and used with a consistent `execOpts` object — the new helper uses the same pattern.
- The planning repo path (`repoRoot`) is the git working tree root — `docs/plans/runbook-*.md` paths are relative to it.
- Git's short-status format (`git status --porcelain`) is available in all environments where the server runs.
- The current branch at call time is always the base branch (e.g. `main`) — no prior checkout has occurred.
- Runbook commit message format: `chore(runbook): flush runbook checkoffs before checkpoint branch` — plain, no Jira key, consistent with planning-repo commit conventions.
- Test framework: Vitest; `execSync` is already fully mocked in `checkpointGit.test.ts`.

---

## Implementation Steps

### Step 1 — Extract `flushRunbookChangesToBaseBranch` helper

**File:** `packages/server/src/git/checkpointGit.ts`

**Action:**
Add a new private function `flushRunbookChangesToBaseBranch(repoRoot: string, execOpts: object): void` immediately before `createCheckpointBranchAndPR`.

Logic:
1. Run `git status --porcelain` in `repoRoot` with the shared `execOpts`.
2. Parse each output line. Extract the file path (column 4 onward after stripping the two-character status prefix and optional rename arrow).
3. Filter paths matching the glob `docs/plans/runbook-*.md` (use a simple `String.includes('docs/plans/runbook-')` prefix + `.md` suffix check — no external glob dependency).
4. If the filtered list is empty, return immediately (no-op).
5. Stage each matched file: `git add -- <path>` for each path in the list.
6. Commit: `git commit -m "chore(runbook): flush runbook checkoffs before checkpoint branch"`.

The function throws synchronously on any `execSync` error — do not swallow git failures.

**Verification:** Unit tests for the helper (added in Step 3) pass.

---

### Step 2 — Call the helper in `createCheckpointBranchAndPR`

**File:** `packages/server/src/git/checkpointGit.ts`

**Action:**
In `createCheckpointBranchAndPR`, move `execOpts` to be declared before the `branchExistsLocally` block (it already is — confirm it is in scope). Insert a single call:

```
flushRunbookChangesToBaseBranch(repoRoot, execOpts);
```

immediately before the `try { execSync('git rev-parse ...' }` block (i.e., before any branch-existence check or checkout). No other changes to the existing function body.

**Verification:** Existing tests that count `execSync` call counts will need to be updated (Step 4) to account for the two new calls (`git status --porcelain` + `git add` + `git commit` when runbook files are present, or just `git status --porcelain` when they are absent).

---

### Step 3 — Add unit tests for `flushRunbookChangesToBaseBranch`

**File:** `packages/server/src/git/checkpointGit.test.ts`

**Action:**
Add a new `describe` block: `'flushRunbookChangesToBaseBranch'`. Because the helper is private, test it indirectly through `createCheckpointBranchAndPR` by controlling what `execSyncMock` returns for the `git status --porcelain` call.

Test scenarios:

1. **No runbook changes** — `git status --porcelain` returns empty string. Assert `git add` and `git commit` are NOT called. Assert subsequent checkout/push calls proceed as normal.

2. **One runbook file changed** — `git status --porcelain` returns ` M docs/plans/runbook-session-setup.md`. Assert `git add -- docs/plans/runbook-session-setup.md` is called, then `git commit -m "chore(runbook): flush runbook checkoffs before checkpoint branch"` is called, before the `git checkout -b` call.

3. **Multiple runbook files changed** — `git status --porcelain` returns two matching lines. Assert both `git add` calls are issued (one per file) before the commit.

4. **Non-runbook files only** — `git status --porcelain` returns `M  src/something.ts`. Assert no `git add` or `git commit` flush call is made.

5. **Mixed: runbook + non-runbook files** — `git status --porcelain` returns one runbook line and one non-runbook line. Assert only the runbook file is staged; the non-runbook file is not touched.

**Verification:** `pnpm test` in `packages/server` — all five new tests pass alongside all existing tests.

---

### Step 4 — Update existing execSync call-count assertions

**File:** `packages/server/src/git/checkpointGit.test.ts`

**Action:**
The existing `'calls execSync for git checkout -b, git push, and git checkout -'` test asserts `execSyncMock` was called exactly 3 times. After the change, there is a minimum of 1 additional call (`git status --porcelain`). Update affected assertions:

- For tests where `git status --porcelain` returns `''` (empty — no runbook changes): add 1 to the expected call count (the status check call).
- Update `toHaveBeenCalledTimes(3)` → `toHaveBeenCalledTimes(4)` and update the `toHaveBeenNthCalledWith` positions accordingly (status check is now call 1; existing calls shift by 1).
- The `branchExistsLocally` check (`git rev-parse --verify`) also exists in the flow — trace the exact call sequence and set counts accordingly.

Verify the existing `'throws synchronously when execSync throws (git failure is not swallowed)'` test still passes — the first execSync call is now `git status --porcelain`, so the mock should throw on that call and the function should still reject.

**Verification:** `pnpm test` — all tests pass, no regressions in the existing PR-creation and failure-path suites.

---

## Testing & Validation

Run from `packages/server`:

```
pnpm test
```

Expected: all tests in `checkpointGit.test.ts` pass (new + updated). No failures in `checkpoints.test.ts` or other suites.

Manual integration check (optional, no CI required for this story):
- In a local planning repo with a dirty `docs/plans/runbook-*.md` file and an unstaged implementation plan, invoke `createCheckpointBranchAndPR` and verify:
  - The runbook file is committed to `main` before the branch is created.
  - The checkpoint branch contains only the implementation plan diff.

---

## Risks & Open Questions

- **`execSync` call-count drift in existing tests** — the existing tests count exact call numbers. The step-by-step ordering in Step 4 mitigates this; the plan explicitly requires updating those counts.
- **Renamed files in `git status --porcelain`** — rename lines have format `R  old -> new`. The helper must correctly extract the destination path. Scenario: `R  docs/plans/runbook-old.md -> docs/plans/runbook-new.md`. The parsing logic should split on ` -> ` and take the second segment when present.
- **No new runtime dependencies required** — the implementation uses only `execSync` (already present) and standard string operations.
