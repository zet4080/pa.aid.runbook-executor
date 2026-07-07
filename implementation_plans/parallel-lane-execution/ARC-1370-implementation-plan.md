# ARC-1370 Conductor: auto-update issue status and archive story artifacts on completion — Implementation Plan

**Issue:** `issues/parallel-lane-execution/ARC-1370-conductor-auto-archive-on-completion.md`
**Completion Summary:** `task-completions/ARC-1370-COMPLETION-SUMMARY.md` (TBD)
**Approach:** New `planningArchiver.ts` module + lane-done hook in `laneRunner.ts`
**Owner:** agent
**Date:** 2026-07-07

## Scope & Alignment

This plan covers the automatic archiving of story artifacts when the conductor marks a lane as fully complete: frontmatter update, file moves to `done/` subdirectories, `done/README.md` update, and commit/push to planning repo `main`. This eliminates the `close-issue` skill ceremony from agent workflows.

AC mapping:

| AC | Step |
|----|------|
| AC1 — issue file `Status` → `Completed`, `Last Updated` → today | Step 1 (frontmatter patch) |
| AC2 — issue file moved `issues/{lane}/` → `done/issues/` | Step 1 (file move) |
| AC3 — implementation plan moved `implementation_plans/{lane}/` → `done/plans/` | Step 1 (file move) |
| AC4 — completion summary moved `task-completions/` → `done/completions/` | Step 1 (file move) |
| AC5 — completion summary missing → skip archive, log warning | Step 1 (guard) |
| AC6 — `done/README.md` updated + committed + pushed | Step 1 (README + git) |
| AC7 — `close-issue` skill updated | Step 4 |

## Assumptions & Dependencies

- ARC-1369 is merged: `commitPlanningArtifacts` exists in `checkpointGit.ts` and the auto-commit pattern is established.
- ARC-1358 (`update_issue_status` tool) and ARC-1360 (`archive_issue` tool) are live as reference implementations. This plan re-implements the same logic inline in a server-side module (Node.js `fs` synchronous APIs), not by calling the Bun-based tools.
- `getState().repoRoot` holds the absolute path of the planning repo at runtime.
- `stripNamespace` is exported from `markdownWriter.ts` — used to extract the raw story key from a namespaced step ID.
- `laneFromRunbookPath` is a private helper in `laneRunner.ts`; it must be exported or its logic inlined in the new module.
- Step IDs follow the pattern `{lane}__{KEY}` (e.g. `parallel-lane-execution__ARC-1370`). `stripNamespace` returns `ARC-1370`.
- The `done/` top-level directory already exists in the planning repo. The module must create missing subdirectories (`done/issues/`, `done/plans/`, `done/completions/`) via `mkdirSync` with `{ recursive: true }`.
- Git operations use `execSync` with `{ encoding: 'utf8', stdio: 'pipe', timeout: 10000, cwd: repoRoot }`.
- Archive failures must not fail the lane — all errors are caught and logged.

## Implementation Steps

---

### Step 1: Create `planningArchiver.ts` module

**Files:** `packages/server/src/git/planningArchiver.ts` (new file)

**Action:**

Import `execSync` from `child_process`, and `existsSync`, `readdirSync`, `renameSync`, `mkdirSync`, `readFileSync`, `writeFileSync`, `appendFileSync` from `fs`; `path` from `path`.

Export one function:

```typescript
export function archiveStoryArtifacts(
  repoRoot: string,
  storyKey: string,
  lane: string,
): void
```

Wrap the entire function body in try/catch — on any uncaught error: `console.warn('[ARC-1370] archiveStoryArtifacts failed: ' + String(err))` and return. This ensures the function never throws.

**Internals — in order:**

**1a. Locate artifacts:**

- `issueDir = path.join(repoRoot, 'issues', lane)` — scan with `readdirSync`, find file matching `${storyKey}-` prefix and `.md` suffix. Assign to `issueFile` (absolute path) or `null`.
- `planFile = path.join(repoRoot, 'implementation_plans', lane, '${storyKey}-implementation-plan.md')` — check `existsSync`. Set to `null` if absent (plan is optional per AC3/AC5 logic).
- `summaryDir = path.join(repoRoot, 'task-completions')` — scan, find file matching `${storyKey}-` prefix and `.md` suffix. Assign to `summaryFile` or `null`.

**1b. Guard — completion summary required (AC5):**

If `summaryFile` is `null`: `console.warn('[ARC-1370] summary missing for ' + storyKey + ' — skipping archive')` and return.

If `issueFile` is `null`: `console.warn('[ARC-1370] issue file missing for ' + storyKey + ' — skipping archive')` and return.

