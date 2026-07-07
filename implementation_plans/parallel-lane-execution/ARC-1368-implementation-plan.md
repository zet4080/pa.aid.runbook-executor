# ARC-1368 Conductor: auto-provision feature git worktree for HIGH stories — Implementation Plan

**Issue:** `issues/parallel-lane-execution/ARC-1368-conductor-auto-provision-worktree.md`
**Completion Summary:** `task-completions/ARC-1368-COMPLETION-SUMMARY.md` (TBD)
**Approach:** New `worktreeGit.ts` module + integration into `laneRunner.ts` before agent dispatch
**Owner:** agent
**Date:** 2026-07-07

## Overview

When the conductor identifies an agent step that contains a HIGH-priority checkpoint sub-step (i.e. `INDIVIDUAL PLAN CHECKPOINT`), it must auto-provision a git worktree at `/repos/<KEY>` on branch `<KEY>` (branched from `main` in `pa.aid.conductor.ts`) before dispatching the agent prompt. This eliminates all manual worktree setup ceremony from agent workflows.

**Why now:** ARC-1367 introduced auto-claim (fires before agent dispatch). ARC-1368 follows the same pattern: a side-effect that runs before `executeStep`, implemented in its own git module and called from `runLane`.

**Application code target:** All changes go in `/repos/pa.aid.conductor.ts`. This is a planning-repo file only.

## Assumptions & Dependencies

- ARC-1367 is merged to main: `laneRunner.ts` already calls `markClaimedSubStepInMarkdown` + `commitAndPushClaim` before `executeStep`.
- The `pa.aid.conductor.ts` repo root is known at runtime as `getState().repoRoot`. However, the worktree target is NOT the planning repo — it is the **application repo** (`/repos/pa.aid.conductor.ts`). This path is read from `process.env.APP_REPO_ROOT` (with fallback to `/repos/pa.aid.conductor.ts` for local dev). This env var must be added to `.env.example`.
- A HIGH story is detected by inspecting the step's sub-steps: if any sub-step label contains `CHECKPOINT` and does **not** contain `WAVE` or `GATE` without `HIGH` (i.e. matches `checkpointLevelFromSubStepLabel(label) === 'high'`), the step is a HIGH story.
- The ARC key is extracted from `step.id` via `stripNamespace(step.id)` — the same utility used by the auto-claim code (exported from `markdownWriter.ts`).
- Branch name = story key exactly (no prefix, no suffix). e.g. `ARC-1368`.
- Worktree base: always `main` in the app repo.
- Already-exists is not an error: if worktree path `/repos/<KEY>` already exists or the branch already exists on the local repo, reuse it.
- Failure to create the worktree is a **hard error** — throw, let `runLane` catch it as a step failure, and pause the lane.

## Acceptance Criteria Mapping

- **AC 1** (worktree created before agent prompt): covered by Step 3 — `provisionWorktree` is called and awaited before `executeStep` in `runLane`.
- **AC 2** (resume: reuse existing worktree): covered by Step 2 — `provisionWorktree` checks `git worktree list` and branch existence before calling `git worktree add`.
- **AC 3** (prompt context includes worktree path): covered by Step 3b — `buildPromptFromStep` accepts an optional `worktreePath` parameter and injects it as `WORKTREE_PATH=/repos/<KEY>` into the prompt header.
- **AC 4** (start-execution-session skill updated): covered by Step 5 — remove manual `git worktree list` / `git worktree add` instructions.

## Implementation Steps

---

### Step 1: Create `worktreeGit.ts` module

**File:** `packages/server/src/git/worktreeGit.ts` (new)

**Action:**

Create a module that exports one function:

```typescript
/**
 * Provisions a git worktree at `/repos/<storyKey>` on branch `<storyKey>`
 * in the given app repo (defaults to APP_REPO_ROOT env or /repos/pa.aid.conductor.ts).
 *
 * Idempotent:
 *  - If worktree path already exists in `git worktree list`, returns early (reuse).
 *  - If branch already exists locally, uses `git worktree add <path> <branch>` (no -b flag).
 *  - If neither exists, creates with `git worktree add <path> -b <branch> main`.
 *
 * Throws a descriptive Error on any git failure other than "already exists".
 *
 * @param storyKey  The story key (e.g. "ARC-1368") — used as branch name and worktree path suffix.
 * @param appRepoRoot  Absolute path to the pa.aid.conductor.ts repo. Defaults to APP_REPO_ROOT env || '/repos/pa.aid.conductor.ts'.
 */
export function provisionWorktree(storyKey: string, appRepoRoot?: string): void
```

