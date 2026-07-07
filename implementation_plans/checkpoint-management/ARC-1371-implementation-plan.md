# ARC-1371 Implementation Plan: Conductor — Open PR for Implementation Plans and Decision Documents as Approval Gate

**Date:** 2026-07-07
**Issue:** `issues/checkpoint-management/ARC-1371-conductor-pr-approval-gate-for-plans-and-decisions.md`
**Lane:** checkpoint-management
**Epic:** ARC-1295 — ARC: Checkpoint Management

## Goal

Extend the conductor so that when an agent writes an implementation plan or a decision document, the artifact is committed to a short-lived branch and a Bitbucket PR is opened automatically. The lane pauses at a 🔴 checkpoint; PR merge is the canonical approval signal that resumes the lane. This eliminates the manual `status: answered` editing pattern and gives supervisors inline Bitbucket comment capability on every planning artifact.

## Pre-conditions / Assumptions

| # | Pre-condition | How to verify |
|---|---------------|---------------|
| 1 | ARC-1366 is merged — `resume_checkpoint` endpoint exists and polls PR state for lane resume | `git log --oneline` in `pa.aid.conductor.ts`; check for ARC-1366 in commits |
| 2 | ARC-1369 is merged — `commitPlanningArtifacts` function exists in `checkpointGit.ts` and is wired into `laneRunner.ts` | Check `checkpointGit.ts` exports; grep for `commitPlanningArtifacts` in `laneRunner.ts` |
| 3 | `BITBUCKET_USERNAME` and `BITBUCKET_PASSWORD` env vars are set in the running server process | Confirmed pattern used by `createCheckpointBranchAndPR` |
| 4 | The planning repo's git remote is a reachable Bitbucket URL | Auto-detected via `git remote get-url origin` — same as existing checkpoint PR logic |
| 5 | `SessionConfig` can be extended with a `prPollIntervalMs` field without a version bump | Schema allows new optional fields |

**Dependency note:** This plan is written against the expected post-ARC-1369 codebase. If ARC-1369 is not yet merged at execution time, step 2 may need to be adjusted to account for a different signature for planning-artifact commit logic. Verify before starting.

## Architecture Decision

**Decision:** Extend `checkpointGit.ts` with two new exported functions — `createPlanningArtifactBranchAndPR` (shared core) plus thin wrappers `createPlanPR` and `createDecisionPR`. Do **not** introduce a separate `planningArtifactGit.ts` module.

**Rationale:**
- The issue constraint explicitly states: "Must reuse the existing `checkpointGit.ts` PR-creation logic — do not introduce a second Bitbucket API client."
- The private helpers `createBitbucketPR`, `getBitbucketCredentials`, `parseRemoteUrl`, `parsePrId`, and `parsePrUrlRepo` are already defined in `checkpointGit.ts`; co-location avoids making them exported or duplicated.
- The new functions follow the exact same pattern as `createCheckpointBranchAndPR`: branch creation → push → fire-and-forget Bitbucket API call → return `{ branch, prUrl? }`.

**Decision:** PR merge detection reuses the existing `resume_checkpoint` polling mechanism introduced by ARC-1366. No new polling loop is added in this story; instead, `CheckpointQueueEntry.prUrl` is populated with the planning PR URL so the existing resume route polls it.

**Decision:** Artifact commit to the planning branch happens inside the new `createPlanPR` / `createDecisionPR` functions (not in `laneRunner.ts`), keeping git operations centralized in `checkpointGit.ts`.

## Implementation Steps

### Step 1 — Add `prPollIntervalMs` to `SessionConfig`

**File:** `packages/server/src/state/types.ts`

Add an optional field to `SessionConfig`:

```typescript
/**
 * Polling interval in milliseconds for checking Bitbucket PR merge status.
 * Used by the planning-artifact PR resume poller (ARC-1371).
 * Default: 30000 (30 seconds).
 */
prPollIntervalMs?: number;
```

Update `DEFAULT_SESSION_CONFIG`:

```typescript
prPollIntervalMs: 30000,
```

**Why here:** The issue constraint specifies the polling interval must be configurable. `SessionConfig` is the existing home for all tunable conductor parameters. The default of 30 s matches the issue constraint's stated default.

### Step 2 — Add `CreatePlanningArtifactPROptions` and `CreatePlanningArtifactPRResult` interfaces to `checkpointGit.ts`

