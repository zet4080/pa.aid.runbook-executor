# ARC-1369 Conductor: auto-commit implementation plans and completion summaries to planning repo — Implementation Plan

**Issue:** `issues/parallel-lane-execution/ARC-1369-conductor-auto-commit-planning-artifacts.md`
**Completion Summary:** `task-completions/ARC-1369-COMPLETION-SUMMARY.md` (TBD)
**Approach:** Single-call: add `commitPlanningArtifacts` to `checkpointGit.ts` + post-step hook in `laneRunner.ts`
**Owner:** agent
**Date:** 2026-07-07

## Scope & Alignment

This plan covers the automatic git commit-and-push of implementation plans and completion summaries to planning repo `main` after each agent step completes successfully. It eliminates all manual `planning_commit` ceremony from agent workflows.

AC mapping:

| AC | Step |
|----|------|
| AC1 — `implementation_plans/` new file → committed + pushed with `docs({lane}): add {filename}` | Steps 1 + 2 |
| AC2 — `task-completions/` new file → committed + pushed with appropriate message | Steps 1 + 2 |
| AC3 — no dirty files after step → idempotent no-op | Step 1 (git status guard) |
| AC4 — skills updated to remove manual git ceremony | Step 4 |

## Assumptions & Dependencies

- ARC-1368 is merged to `main` in `pa.aid.conductor.ts`: `laneRunner.ts` already calls `provisionWorktree` and `buildPromptFromStep` with `worktreePath` before `executeStep`.
- ARC-1367 is merged: auto-claim sub-step code exists in `laneRunner.ts`.
- `getState().repoRoot` holds the planning repo absolute path at runtime.
- The conductor runs with git configured so that `git push` without explicit remote/branch targets `origin main`.
- `checkpointGit.ts` uses `execSync` from `child_process` with `{ encoding: 'utf8', stdio: 'pipe', timeout: 10000, cwd: repoRoot }` — new code follows the same shape.
- Push failure must log a warning and not propagate — same contract as existing runbook auto-commit in `checkpointGit.ts`.

## Implementation Steps

---

### Step 1: Add `commitPlanningArtifacts` to `checkpointGit.ts`

**Files:** `packages/server/src/git/checkpointGit.ts`

**Action:**

Export a new function with the signature:

```typescript
export function commitPlanningArtifacts(repoRoot: string, lane: string): void
```

Logic inside the function:

1. Build `execOpts` with `{ encoding: 'utf8' as const, stdio: 'pipe' as const, timeout: 10000, cwd: repoRoot }` — identical shape to the existing `execOpts` in `createCheckpointBranchAndPR`.
2. Call `git status --porcelain -- implementation_plans/ task-completions/` via `execSync`. Capture the result as a string.
3. If the result is empty string (after `.trim()`), return immediately — no work to do (idempotent no-op, satisfies AC3).
4. Parse the `git status --porcelain` output to extract the list of changed filenames. Each non-empty line has format `XY filename` or `XY old -> new`; extract the last whitespace-separated token from each line as the filename.
5. Determine commit message: if exactly one file changed, use `docs(${lane}): add ${path.basename(filename)}`; if multiple files changed, use `docs(${lane}): add planning artifacts`.
6. Stage: `git add -- implementation_plans/ task-completions/`.
7. Commit: `git commit -m "${message}"`.
8. Push — wrap in a separate try/catch: `git push`. On any error, call `console.warn('[ARC-1369] push failed: ' + String(err))` and return without rethrowing.
9. Wrap steps 2–8 in an outer try/catch: on any error except the inner push catch, call `console.warn('[ARC-1369] commitPlanningArtifacts failed: ' + String(err))` and return. This ensures the function never throws.

**Verification:** `checkpointGit.test.ts` tests (Step 3) pass. TypeScript compiles with no errors.

---

### Step 2: Call `commitPlanningArtifacts` in `laneRunner.ts` after successful step completion (no pending gate)

**Files:** `packages/server/src/runner/laneRunner.ts`

**Action:**

Add an import at the top of the file:

```typescript
import { createCheckpointBranchAndPR, commitPlanningArtifacts } from '../git/checkpointGit.js';
```

Within `runLane`, immediately after the existing call `markStepCheckedInMarkdown(runbookPath, step.id)` (the "No pending gate — step fully completed" block), add:

```typescript
// ARC-1369: auto-commit any new planning artifacts produced by this step
const stepState = getState();
commitPlanningArtifacts(stepState.repoRoot, laneFromRunbookPath(runbookPath));
```

This call is synchronous (`commitPlanningArtifacts` uses `execSync`). It follows the same fire-and-survive contract: internal errors are caught and logged; this line never throws.

No change is needed for the "pending gate" branch because that branch does not reach full step completion — `markStepCheckedInMarkdown` is not called there, and planning artifacts are not yet written at that point.

