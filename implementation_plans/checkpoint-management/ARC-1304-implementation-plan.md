# ARC-1304 Read PR Comment Threads on Resume and Address as Required Action Items — Implementation Plan

**Issue:** `issues/checkpoint-management/ARC-1304-inline-edit-artifacts.md`
**Completion Summary:** `task-completions/ARC-1304-COMPLETION-SUMMARY.md` (TBD)
**Approach:** Bitbucket REST API for thread reads/resolves (extends existing checkpointGit.ts pattern); executeStep for per-comment agent execution; resume route extended with thread-check loop
**Owner:** build-agent
**Date:** 2026-07-05

---

## Scope & Alignment

This plan implements the full ARC-1304 checkpoint comment loop:

| AC | Coverage |
|----|----------|
| AC 1 — [Resume] clicked with unresolved threads → executor reads all unresolved threads | Step 1 + Step 3 |
| AC 2 — Each unresolved thread treated as required action item → addressed + committed | Step 2 + Step 4 |
| AC 3 — After addressing → reply on thread with commit SHA + resolve thread via Bitbucket API | Step 1 + Step 4 |
| AC 4 — All comments addressed → re-queue checkpoint, re-notify with "Addressed N comments — [view changes]" | Step 3 + Step 5 |
| AC 5 — [Resume] with zero unresolved threads → lane proceeds | Step 3 |

The plan does **not** touch: inline artifact editing, sidecar handoff file persistence, drag-and-drop queue reorder — those are out of scope per the issue.

---

## Assumptions & Dependencies

- ARC-1302 (PR creation, `prUrl` field on `CheckpointQueueEntry`) must be merged. The resume route and `createCheckpointBranchAndPR` are already in `packages/server/src/git/checkpointGit.ts` and `packages/server/src/routes/checkpoints.ts`.
- ARC-1303 (checkpoint queue panel) must be merged for re-notification UI.
- Bitbucket REST API is used for thread reads and resolution — same pattern as `createCheckpointBranchAndPR`. The server does not host MCP clients; "Bitbucket MCP" in the issue refers to the Bitbucket integration, which the server achieves via direct REST.
- PR number is recoverable from `prUrl` by parsing the numeric segment from `https://bitbucket.org/{workspace}/{repo}/pull-requests/{prId}`.
- "Addressing" a comment means spawning `executeStep` with the comment body as a prompt in the lane's repo root. The agent executes the instruction and its work is committed by whatever tooling it uses.
- After addressing, the commit SHA is obtained from `git log -1 --format="%H"` in `repoRoot`.
- Bitbucket env vars in scope: `BITBUCKET_USERNAME`, `BITBUCKET_PASSWORD`, `BITBUCKET_WORKSPACE`, `BITBUCKET_REPO_SLUG` (already present for ARC-1302).
- Bitbucket Cloud REST API v2.0 endpoints used:
  - List comments: `GET /2.0/repositories/{workspace}/{repo}/pullrequests/{id}/comments?q=resolved=false&pagelen=50`
  - Add comment: `POST /2.0/repositories/{workspace}/{repo}/pullrequests/{id}/comments`
  - Resolve comment thread: `PUT /2.0/repositories/{workspace}/{repo}/pullrequests/{id}/comments/{commentId}` with `{"resolved": true}`

---

## Implementation Steps

### Step 1: Extend `checkpointGit.ts` with Bitbucket PR comment thread functions

**Files:** `packages/server/src/git/checkpointGit.ts`

**Action:** Add three new exported async functions at the bottom of the module, following the existing `createBitbucketPR` helper pattern:

- `listUnresolvedPRComments(prUrl: string): Promise<BitbucketPRComment[]>` — reads all unresolved comment threads from the PR. Parses the PR ID from `prUrl`, calls `GET .../comments?q=resolved=false&pagelen=50`, returns an array of `BitbucketPRComment` objects. Throws on HTTP error (callers catch).
- `replyToThread(prUrl: string, inlineCommentId: number, message: string): Promise<void>` — posts a new comment reply on the PR with `parent.id = inlineCommentId`. Used to write "Addressed in commit {sha}".
- `resolveThread(prUrl: string, commentId: number): Promise<void>` — marks an existing thread resolved via `PUT .../comments/{commentId}` with `{"resolved": true}`.

Add a new exported interface:
```typescript
interface BitbucketPRComment {
  id: number;
  content: { raw: string };
  resolved: boolean;
}
```

The helper that extracts env vars and builds the credentials header should be extracted into a shared private `getBitbucketCredentials()` function (deduplicated from `createBitbucketPR`).

**Verification:** TypeScript compiles with no new errors (`pnpm --filter @runbook-executor/server typecheck`).

---

### Step 2: Add `fetchCommitSha(repoRoot: string): string` to `checkpointGit.ts`

**Files:** `packages/server/src/git/checkpointGit.ts`

**Action:** Add a non-exported helper that calls `execSync('git log -1 --format="%H"', { cwd: repoRoot, encoding: 'utf8', stdio: 'pipe', timeout: 5000 })` and returns the trimmed output. Returns an empty string on error (does not throw — reply message gracefully omits sha if unavailable).

**Verification:** Unit test in `packages/server/src/git/checkpointGit.test.ts` covers the happy-path and error cases via `execSync` mock.

---

### Step 3: Extend resume route with PR thread check gate

**Files:** `packages/server/src/routes/checkpoints.ts`

**Action:** Modify the `POST /api/checkpoints/:stepId/resume` handler. Before dequeuing and calling `runLane`, check if the checkpoint entry has a `prUrl`. If it does:

