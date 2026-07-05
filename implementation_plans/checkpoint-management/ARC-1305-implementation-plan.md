# ARC-1305 Display persistent [Resume] card and handle Resume click per lane — Implementation Plan

**Issue:** `issues/checkpoint-management/ARC-1305-approve-reject-checkpoint.md`
**Completion Summary:** `task-completions/ARC-1305-COMPLETION-SUMMARY.md` (TBD)
**Approach:** Single approach — established pattern (extend useCheckpointQueue + wire ResumeCard into Dashboard per queue entry)
**Owner:** build-agent
**Date:** 2026-07-05

## Scope & Alignment

This plan covers:
- Adding a `resumePost(stepId)` helper to `useCheckpointQueue` that calls `POST /api/checkpoints/:stepId/resume` and tracks per-stepId `resuming` state
- Rendering one `ResumeCard` per checkpoint queue entry in `Dashboard` (independent per lane, replaces the raw Resume buttons in `CheckpointQueuePanel`)
- Updating `useCheckpointQueue` to expose `resumingStepId` so `ResumeCard` can show the disabled "Resuming…" state while the POST is in flight
- Verifying card persistence: card stays until entry leaves the queue (next poll removes it from state), independent across lanes

AC mapping:
- AC 1 (card visible with PR link and context) → ResumeCard already renders prUrl, checkpointLevel, runbookLabel — wired to queue in Step 2
- AC 2 (Resume click triggers ARC-1304 flow) → Step 1 adds `resumePost` calling POST endpoint
- AC 3 (card persists until Resume clicked or lane cancelled) → Step 2 wires card to live queue; card disappears only when entry removed by server
- AC 4 (per-lane isolation) → Step 2 renders one card per entry with independent `resuming` state keyed by stepId

## Assumptions & Dependencies

- `ResumeCard` component (`packages/ui/src/components/ResumeCard.tsx`) is fully implemented (ARC-1302 ✅)
- `POST /api/checkpoints/:stepId/resume` route is implemented (ARC-1302 ✅, ARC-1304 ✅)
- `useCheckpointQueue` polls `/api/state` every 2s and exposes `queue` (ARC-1303 ✅)
- `resumeCheckpoint(stepId)` in `useCheckpointQueue` already calls POST — it will be superseded by the new `resumePost` which adds `resuming` tracking
- Dashboard currently renders `CheckpointQueuePanel` for the queue; ARC-1305 adds `ResumeCard` rendering per entry

## Implementation Steps

### Step 1: Add `resumePost` with per-stepId `resuming` tracking to `useCheckpointQueue`

**Files:** `packages/ui/src/hooks/useCheckpointQueue.ts`

**Action:**
- Add `resumingStepId: string | null` to the hook's return type (`UseCheckpointQueueResult`)
- Add `useState<string | null>(null)` for `resumingStepId` inside the hook
- Add `resumePost(stepId: string): Promise<void>` function:
  - Sets `resumingStepId` to `stepId`
  - POSTs to `/api/checkpoints/${encodeURIComponent(stepId)}/resume`
  - On non-2xx: throws with parsed error message
  - Clears `resumingStepId` in a `finally` block (regardless of success or error)
- Add `resumingStepId` and `resumePost` to the return object

**Verification:** TypeScript compiles with no errors; `resumePost` is callable from Dashboard

### Step 2: Render `ResumeCard` instances in `Dashboard` per queue entry

**Files:** `packages/ui/src/pages/Dashboard.tsx`

**Action:**
- Import `ResumeCard` from `@/components/ResumeCard`
- Destructure `resumePost` and `resumingStepId` from `useCheckpointQueue()` (already called in Dashboard)
- Replace or augment the current `CheckpointQueuePanel` section with a block that maps `checkpointQueue` to individual `ResumeCard` instances:
  - Each `ResumeCard` receives: `entry={entry}`, `onResume={resumePost}`, `resuming={resumingStepId === entry.stepId}`
  - Cards are rendered in a vertical stack (e.g., `space-y-3`) inside a container div with `mt-8`
  - Section heading "Suspended Checkpoints" above the card list
- The existing `CheckpointQueuePanel` section may remain or be removed — the per-lane `ResumeCard` rendering is the primary deliverable for ARC-1305

**Verification:** When queue has entries, each renders as an independent `ResumeCard`; clicking one shows "Resuming…" only for that card

### Step 3: Add per-stepId `resuming` isolation to `useCheckpointQueue` tests

**Files:** `packages/ui/src/hooks/useCheckpointQueue.test.ts`

**Action:**
- Add a `describe('useCheckpointQueue — resumePost')` block with:
  - Test: calls `POST /api/checkpoints/:stepId/resume` successfully
  - Test: sets `resumingStepId` to stepId while POST is in flight
  - Test: clears `resumingStepId` after POST resolves
  - Test: clears `resumingStepId` after POST rejects
  - Test: clicking resumePost for one stepId does not affect `resumingStepId` for a different stepId

**Verification:** All new tests pass; existing tests unaffected

### Step 4: Add integration tests for ResumeCard rendering in Dashboard

**Files:** `packages/ui/src/pages/Dashboard.test.tsx`

**Action:**
- Add test cases in a `describe('Dashboard — ResumeCard rendering')` block:
  - Test: when queue has one entry, one `ResumeCard` is rendered (`data-testid="resume-card"`
  - Test: when queue has two entries with different runbookPaths, two independent `ResumeCard` elements are rendered
  - Test: clicking Resume on one card does not affect the other card's state

**Verification:** Tests pass and correctly assert per-lane card isolation

## Testing & Validation

**Unit tests:**
- `useCheckpointQueue.test.ts`: run `pnpm test --filter=ui` — `resumePost` scenarios all pass
- `ResumeCard.test.tsx`: existing tests still pass (no changes to the component itself)
- `Dashboard.test.tsx`: new card rendering tests pass

**End-to-end smoke check:**
- Start server with a checkpoint entry in state; load UI; verify one `ResumeCard` per entry appears
- Add two entries with different runbookPaths; verify two independent cards; clicking one shows "Resuming…" only on that card

**Acceptance criteria verification:**
- AC 1: `ResumeCard` renders with `entry.prUrl` as View PR link and `entry.checkpointLevel` emoji visible
- AC 2: `onResume` calls `POST /api/checkpoints/:stepId/resume`; server triggers ARC-1304 thread-read flow
- AC 3: card remains until server removes entry from `checkpointQueue`; next poll (≤2s) removes it from UI
- AC 4: two entries in queue → two independent cards; `resumingStepId` keyed per stepId

## Risks & Open Questions

- `ResumeCard` was built in ARC-1302 but never rendered in the live app — integration tests in Step 4 will confirm it mounts correctly in Dashboard context
- The `resumePost` POST response (200 `{ ok: true }`) is fire-and-forget on the server side; the card will not disappear until the server actually removes it from the queue (which happens after threads are addressed). This is correct per AC 3.
- Dashboard currently renders `CheckpointQueuePanel` and `ResumeCard` instances may duplicate Resume UI. Decision: keep `CheckpointQueuePanel` for the overview list; add `ResumeCard` section above it (or replace — preference to keep both so the queue panel shows priority ordering and the cards provide the persistent per-lane action surface). If supervisor feedback during review recommends removing one, that is a cosmetic change out of scope for this story.