**File:** `packages/server/src/git/checkpointGit.ts`

Add near the top of the file, after the existing `CreateCheckpointBranchAndPRResult` interface:

```typescript
// -----------------------------------------------------------------
// ARC-1371: Planning artifact PR types
// -----------------------------------------------------------------

export interface CreatePlanPROptions {
  /** Absolute path of the planning repo on disk */
  repoRoot: string;
  /** Lane identifier (e.g. "checkpoint-management") */
  lane: string;
  /** Story key (e.g. "ARC-1371") */
  storyKey: string;
  /** Story title — used in the PR title */
  storyTitle: string;
  /** Absolute path of the implementation plan file to commit */
  artifactPath: string;
}

export interface CreateDecisionPROptions {
  /** Absolute path of the planning repo on disk */
  repoRoot: string;
  /** Story key (e.g. "ARC-1371") */
  storyKey: string;
  /** Decision type suffix (e.g. "approach-decision") */
  decisionType: string;
  /** Absolute path of the decision document file to commit */
  artifactPath: string;
}

export interface CreatePlanningArtifactPRResult {
  /** Git branch that was created and pushed */
  branch: string;
  /** Bitbucket PR URL, or undefined if PR creation failed */
  prUrl?: string;
}
```

### Step 3 — Implement `createPlanPR` in `checkpointGit.ts`

**File:** `packages/server/src/git/checkpointGit.ts`

Add the following exported function after `createCheckpointBranchAndPR`:

```typescript
/**
 * Creates a planning-repo branch for an implementation plan, commits the plan file,
 * pushes the branch, and opens a Bitbucket PR for supervisor review.
 *
 * Branch naming: `plan/{lane}/{storyKey}`  (e.g. `plan/checkpoint-management/ARC-1371`)
 * PR title: `Plan review: {storyKey} — {storyTitle}`
 * PR description: includes artifact path, lane, story key, and review instructions.
 *
 * Implementation plan commits happen on the new branch (not on main).
 * PR merge is the approval signal; the conductor resumes the lane via the existing
 * resume_checkpoint polling mechanism (ARC-1366).
 *
 * Fallback: if PR creation fails, the artifact is still committed to the branch and
 * the function returns `{ branch, prUrl: undefined }`. The caller raises the checkpoint
 * without a PR URL, notifying the supervisor via the checkpoint queue.
 */
export async function createPlanPR(
  opts: CreatePlanPROptions,
): Promise<CreatePlanningArtifactPRResult> {
  const { repoRoot, lane, storyKey, storyTitle, artifactPath } = opts;
  const branch = `plan/${lane}/${storyKey}`;

  const execOpts = {
    encoding: 'utf8' as const,
    stdio: 'pipe' as const,
    timeout: 10000,
    cwd: repoRoot,
  };

  // Ensure we are on main before branching
  execSync('git checkout main', execOpts);

  // Create or reuse branch
  let branchExistsLocally = false;
  try {
    execSync(`git rev-parse --verify ${branch}`, execOpts);
    branchExistsLocally = true;
  } catch {
    // branch does not exist locally — will create it below
  }

  if (branchExistsLocally) {
    execSync(`git checkout ${branch}`, execOpts);
  } else {
    execSync(`git checkout -b ${branch}`, execOpts);
  }

  // Stage and commit the plan artifact on the branch
  const relPath = artifactPath.startsWith(repoRoot)
   ? artifactPath.slice(repoRoot.length + 1)
    : artifactPath;
  execSync(`git add -- "${relPath}"`, execOpts);
  execSync(
    `git commit -m "docs(${lane}): add ${storyKey} implementation plan for review"`,
    execOpts,
  );

  // Push branch (only if not already on remote)
  const remoteRef = execSync(`git ls-remote --heads origin ${branch}`, execOpts).trim();
  if (remoteRef === '') {
    execSync(`git push origin ${branch}`, execOpts);
  } else {
    execSync(`git push origin ${branch}`, execOpts);
  }

  // Return to main
  execSync('git checkout main', execOpts);

  // Attempt PR creation — non-blocking on failure
  let prUrl: string | undefined;
  try {
    const { credentials } = getBitbucketCredentials();
    const remoteUrl = execSync(`git -C ${repoRoot} remote get-url origin`, {
      encoding: 'utf8',
      stdio: 'pipe',
      timeout: 10000,
    }).trim();
    const { workspace: detectedWorkspace, repoSlug } = parseRemoteUrl(remoteUrl);
    const workspace = process.env.BITBUCKET_WORKSPACE?? detectedWorkspace;
    const defaultBranch = process.env.BITBUCKET_DEFAULT_BRANCH?? 'main';

    const url = `https://api.bitbucket.org/2.0/repositories/${workspace}/${repoSlug}/pullrequests`;
    const body = JSON.stringify({
      title: `Plan review: ${storyKey} — ${storyTitle}`,
      description: [
        `Implementation plan for ${storyKey} ready for supervisor review.`,
        ``,
        `**Artifact:** \`${relPath}\
`,
        `**Lane:** ${lane}`,
        `**Story:** ${storyKey}`,
        ``,
        `Review the plan file in this PR. Merging this PR approves the plan and unblocks lane execution.`,
      ].join('\n'),
      source: { branch: { name: branch } },
      destination: { branch: { name: defaultBranch } },
      close_source_branch: true,
    });

    const response = await fetch(url, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', Authorization: `Basic ${credentials}` },
      body,
    });

    if (!response.ok) {
      const text = await response.text().catch(() => '');
      throw new Error(`Bitbucket API returned ${response.status}: ${text}`);
    }

    const data = (await response.json()) as { links?: { html?: { href?: string } } };
    prUrl = data.links?.html?.href;
    if (!prUrl) throw new Error('Bitbucket PR created but response contained no PR URL');
  } catch (err) {
    console.error(`[ARC-1371] Plan PR creation failed for branch ${branch}: ${String(err)}`);
  }

  return { branch, prUrl };
}
```

### Step 4 — Implement `createDecisionPR` in `checkpointGit.ts`

**File:** `packages/server/src/git/checkpointGit.ts`

Add immediately after `createPlanPR`:

```typescript
/**
 * Creates a planning-repo branch for a decision document, commits the file,
 * pushes the branch, and opens a Bitbucket PR for supervisor review.
 *
 * Branch naming: `decision/{storyKey}-{decisionType}`
 * PR title: `Decision: {storyKey} — {decisionType}`
 * PR description: includes artifact path, story key, decision type, and
 *   instruction to set `status: answered` and `selected_approach` before merging.
 *
 * Same fallback and return contract as createPlanPR.
 */
