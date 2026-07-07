`# ARC-1367 Conductor: auto-claim story in runbook before dispatching agent - Implementation Plan

**Issue:** `issues/parallel-lane-execution/ARC-1367-conductor-auto-claim-story.md`
**Completion Summary:** `task-completions/ARC-1367-COMPLETION-SUMMARY.md`
**Approach:** single approach — pattern established
**Owner:** build agent
**Date:** 2026-07-07

## Scope & Alignment

This plan covers implementing automatic story claiming in the conductor's `laneRunner.ts` before agent dispatch, adding the corresponding `markClaimedSubStepInMarkdown` function to `markdownWriter.ts`, adding a fire-and-forget git commit/push helper in a new `claimGit.ts` module, and removing the manual `runbook_claim_story` ceremony step from the `start-execution-session` skill.

| Acceptance Criterion | Step(s) |
|---|---|
| Claim checkbox is set before agent process starts | Step 1, Step 2, Step 3 |
| Change committed and pushed to planning repo `main` after claim | Step 2, Step 3 |
| Claim is idempotent — no-op if already claimed | Step 1 |
| `start-execution-session` skill no longer instructs manual `runbook_claim_story` call | Step 4 |
| Unit test: auto-claim fires before first agent prompt is built | Step 5 |

## Assumptions & Dependencies

- `RunbookSubStep` in `packages/server/src/runbook/types.ts` does not carry line numbers — the conductor types omit the `line` field present on the OpenCode tool's extended type. The `markClaimedSubStepInMarkdown` function must therefore use label-matching (same strategy as `markSubStepsCheckedInMarkdown`) rather than direct line indexing.
- `SidecarState.repoRoot` (available via `getState().repoRoot`) is the planning repo root path needed for `git -C {repoRoot} commit/push`.
- The `laneFromRunbookPath` helper already exists in `laneRunner.ts` and produces the lane name used in the commit message `chore({lane}): claim {KEY}`.
- Claim format written to the markdown sub-item: `- [x] 🔒 Claimed: {lane} / {ISO-8601 timestamp}` — same format as the `runbook_claim_story` OpenCode tool.
- Timestamp format: `new Date().toISOString().slice(0, 16).replace('T', ' ')` (YYYY-MM-DD HH:MM), matching the existing tool.
- Push failures must not block agent dispatch — log a warning and continue.
- ARC-1349 (`runbook_claim_story` MCP tool) remains available for manual edge-case use; this plan does not modify it.
- The `start-execution-session` skill file lives at `/home/zimmermann/.config/opencode/skills/start-execution-session/SKILL.md`.

## Implementation Steps

### Step 1: Add `markClaimedSubStepInMarkdown` to `markdownWriter.ts`

**File:** `packages/server/src/runbook/markdownWriter.ts`

**Action:** Add a new exported function `markClaimedSubStepInMarkdown(runbookPath, storyKey, lane, timestamp)`. The function reads the runbook file, scans for the first indented sub-step line that is both unchecked (`^\s+- \[ \]`) and matches `/claimed/i`, and replaces `- [ ]` with `- [x] 🔒 Claimed: {lane} / {timestamp}` preserving leading whitespace. If no such line is found the function logs a warning and returns without writing. If a matching line is found but the checkbox is already checked (`^\s+- \[x\]`) it returns immediately as a no-op (idempotent). Writes atomically using the existing `atomicWrite` helper. The `storyKey` parameter is used to scope the search: the function only scans lines within the step block whose top-level line contains `storyKey`. To scope the scan: find the top-level line containing `storyKey`, then scan only the subsequent indented lines (those beginning with whitespace) until the next non-indented, non-blank line is encountered.

**Verification:** The function is exported and callable. The `markdownWriter.test.ts` new test suite passes (see Step 5).

---

### Step 2: Add `commitAndPushClaim` to a new `claimGit.ts` module

**File:** `packages/server/src/git/claimGit.ts` (new file)

**Action:** Create a new module exporting one async function `commitAndPushClaim(repoRoot, runbookPath, lane, storyKey)`. The function performs three `execSync` operations in sequence: `git -C {repoRoot} add {runbookPath}`, `git -C {repoRoot} commit -m "chore({lane}): claim {storyKey}"