1. Call `listUnresolvedPRComments(entry.prUrl)` (await — this is the inner try/catch block added around the existing comment flow).
2. If the list is non-empty, do **not** remove the entry from the queue and do **not** call `runLane`. Instead, proceed to the comment-address loop (Step 4 below).
3. If the list is empty (or `prUrl` is absent), proceed with the existing dequeue + `runLane` logic unchanged.

The handler must remain synchronous in its HTTP response contract — it returns `200 { ok: true }` immediately in both cases, with comment-address work happening fire-and-forget (same pattern as `createCheckpointBranchAndPR` in laneRunner).

Keep the existing 404 and 500 error paths unchanged.

**Verification:** New test cases in `packages/server/src/__tests__/checkpoints.test.ts` covering: (a) no prUrl → runs existing path, (b) prUrl with zero unresolved → runs existing path, (c) prUrl with N unresolved → does not call runLane synchronously, returns 200.

---

### Step 4: Implement the comment-address loop (fire-and-forget async)

**Files:** `packages/server/src/routes/checkpoints.ts`

**Action:** Extract a new async function `addressPRComments(entry: CheckpointQueueEntry, unresolvedComments: BitbucketPRComment[], repoRoot: string): Promise<void>`. This function is called fire-and-forget from the resume route (Step 3) when there are unresolved threads.

Algorithm:
1. For each comment in `unresolvedComments`:
   a. Construct a prompt: `"Address the following code review comment:\n\n${comment.content.raw}"`.
   b. Call `await executeStep(entry.stepId + '__comment_' + comment.id, prompt)`.
   c. After `executeStep` resolves (success or failure), obtain `sha = fetchCommitSha(repoRoot)`.
   d. Call `await replyToThread(entry.prUrl, comment.id, sha ? \`Addressed in commit ${sha}\` : 'Addressed.')`.
   e. Call `await resolveThread(entry.prUrl, comment.id)`.
   f. Individual reply/resolve failures are caught and logged — do not abort remaining comments.
2. After all comments are processed:
   a. Count `n = unresolvedComments.length`.
   b. Update `CheckpointQueueEntry` in state to include `renotifyMessage: \`Agent addressed ${n} comment${n === 1 ? '' : 's'} — [view changes]\`` (Step 5 adds this field).
   c. The entry stays in the queue (it was never removed in Step 3).
   d. Persist updated state via `writeSidecar`.

Import `executeStep` from `../runner/stepRunner.js`.

**Verification:** Unit tests with mocked `executeStep`, `listUnresolvedPRComments`, `replyToThread`, `resolveThread`, and `writeSidecar`. Scenarios: 1 comment success, 2 comments with reply failure on second (continues), 0 comments (function returns immediately).

---

### Step 5: Add `renotifyMessage` field to `CheckpointQueueEntry`

**Files:**
- `packages/server/src/state/types.ts`
- `packages/ui/src/types/runbook.ts`

**Action:** Add optional field to `CheckpointQueueEntry`:
```typescript
renotifyMessage?: string;
```

Both files must be updated (server and client-side mirror).

**Verification:** TypeScript compiles on both packages. Existing tests unaffected (field is optional).

---

### Step 6: Display `renotifyMessage` in `ResumeCard`

**Files:** `packages/ui/src/components/ResumeCard.tsx`

**Action:** When `entry.renotifyMessage` is truthy, render it as a `<p>` element beneath the step label line (above the PR link). Use a distinct style — e.g. `text-xs text-amber-600 font-medium` — to distinguish it from the standard step label.

**Verification:** Updated test in `packages/ui/src/components/ResumeCard.test.tsx` covers: (a) no `renotifyMessage` → element absent, (b) `renotifyMessage` set → element present with correct text.

---

## Testing & Validation

Run after each implementation step:

```
pnpm --filter @runbook-executor/server test
pnpm --filter @runbook-executor/ui test
pnpm --filter @runbook-executor/server typecheck
pnpm --filter @runbook-executor/ui typecheck
```

End-to-end scenario (manual, localhost):
1. Start a session with a lane that has a checkpoint.
2. After checkpoint is hit, a PR is created (`prUrl` present in queue entry).
3. Add an unresolved comment to the PR via Bitbucket UI.
4. Click [Resume] in the checkpoint queue panel.
5. Verify: server logs show per-comment `executeStep` calls; threads appear resolved on Bitbucket; ResumeCard shows "Agent addressed N comments — [view changes]".
6. Click [Resume] again.
7. Verify: lane proceeds (runLane is invoked), checkpoint removed from queue.

---

## Risks & Open Questions

- **Bitbucket pagination:** `listUnresolvedPRComments` fetches up to 50 comments per call (`pagelen=50`). PRs with > 50 unresolved comments will silently miss items. For MVP this is acceptable — can add pagination in a follow-up.
- **`executeStep` for comment addressing:** Using the same `stepId` namespace as the lane step could cause `stepLog` key collisions. Step 4 uses `entry.stepId + '__comment_' + comment.id` as a unique composite key to avoid this.
- **Bitbucket resolve API availability:** The `PUT .../comments/{id}` with `{"resolved": true}` is available in Bitbucket Cloud REST API v2.0. Bitbucket Server uses a different endpoint — not in scope for this implementation.
- **Re-entry guard:** If supervisor clicks [Resume] again while `addressPRComments` is still running, the second call will read the same (or fewer) unresolved threads. This is safe — comments already resolved won't appear in the second list.
- **No MCP client in server:** The issue mentions "Bitbucket MCP" but the server does not run as an MCP host or client. All Bitbucket operations in the server use the direct REST API (same as `createCheckpointBranchAndPR`). This is consistent with the established architecture.