**Verification:** `laneRunner.test.ts` tests (Step 3) confirm `commitPlanningArtifacts` is called after step success and not called on failure or gate-pause.

---

### Step 3: Unit tests

**File:** `packages/server/src/git/checkpointGit.test.ts` (extend existing)

Add a `describe('commitPlanningArtifacts', ...)` block. Mock `execSync` from `child_process` per test using `vi.spyOn` or factory mock. Test scenarios:

1. **No dirty files → no commit, no push:** `git status --porcelain` returns `''` → `execSync` called once (for status), not called for add/commit/push.
2. **One `implementation_plans/` file dirty → commit with single-file message:** status output contains one line for `implementation_plans/parallel-lane-execution/ARC-1369-implementation-plan.md` → commit message is `docs(parallel-lane-execution): add ARC-1369-implementation-plan.md`.
3. **One `task-completions/` file dirty → commit with single-file message:** status output contains one line for a `task-completions/` file → commit message is `docs({lane}): add {filename}`.
4. **Multiple dirty files → commit with generic message:** two lines in status output → commit message is `docs({lane}): add planning artifacts`.
5. **Push failure → warn, no throw:** `git push` throws → function catches, calls `console.warn`, does not rethrow.
6. **Git status throws → warn, no throw:** outer catch fires, function returns without throwing.

**File:** `packages/server/src/__tests__/laneRunner.test.ts` (extend existing)

Add `commitPlanningArtifactsMock` to the `checkpointGit.js` mock factory. Add a new `describe` block: `'runLane — auto-commit planning artifacts (ARC-1369)'` with tests:

1. **Called after successful agent step with no pending gate:** step succeeds, no checkpoint sub-step → `commitPlanningArtifactsMock` called once with `(tmpDir, 'test')`.
2. **NOT called when step fails (after retries exhausted):** all `executeStep` calls return `'failed'` → `commitPlanningArtifactsMock` not called.
3. **NOT called when step succeeds but pending gate blocks completion:** step has unchecked CHECKPOINT sub-step → lane pauses, `commitPlanningArtifactsMock` not called.
4. **Called once per completed step across multi-step lane:** two agent steps both succeed → `commitPlanningArtifactsMock` called twice.

**Verification:** All new tests pass. `npm test` in `packages/server` is green.

---

### Step 4: Update skills to remove manual git ceremony

**Files:**
- `/home/zimmermann/.config/opencode/skills/write-implementation-plan/SKILL.md`
- `/home/zimmermann/.config/opencode/skills/execute-implementation-plan/SKILL.md`
- `/home/zimmermann/.config/opencode/skills/write-completion-summary/SKILL.md`

**Action:**

In `write-implementation-plan/SKILL.md`: Remove the "Commit and Push" section containing the three-line `git -C` block. Replace with a note inline (after the "Write the plan" instruction):

> **Note:** The conductor auto-commits implementation plans to planning repo `main` after each step completes. No manual git commit is needed after writing the plan file.

Keep the "commit and push for approach decision documents" block (for `done/decisions/`) unchanged — that is a different artifact not covered by ARC-1369.

In `execute-implementation-plan/SKILL.md`: Locate any instruction to run `git add`/`git commit`/`git push` or call `planning_commit` for the implementation plan file. Remove or replace with the same conductor-note.

In `write-completion-summary/SKILL.md`: Locate any instruction to commit/push the completion summary file. Remove or replace with: "The conductor auto-commits completion summaries to planning repo `main` after each step completes."

**Verification:** All three skill files no longer contain instructions to manually commit implementation plan or completion summary files. Spot-check by grepping for `git -C` in the edited sections.

---

## Testing & Validation

- Run `npm test` in `packages/server/` — all existing tests plus the new `commitPlanningArtifacts` tests must pass.
- Run `npm run build` in `packages/server/` — TypeScript must compile with no errors.
- Manual smoke test: start the conductor against a test runbook, have an agent step write a file under `implementation_plans/`; after the step completes, verify the file appears in planning repo git log with the correct commit message.

## Risks & Open Questions

- **Commit timing:** `commitPlanningArtifacts` is called after `markStepCheckedInMarkdown` but before the next step starts. Because `execSync` is synchronous and Node.js is single-threaded, no interleaving with other lanes can occur at this point.
- **Empty `implementation_plans/` or `task-completions/` dirs:** `git status --porcelain` against non-existent directories returns empty output — idempotent no-op as expected.
- **Concurrent lanes writing different files:** each lane calls `commitPlanningArtifacts` independently after its own step completes. Because `markStepCheckedInMarkdown` → `commitPlanningArtifacts` is a synchronous sequence and only fires when a step fully completes, concurrent commits are not possible within a single conductor instance.