Implementation logic:
1. Resolve `appRepoRoot` from parameter → `process.env.APP_REPO_ROOT` → `'/repos/pa.aid.conductor.ts'`.
2. Set `worktreePath = '/repos/' + storyKey`.
3. Call `git -C <appRepoRoot> worktree list --porcelain`. Parse output lines starting with `worktree`. If any line equals `worktree <worktreePath>` exactly, return (already provisioned).
4. Check if branch exists: `git -C <appRepoRoot> rev-parse --verify <storyKey>`. If exit code 0 → branch exists locally: run `git -C <appRepoRoot> worktree add <worktreePath> <storyKey>`. If non-zero → branch does not exist: run `git -C <appRepoRoot> worktree add <worktreePath> -b <storyKey> main`.
5. Use `execSync` with `{ encoding: 'utf8', stdio: 'pipe', timeout: 15000 }` for all git calls.
6. Do NOT catch the final worktree-add error — let it propagate to the caller.

Import: `import { execSync } from 'child_process';`

**Verification:** TypeScript compiles. Unit tests (Step 4) pass.

---

### Step 2: Add `APP_REPO_ROOT` to `.env.example`

**File:** `.env.example`

**Action:**

Add the following line (with a comment) to `.env.example`:

```
# Absolute path to the pa.aid.conductor.ts application repo (used for git worktree provisioning)
APP_REPO_ROOT=/repos/pa.aid.conductor.ts
```

---

### Step 3: Detect HIGH story and provision worktree in `laneRunner.ts`

**File:** `packages/server/src/runner/laneRunner.ts`

**Action:**

3a. Add import at top of file:

```typescript
import { provisionWorktree } from '../git/worktreeGit.js';
```

3b. Add helper function `isHighStory(step: RunbookStep): boolean`:

```typescript
/**
 * Returns true if the agent step contains a HIGH-level checkpoint sub-step.
 * Used to decide whether to auto-provision a git worktree before dispatch.
 */
function isHighStory(step: RunbookStep): boolean {
  return (step.subSteps?? []).some(
    (ss: RunbookSubStep) =>
     !ss.checked &&
      ss.label.toUpperCase().includes('CHECKPOINT') &&
      checkpointLevelFromSubStepLabel(ss.label) === 'high',
  );
}
```

3c. Modify `buildPromptFromStep` signature to accept optional `worktreePath: string | undefined`:

```typescript
function buildPromptFromStep(step: RunbookStep, runbookPath: string, worktreePath?: string): string
```

After building `header`, if `worktreePath` is defined, append it:

```typescript
const worktreeContext = worktreePath? `WORKTREE_PATH=${worktreePath}` : null;
const header = [skillInstruction, identity,...(contextLine? [contextLine] : []),...(worktreeContext? [worktreeContext] : [])].join('\n');
```

3d. In `runLane`, within the `step.type === 'agent'` block, before the existing auto-claim block (which precedes `laneActiveStep.set`), add worktree provisioning:

```typescript
// ARC-1368: Auto-provision git worktree for HIGH stories before agent dispatch.
let worktreePath: string | undefined;
if (isHighStory(step)) {
  const storyKey = stripNamespace(step.id);
  worktreePath = `/repos/${storyKey}`;
  provisionWorktree(storyKey);  // throws on failure → caught by lane error handler
}

const prompt = buildPromptFromStep(step, runbookPath, worktreePath);
```

Note: `buildPromptFromStep` is currently called at the top of the agent block, before the auto-claim code. Move the `const prompt = buildPromptFromStep(...)` call to after the worktree provisioning so it can receive `worktreePath`.

**Order of operations in the agent step block:**
1. `provisionWorktree` (new — ARC-1368)
2. `buildPromptFromStep` with `worktreePath` (moved to after step 1)
3. Auto-claim: `markClaimedSubStepInMarkdown` + `void commitAndPushClaim(...)` (existing — ARC-1367)
4. `laneActiveStep.set`
5. `notifyLaneStep`
6. `executeStep`