**1c. Idempotency guard:**

Check if destination `path.join(repoRoot, 'done', 'issues', path.basename(issueFile))` already exists via `existsSync`. If so: `console.warn('[ARC-1370] ' + storyKey + ' already archived — skipping')` and return.

**1d. Update issue frontmatter (AC1):**

Read `issueFile` content. Apply `patchIssueFrontmatter(content, today)` — a private helper in the same module — that replaces the `| Status | ... |` row with `| Status | Completed |` and upserts `| Last Updated | ${today} |`. The patch logic mirrors `patchFrontmatter` from `update_issue_status.ts`: split on `\n`, find the first `|` table block, replace/insert rows using exact `| Field | Value |` format. Write result back to `issueFile` using `writeFileSync`.

**1e. Create destination directories:**

```typescript
mkdirSync(path.join(repoRoot, 'done', 'issues'), { recursive: true });
mkdirSync(path.join(repoRoot, 'done', 'plans'), { recursive: true });
mkdirSync(path.join(repoRoot, 'done', 'completions'), { recursive: true });
```

**1f. Move files (AC2–AC4):**

```typescript
renameSync(issueFile, path.join(repoRoot, 'done', 'issues', path.basename(issueFile)));
if (planFile !== null) {
  renameSync(planFile, path.join(repoRoot, 'done', 'plans', path.basename(planFile)));
}
renameSync(summaryFile, path.join(repoRoot, 'done', 'completions', path.basename(summaryFile)));
```

**1g. Update `done/README.md` (AC6 — first half):**

Extract title: read the (already-moved) issue file from `done/issues/`, scan lines for first `# ` heading.

Ensure `done/README.md` has table header: check if it exists and contains `| Key |`; if not, prepend/append `| Key | Title | Completed | Issue File |\n|-----|-------|-----------|------------|`.

Append row: `| ${storyKey} | ${title} | ${today} | \`done/issues/${path.basename(issueFile)}\` |`.

**1h. Commit + push (AC6 — second half):**

Build `execOpts = { encoding: 'utf8' as const, stdio: 'pipe' as const, timeout: 10000, cwd: repoRoot }`.

```
git -C repoRoot add -A
git -C repoRoot commit -m "docs(close): archive ${storyKey}"
```

Then in a nested try/catch: `git push`. On push error: `console.warn('[ARC-1370] push failed: ' + String(err))`. Do not rethrow.

**Verification:** TypeScript compiles. Unit tests (Step 3) pass. The `done/` structure is created correctly in a temp directory.

---

### Step 2: Detect lane-done and call archiver in `laneRunner.ts`

**Files:** `packages/server/src/runner/laneRunner.ts`

**Action:**

Add import at top of file:

```typescript
import { archiveStoryArtifacts } from '../git/planningArchiver.js';
```

After the existing line `laneStatusMap.set(runbookPath, 'done')` (the last line of `runLane` before the function returns), add:

```typescript
// ARC-1370: auto-archive story artifacts on lane completion
const arcKey = steps
  .map((s) => stripNamespace(s.id))
  .find((id) => /^ARC-\d+$/.test(id));
if (arcKey !== undefined) {
  const doneState = getState();
  archiveStoryArtifacts(doneState.repoRoot, arcKey, laneFromRunbookPath(runbookPath));
}
```

`stripNamespace` is already imported from `markdownWriter.js` — no new import needed for it.

This call is synchronous (all file I/O inside `archiveStoryArtifacts` uses synchronous `fs` APIs and `execSync`). Errors are suppressed inside the function; this block never throws.

The ARC key search is ordered: it uses the first step in the array whose raw ID matches `ARC-\d+`. Runbook steps for a single-story lane always have this pattern (e.g. `parallel-lane-execution__ARC-1370`).

**Verification:** `laneRunner.test.ts` tests (Step 3) confirm `archiveStoryArtifactsMock` is called with the correct key after all steps complete, and not called when the lane pauses.

---

### Step 3: Unit tests

**File:** `packages/server/src/__tests__/planningArchiver.test.ts` (new file)

Use Vitest. Create a temp directory with `fs.mkdtempSync` for each test. Set up file fixtures by writing files into the temp dir structure before each test case. Test `archiveStoryArtifacts` with a real (non-mocked) filesystem but mocked `execSync`:

1. **Completion summary missing → warns and returns without moving files:** Create issue and plan files but no summary file → function returns, issue file remains in place, `git add` not called.
2. **Issue file missing → warns and returns:** No issue file in `issues/{lane}/` → function returns without side effects.
3. **Already archived (idempotent) → warns and returns:** Pre-create `done/issues/{file}` → function skips without overwriting.
4. **All files present → moved to correct destinations:** Issue, plan, and summary all present → after call, files exist at `done/issues/`, `done/plans/`, `done/completions/` respectively; originals absent.
5. **Issue frontmatter patched:** Issue file has `| Status | To Do |` → after move, content at `done/issues/` has `| Status | Completed |` and `| Last Updated | {today} |`.
6. **Implementation plan optional:** Issue + summary present but no plan file → function completes, two files moved, `done/plans/` not created for the plan.
7. **`done/README.md` updated:** After archive, `done/README.md` contains a table row with the story key, title, date, and issue file path.
8. **Commit called with `docs(close): archive {key}`:** Mock `execSync`, verify call sequence: `git add -A`, then `git commit -m "docs(close): archive ARC-1370"`.
9. **Push failure → warns, does not throw:** Inner `execSync` for `git push` throws → function catches, `console.warn` called, no rethrow.
10. **Outer error → warns, does not throw:** `readdirSync` throws → outer catch fires, function returns without throw.

**File:** `packages/server/src/__tests__/laneRunner.test.ts` (extend existing)

Add `archiveStoryArtifactsMock` to the `planningArchiver.js` mock factory (alongside existing mocks). Add `describe` block `'runLane — auto-archive on lane completion (ARC-1370)'`:

1. **Called when all steps complete:** Two agent steps both succeed, step IDs contain `ARC-1370` → `archiveStoryArtifactsMock` called with `(repoRoot, 'ARC-1370', 'test')`.
2. **NOT called when lane pauses (step fails):** Step fails, retries exhausted → lane pauses → archiver not called.
3. **NOT called when lane pauses (checkpoint gate hit):** Step succeeds but has unchecked CHECKPOINT sub-step → lane pauses → archiver not called.
4. **NOT called when step IDs contain no ARC key:** Step ID is `test__plain-step` (no ARC-NNNN) → archiver not called.
5. **Uses correct story key from steps:** Step ID `test__ARC-1370` → archiver called with `'ARC-1370'` as second argument.

**Verification:** All new tests pass. `npm test` in `packages/server/` is green.

---

### Step 4: Update `close-issue` skill

**Files:** `/home/zimmermann/.config/opencode/skills/close-issue/SKILL.md`

**Action:**

Prepend a prominent note after the `## Purpose` section:

> **Conductor automation note:** When running inside the conductor, story archiving is handled automatically at lane completion — issue status is patched, artifacts are moved, `done/README.md` is updated, and everything is committed to planning repo `main`. This skill is only needed for **manual archiving** outside conductor context (e.g. hand-edited runbooks, direct CLI invocations, or stories completed outside conductor lanes).

Keep the rest of the skill body intact — it remains the reference for manual archiving and the description of the artifact layout.

**Verification:** The skill file has the conductor note above `## Pre-Flight Check`. The existing checklist and table remain unchanged.

---

## Testing & Validation

- Run `npm test` in `packages/server/` — all tests including new `planningArchiver.test.ts` and the new `laneRunner.test.ts` describe blocks must pass.
- Run `npm run build` in `packages/server/` — TypeScript must compile with no errors.
- Manual smoke test: run a conductor session to completion against a test runbook; verify that after the lane reaches `done`, the issue file, plan, and summary appear in `done/issues/`, `done/plans/`, and `done/completions/` respectively, and `done/README.md` has the entry.

## Risks & Open Questions

- **Story key detection:** The ARC key scan uses the first step whose `stripNamespace(step.id)` matches `ARC-\d+`. Runbooks with multiple story steps per lane (unusual but possible) will archive only the first ARC story found. This is acceptable for the current runbook structure where each lane maps to one story. If multi-story lanes are introduced later, the archiver call site in `laneRunner.ts` must be revisited.
- **Frontmatter patch correctness:** The inline `patchIssueFrontmatter` helper must handle the exact table format used in issue files (`| Field | Value |` with single spaces). This is tested in Step 3, test case 5.
- **`done/README.md` encoding:** The file is read and written with synchronous `fs` APIs; consistent with existing patterns in `archive_issue.ts`. No encoding issues expected.
- **Race with ARC-1369 auto-commit:** `archiveStoryArtifacts` is called after `laneStatusMap.set(runbookPath, 'done')`, which is after the last `markStepCheckedInMarkdown` + `commitPlanningArtifacts` call. The archive fires after all ARC-1369 commits, so the completion summary should already be committed to planning repo `main` before the archive runs the `git add -A` that will pick it up.