export async function createDecisionPR(
  opts: CreateDecisionPROptions,
): Promise<CreatePlanningArtifactPRResult> {
  const { repoRoot, storyKey, decisionType, artifactPath } = opts;
  const branch = `decision/${storyKey}-${decisionType}`;

  const execOpts = {
    encoding: 'utf8' as const,
    stdio: 'pipe' as const,
    timeout: 10000,
    cwd: repoRoot,
  };

  execSync('git checkout main', execOpts);

  let branchExistsLocally = false;
  try {
    execSync(`git rev-parse --verify ${branch}`, execOpts);
    branchExistsLocally = true;
  } catch { /* create below */ }

  if (branchExistsLocally) {
    execSync(`git checkout ${branch}`, execOpts);
  } else {
    execSync(`git checkout -b ${branch}`, execOpts);
  }

  const relPath = artifactPath.startsWith(repoRoot)
   ? artifactPath.slice(repoRoot.length + 1)
    : artifactPath;
  execSync(`git add -- "${relPath}"`, execOpts);
  execSync(
    `git commit -m "docs: add ${storyKey} ${decisionType} decision document for review"`,
    execOpts,
  );

  const remoteRef = execSync(`git ls-remote --heads origin ${branch}`, execOpts).trim();
  if (remoteRef === '') {
    execSync(`git push origin ${branch}`, execOpts);
  } else {
    execSync(`git push origin ${branch}`, execOpts);
  }

  execSync('git checkout main', execOpts);

  let prUrl: string | undefined;
  try {
    const { credentials } = getBitbucketCredentials();
    const remoteUrl = execSync(`git -C ${repoRoot} remote get-url origin`, {
      encoding: 'utf8',
      stdio: 'pipe',
      timeout: 10000,
    }).trim();
    const { workspace: detectedWorkspace, repoSlug } = parseRemoteUrl(remoteUrl);
    const workspace = process.env.BITBUCKET_WORKSPACE?? detectedWorkspace;
    const defaultBranch = process.env.BITBUCKET_DEFAULT_BRANCH?? 'main';

    const url = `https://api.bitbucket.org/2.0/repositories/${workspace}/${repoSlug}/pullrequests`;
    const body = JSON.stringify({
      title: `Decision: ${storyKey} — ${decisionType}`,
      description: [
        `Decision document for ${storyKey} awaiting supervisor input.`,
        ``,
        `**Artifact:** \`${relPath}\
`,
        `**Story:** ${storyKey}`,
        `**Type:** ${decisionType}`,
        ``,
        `To approve: edit the file in this PR to set \`status: answered\` and \`selected_approach: Option N\`, then merge.`,
        `The conductor will read the merged file from \`main\` to extract \`selected_approach\`.`,
      ].join('\n'),
      source: { branch: { name: branch } },
      destination: { branch: { name: defaultBranch } },
      close_source_branch: true,
    });

    const response = await fetch(url, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', Authorization: `Basic ${credentials}` },
      body,
    });

    if (!response.ok) {
      const text = await response.text().catch(() => '');
      throw new Error(`Bitbucket API returned ${response.status}: ${text}`);
    }

    const data = (await response.json()) as { links?: { html?: { href?: string } } };
    prUrl = data.links?.html?.href;
    if (!prUrl) throw new Error('Bitbucket PR created but response contained no PR URL');
  } catch (err) {
    console.error(`[ARC-1371] Decision PR creation failed for branch ${branch}: ${String(err)}`);
  }

  return { branch, prUrl };
}
```

### Step 5 — Add `isPRMerged` polling helper to `checkpointGit.ts`

**File:** `packages/server/src/git/checkpointGit.ts`

Add after `createDecisionPR`. This function is used by the lane poller in step 6.

```typescript
/**
 * Checks whether a Bitbucket PR has been merged.
 * Fetches the PR state from the Bitbucket Cloud REST API.
 * Returns `true` if `state === "MERGED"`, `false` otherwise.
 * Throws on HTTP or network errors — callers must catch.
 */