**Verification:** TypeScript compiles. Unit tests (Step 4) pass.

---

### Step 4: Unit tests

**File:** `packages/server/src/__tests__/laneRunner.test.ts` (extend existing)

**Action:**

Add a `vi.mock('../git/worktreeGit.js')` factory at the top of the file (alongside existing mocks) with `provisionWorktreeMock`.

Add a new `describe` block: `'runLane — auto-provision worktree for HIGH stories (ARC-1368)'` containing:

1. **Worktree provisioned for HIGH story:** Given an agent step with an unchecked HIGH CHECKPOINT sub-step, when `runLane` executes it, `provisionWorktreeMock` is called with the correct story key before `executeStepMock` is called.

2. **Correct worktree path injected in prompt:** `buildPromptFromStep` (tested via the prompt passed to `executeStep`) contains `WORKTREE_PATH=/repos/ARC-1368`.

3. **Worktree NOT provisioned for non-HIGH story:** Given an agent step with no HIGH checkpoint sub-step (or a MEDIUM batch step), `provisionWorktreeMock` is NOT called.

4. **Worktree NOT provisioned for already-checked HIGH checkpoint:** Given an agent step whose HIGH CHECKPOINT sub-step is already checked (`checked: true`), `provisionWorktreeMock` is NOT called.

5. **Worktree failure is hard error:** `provisionWorktreeMock` throws. Expect `runLane` to pause the lane (status becomes `'paused'`) and push a checkpoint entry, not call `executeStep`.

Also add a new file: `packages/server/src/__tests__/worktreeGit.test.ts`

Test `provisionWorktree`:

1. **Creates new worktree:** `git worktree list` returns no matching path; branch does not exist → calls `git worktree add /repos/ARC-1368 -b ARC-1368 main`.

2. **Reuses existing worktree:** `git worktree list` output includes `worktree /repos/ARC-1368` → returns without calling `worktree add`.

3. **Reuses existing branch:** `git worktree list` has no match; `rev-parse` succeeds → calls `git worktree add /repos/ARC-1368 ARC-1368` (no `-b`).

4. **Propagates failure:** `git worktree add` throws → error propagates from `provisionWorktree`.

5. **Resolves APP_REPO_ROOT from env:** Sets `process.env.APP_REPO_ROOT = '/custom/path'`; confirms `git -C /custom/path` is used in the `execSync` calls.

Use `vi.mock('child_process')` and mock `execSync` per test case.

---

### Step 5: Update `start-execution-session` skill

**File:** `/home/zimmermann/.config/opencode/skills/start-execution-session/SKILL.md`

**Action:**

Remove the instructions that tell agents to manually run `git worktree list` and `git worktree add`. The conductor now handles this before the agent receives its first prompt. Replace those instructions with a note:

> The conductor auto-provisions a git worktree at `/repos/<KEY>` before dispatching your first prompt. You do not need to run `git worktree list` or `git worktree add` — the worktree already exists when your session starts.

---

## Test Strategy

- Unit tests are the primary verification mechanism (Step 4).
- `worktreeGit.test.ts`: exercises all branching paths of `provisionWorktree` via mocked `execSync`.
- `laneRunner.test.ts` additions: verify ordering (worktree before claim before dispatch), HIGH detection, non-HIGH bypass, failure handling.
- Run: `npm test` in `packages/server`.
- TypeScript: `npm run build` in `packages/server` must pass with no type errors.

## Files Touched

| File | Change |
|------|--------|
| `packages/server/src/git/worktreeGit.ts` | New — `provisionWorktree` function |
| `packages/server/src/__tests__/worktreeGit.test.ts` | New — unit tests for `provisionWorktree` |
| `packages/server/src/runner/laneRunner.ts` | Add `isHighStory`, update `buildPromptFromStep` signature, add worktree provisioning call in `runLane` |
| `packages/server/src/__tests__/laneRunner.test.ts` | New describe block for ARC-1368 worktree tests |
| `.env.example` | Add `APP_REPO_ROOT` comment + value |
| `/home/zimmermann/.config/opencode/skills/start-execution-session/SKILL.md` | Remove manual worktree ceremony instructions |