export async function isPRMerged(prUrl: string): Promise<boolean> {
  const { credentials } = getBitbucketCredentials();
  const prId = parsePrId(prUrl);
  const { workspace, repoSlug } = parsePrUrlRepo(prUrl);

  const url = `https://api.bitbucket.org/2.0/repositories/${workspace}/${repoSlug}/pullrequests/${prId}`;

  const response = await fetch(url, {
    method: 'GET',
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Basic ${credentials}`,
    },
  });

  if (!response.ok) {
    const text = await response.text().catch(() => '');
    throw new Error(`Bitbucket API returned ${response.status}: ${text}`);
  }

  const data = (await response.json()) as { state?: string };
  return data.state === 'MERGED';
}
```

### Step 6 — Add `startPlanningPRPoller` to `checkpointGit.ts`

**File:** `packages/server/src/git/checkpointGit.ts`

Add after `isPRMerged`. This drives the PR-merge detection loop without adding a new polling mechanism elsewhere:

```typescript
/**
 * Starts a polling loop that checks a planning-artifact PR for merge every
 * `intervalMs` milliseconds. When the PR is merged, `onMerged` is called once
 * and the polling stops.
 *
 * Returns a `stop` function that cancels polling (used in tests or if the lane
 * is externally resumed/cancelled before the PR merges).
 *
 * Error handling: individual poll failures are logged and skipped — the loop
 * continues until stopped or the PR is merged.
 */
export function startPlanningPRPoller(opts: {
  prUrl: string;
  intervalMs: number;
  onMerged: () => void;
}): { stop: () => void } {
  const { prUrl, intervalMs, onMerged } = opts;
  let active = true;

  const tick = async () => {
    if (!active) return;
    try {
      const merged = await isPRMerged(prUrl);
      if (merged && active) {
        active = false;
        onMerged();
        return;
      }
    } catch (err) {
      console.error(`[ARC-1371] isPRMerged poll failed for ${prUrl}: ${String(err)}`);
    }
    if (active) {
      setTimeout(() => { void tick(); }, intervalMs);
    }
  };

  setTimeout(() => { void tick(); }, intervalMs);

  return { stop: () => { active = false; } };
}
```

### Step 7 — Wire `createPlanPR` / `createDecisionPR` into `laneRunner.ts`

**File:** `packages/server/src/runner/laneRunner.ts`

**Context:** ARC-1369 added `commitPlanningArtifacts` logic. That function currently commits plan/decision files directly to `main`. ARC-1371 replaces that direct-to-main commit path with a branch + PR path for plans and decisions.

**7a. Update imports** — add to the existing `checkpointGit` import line:

```typescript
import {
  createCheckpointBranchAndPR,
  createPlanPR,
  createDecisionPR,
  startPlanningPRPoller,
} from '../git/checkpointGit.js';
```

**7b. Add a `planningPRPollers` registry** — track active pollers for cleanup:

```typescript
/**
 * Active planning PR pollers keyed by PR URL.
 * Stopped when a lane resumes or is done.
 */
const planningPRPollers = new Map<string, { stop: () => void }>();
```

**7c. Add `triggerPlanningArtifactPR` helper function** in `laneRunner.ts`:

This function replaces the direct `commitPlanningArtifacts` call for plan and decision artifacts. It:
1. Calls `createPlanPR` or `createDecisionPR` (determined by `artifactType` parameter).
2. Pushes a `CheckpointQueueEntry` with `checkpointLevel: 'high'` and the `prUrl`.
3. Starts a `planningPRPoller` that calls `resumeLaneFromPlanningPR` on merge.
4. Sets lane status to `'paused'`.

```typescript
type PlanningArtifactType = 'plan' | 'decision';

interface TriggerPlanningArtifactPROpts {
  runbookPath: string;
  artifactType: PlanningArtifactType;
  lane: string;
  storyKey: string;
  storyTitle: string;
  decisionType?: string;   // required when artifactType === 'decision'
  artifactPath: string;
  repoRoot: string;
  intervalMs: number;
}

async function triggerPlanningArtifactPR(opts: TriggerPlanningArtifactPROpts): Promise<void> {
  const {
    runbookPath, artifactType, lane, storyKey, storyTitle,
    decisionType, artifactPath, repoRoot, intervalMs,
  } = opts;

  let result: { branch: string; prUrl?: string };

  if (artifactType === 'plan') {
    result = await createPlanPR({ repoRoot, lane, storyKey, storyTitle, artifactPath });
  } else {
    result = await createDecisionPR({
      repoRoot, storyKey, decisionType: decisionType!, artifactPath,
    });
  }

  const compositeId = `${storyKey}__plan_pr`;
  const current = getState();
  const entry: CheckpointQueueEntry = {
    stepId: compositeId,
    runbookPath,
    checkpointLevel: 'high',
    hitAt: new Date().toISOString(),
    prUrl: result.prUrl,
  };
  const updated = {...current, checkpointQueue: [...current.checkpointQueue, entry] };
  setState(updated);
  writeSidecar(current.repoRoot, updated);

  laneStatusMap.set(runbookPath, 'paused');

  if (result.prUrl) {
    const poller = startPlanningPRPoller({
      prUrl: result.prUrl,
      intervalMs,
      onMerged: () => {
        planningPRPollers.delete(result.prUrl!);
        resumeLaneFromPlanningPR(compositeId, runbookPath, artifactType, storyKey);
      },
    });
    planningPRPollers.set(result.prUrl, poller);
  } else {
    // Fallback: no prUrl — checkpoint is raised but lane stays paused until
    // supervisor manually calls POST /api/checkpoints/:stepId/resume
    console.warn(
      `[ARC-1371] Planning artifact PR creation failed for ${storyKey}. ` +
      `Lane ${lane} paused — supervisor must resume manually via checkpoint queue.`,
    );
  }
}
```

**7d. Add `resumeLaneFromPlanningPR` helper** in `laneRunner.ts`:

```typescript
/**
 * Called by the planning PR poller when a plan or decision PR is merged.
 * Removes the checkpoint entry from the queue, reads the merged artifact from
 * main if this is a decision PR (to extract selected_approach), persists state,
 * and re-runs the lane from disk.
 */
function resumeLaneFromPlanningPR(
  compositeId: string,
  runbookPath: string,
  artifactType: PlanningArtifactType,
  storyKey: string,
): void {
  const current = getState();
  const updatedQueue = current.checkpointQueue.filter((e) => e.stepId!== compositeId);
  const updated = {...current, checkpointQueue: updatedQueue };
  setState(updated);
  writeSidecar(current.repoRoot, updated);

  if (artifactType === 'decision') {
    // Decision documents: the merged file on main now contains status/selected_approach.
    // The next agent invocation will read it from disk — no special extraction needed here;
    // the agent prompt (built by buildPromptFromStep) will include the file path.
    console.info(`[ARC-1371] Decision PR merged for ${storyKey} — lane resuming, agent will read merged file from main.`);
  }

  try {
    const { parseRunbook } = require('../runbook/parser.js');
    const parsed = parseRunbook(runbookPath);
    const steps = parsed.waves.flatMap((wave: { steps: RunbookStep[] }) => wave.steps);
    void runLane(runbookPath, steps).catch((err: unknown) => {
      console.error(`[ARC-1371] runLane failed on planning PR resume for ${runbookPath}: ${String(err)}`);
    });
  } catch (parseErr: unknown) {
    console.warn(`[ARC-1371] parseRunbook failed on planning PR resume for ${runbookPath}: ${String(parseErr)}`);
  }
}
```

> **Note on `require` vs `import`:** `resumeLaneFromPlanningPR` is called from a closure (the `onMerged` callback) at runtime, not at module load time. Use the already-imported `parseRunbook` from the existing static import at the top of `laneRunner.ts`. If `parseRunbook` is not yet imported, add it to the existing parser import. Do not use `require`.

**Correction for 7d:** Replace the `require` call with the static import. The parser is already imported by `laneRunner.ts` (via `buildPromptFromStep` context) or must be added:

```typescript
// Add to imports at top of laneRunner.ts:
import { parseRunbook } from '../runbook/parser.js';
```

Then in `resumeLaneFromPlanningPR`, call `parseRunbook(runbookPath)` directly.

### Step 8 — Hook `triggerPlanningArtifactPR` into the agent-step post-execution path

**File:** `packages/server/src/runner/laneRunner.ts`

ARC-1369 wired a `commitPlanningArtifacts` call somewhere in the agent step success path (after `executeStep` resolves and before the CHECKPOINT/GATE sub-step check). Locate that call site.

**Replace** the ARC-1369 direct-commit logic for plans and decisions with a call to `triggerPlanningArtifactPR`. The replacement logic should:
1. After `executeStep` succeeds, scan the agent's output events for any newly written plan or decision file paths (ARC-1369 will have established a convention for this — look for a `planning_artifact_written` event type or similar signal in the step result).
2. For each such file, call `triggerPlanningArtifactPR(...)` with the appropriate `artifactType`.
3. If `triggerPlanningArtifactPR` was called, `return` immediately (the lane is now paused; the poller will resume it).

**If ARC-1369 established a different signal mechanism**, adapt accordingly. The core contract is:
- Plans → `createPlanPR` → branch `plan/{lane}/{KEY}` → PR titled `Plan review: {KEY} — {title}`
- Decisions → `createDecisionPR` → branch `decision/{KEY}-{type}` → PR titled `Decision: {KEY} — {type}`

The `intervalMs` comes from `getState().config.prPollIntervalMs?? 30000`.

### Step 9 — Update `CheckpointQueueEntry` in `types.ts` (if needed)

**File:** `packages/server/src/state/types.ts`

Check if `CheckpointQueueEntry` needs a new discriminator field to distinguish planning-PR checkpoints from execution checkpoints. The existing `prUrl` field is reused. Add an optional `planningArtifact` field for UI display:

```typescript
/**
 * Present on checkpoint entries created by planning-artifact PR opening (ARC-1371).
 * Used by the UI to display a more informative message in the ResumeCard.
 */
planningArtifact?: {
  artifactType: 'plan' | 'decision';
  storyKey: string;
  branch: string;
};
```

Update the `triggerPlanningArtifactPR` helper in step 7c to populate this field when creating the `CheckpointQueueEntry`.

### Step 10 — Update `resume` route to handle planning PR checkpoints

**File:** `packages/server/src/routes/checkpoints.ts`

The existing `POST /api/checkpoints/:stepId/resume` route already handles `prUrl`-based entries by calling `listUnresolvedPRComments`. For planning PR checkpoints, the comment-address loop is not appropriate — the artifact should be reviewed inline on Bitbucket, not addressed by the agent.

**Add a guard** at the top of the resume route handler:

```typescript
// ARC-1371: Planning-artifact PR checkpoints are resolved by PR merge via the
// startPlanningPRPoller, not by manual resume. Reject manual resume attempts
// with a clear error message.
if (entry.planningArtifact!== undefined && entry.prUrl!== undefined) {
  res.status(409).json({
    error: `This checkpoint is waiting for a planning PR to be merged on Bitbucket. ` +
           `Merge PR at ${entry.prUrl} to resume the lane automatically.`,
    prUrl: entry.prUrl,
  });
  return;
}
```

If the poller is not running (e.g. server was restarted after PR was created), the route should additionally restart the poller:

```typescript
// ARC-1371: Re-start the poller if server was restarted and the entry still has a prUrl
// (the poller lives only in memory and is lost on restart).
if (entry.planningArtifact!== undefined && entry.prUrl!== undefined) {
  // (handled above — fall through is unreachable; this comment is for dead-code clarity)
}
```

**Server restart recovery:** Add a startup hook (in `index.ts` or equivalent startup code) that scans the initial `checkpointQueue` for entries with `planningArtifact` set and `prUrl` present, and restarts pollers for them. The `onMerged` callback calls `resumeLaneFromPlanningPR` just as during normal operation.

**File:** `packages/server/src/index.ts` (or startup module)

```typescript
// ARC-1371: On startup, restart any planning PR pollers for entries that were
// in-flight when the server last shut down.
const initialState = getState();
for (const entry of initialState.checkpointQueue) {
  if (entry.planningArtifact && entry.prUrl) {
    const intervalMs = initialState.config.prPollIntervalMs?? 30000;
    startPlanningPRPoller({
      prUrl: entry.prUrl,
      intervalMs,
      onMerged: () => {
        resumeLaneFromPlanningPR(
          entry.stepId,
          entry.runbookPath,
          entry.planningArtifact!.artifactType,
          entry.planningArtifact!.storyKey,
        );
      },
    });
  }
}
```

> **Note:** `resumeLaneFromPlanningPR` must be exported from `laneRunner.ts` (or a new `planningPRResume.ts` module) for the startup code to call it. Make it an exported function.

### Step 11 — Update `write-implementation-plan` skill

**File:** `/home/zimmermann/.config/opencode/skills/write-implementation-plan/SKILL.md`

Remove the instructions that tell the agent to:
- Commit the implementation plan file manually with `git add... && git commit && git push`
- Stop and wait for supervisor approval of the plan document

Replace with instructions that tell the agent:
- Write the plan file to `implementation_plans/{lane}/{KEY}-implementation-plan.md`
- The conductor automatically opens a PR for the plan and pauses the lane — the agent does not commit the file manually
- The agent's step is complete when the file is written; the conductor handles the rest

### Step 12 — Update `start-execution-session` skill

**File:** `/home/zimmermann/.config/opencode/skills/start-execution-session/SKILL.md`

Remove references to:
- Manually editing `status: answered` in decision documents
- Manually committing decision documents

Replace with:
- Decision documents are written by the agent, committed to a branch, and submitted as a Bitbucket PR automatically by the conductor
- Supervisors answer decisions by editing the file in the PR and merging; the conductor reads the merged file
- Agents do not need to poll or check decision document status manually

## Test Plan

### Unit tests — `packages/server/src/git/checkpointGit.test.ts`

| Test | Description | Assertion |
|------|-------------|----------|
| `createPlanPR — success path` | Valid opts, fetch returns PR URL | Returns `{ branch: 'plan/checkpoint-management/ARC-1371', prUrl: 'https://...' }` |
| `createPlanPR — branch naming` | storyKey `ARC-1371`, lane `checkpoint-management` | Branch is exactly `plan/checkpoint-management/ARC-1371` |
| `createPlanPR — PR title` | storyTitle `Conductor: open PR...` | Fetch body contains `Plan review: ARC-1371 — Conductor: open PR...` |
| `createPlanPR — fallback on Bitbucket failure` | fetch returns 500 | Returns `{ branch, prUrl: undefined }`; does not throw |
| `createPlanPR — artifact committed on branch` | execSync calls inspected | `git add` and `git commit` are called after `git checkout -b` |
| `createDecisionPR — branch naming` | storyKey `ARC-1371`, decisionType `approach-decision` | Branch is `decision/ARC-1371-approach-decision` |
| `createDecisionPR — PR title` | inputs as above | Fetch body contains `Decision: ARC-1371 — approach-decision` |
| `createDecisionPR — fallback on Bitbucket failure` | fetch returns network error | Returns `{ branch, prUrl: undefined }` |
| `isPRMerged — returns true when state is MERGED` | fetch returns `{ state: 'MERGED' }` | Returns `true` |
| `isPRMerged — returns false when state is OPEN` | fetch returns `{ state: 'OPEN' }` | Returns `false` |
| `isPRMerged — throws on non-2xx` | fetch returns 403 | Throws with `403` in message |
| `startPlanningPRPoller — calls onMerged when merged` | isPRMerged resolves `true` after 1 tick | `onMerged` called exactly once |
| `startPlanningPRPoller — stop cancels polling` | stop() called before tick | `onMerged` never called after stop |
| `startPlanningPRPoller — continues on poll error` | isPRMerged throws once then returns true | `onMerged` still called on successful tick |

### Unit tests — `packages/server/src/__tests__/laneRunner.test.ts`

| Test | Description | Assertion |
|------|-------------|----------|
| `triggerPlanningArtifactPR — pauses lane and pushes checkpoint` | Mock `createPlanPR` to return prUrl | `laneStatusMap` is `'paused'`; checkpoint queue contains entry with `planningArtifact` |
| `triggerPlanningArtifactPR — starts poller when prUrl present` | Mock `startPlanningPRPoller` | Poller started with correct `intervalMs` and `prUrl` |
| `triggerPlanningArtifactPR — fallback path (no prUrl)` | Mock `createPlanPR` returns no prUrl | Lane still paused; checkpoint pushed; no poller started; warning logged |
| `resumeLaneFromPlanningPR — dequeues entry and runs lane` | Call directly | `checkpointQueue` no longer contains the entry; `runLane` called |

### Unit tests — `packages/server/src/__tests__/checkpoints.test.ts`

| Test | Description | Assertion |
|------|-------------|----------|
| `POST /checkpoints/:stepId/resume — planning PR checkpoint returns 409` | Entry has `planningArtifact` and `prUrl` | Response is 409 with `prUrl` in body |

### Integration smoke test

1. Start the server pointing at a test planning repo with a decision document branch already pushed.
2. Verify `GET /api/state` shows the checkpoint entry with `planningArtifact` and `prUrl` set.
3. Simulate PR merge by mocking `isPRMerged` to return `true`.
4. Verify lane transitions from `'paused'` to `'running'`.

## Rollback / Risk Notes

| Risk | Likelihood | Mitigation |
|------|-----------|------------|
| ARC-1369 commit logic differs from what this plan assumes | Medium | Step 8 notes explicitly flag the dependency; read ARC-1369 implementation before executing step 8 |
| `git checkout main` at start of `createPlanPR`/`createDecisionPR` fails if repo is in detached HEAD state | Low | Add a `git status` guard; fall back to `git checkout -` |
| Poller is lost on server restart before PR merges | Medium | Step 10 startup recovery handles this; verify in integration test |
| Planning PR from a prior run already exists (branch exists on remote) | Low | The `git push origin` call will fail with "already exists"; catch this error and skip push; poller still functions if the PR URL can be retrieved from the Bitbucket API by branch name |
| `planningArtifact` field absent on checkpoints persisted by previous server versions | Low | Field is optional; absent entries fall through to existing resume logic cleanly |
| Skills updated (step 11–12) break agents that expect old manual-commit instructions | Medium | Update both skills atomically; test with a dry-run agent session before marking complete |

## Commit Message Template

```
feat(checkpoint-management): open Bitbucket PR for planning artifacts as approval gate (ARC-1371)

- Add createPlanPR, createDecisionPR, isPRMerged, startPlanningPRPoller to checkpointGit.ts
- Add triggerPlanningArtifactPR and resumeLaneFromPlanningPR to laneRunner.ts
- Add planningArtifact discriminator field to CheckpointQueueEntry
- Add prPollIntervalMs to SessionConfig (default 30 s)
- Guard POST /checkpoints/:stepId/resume from manual resume of planning PR checkpoints
- Add startup recovery for in-flight planning PR pollers
- Update write-implementation-plan and start-execution-session skills to remove
  manual commit and status:answered instructions
```