# ARC-1312 View Full Step Output History for Any Past Session — Implementation Plan
**Issue:** `issues/session-history/ARC-1312-step-output-history.md`
**Completion Summary:** `task-completions/ARC-1312-COMPLETION-SUMMARY.md` (TBD)
**Approach:** Single approach — extend ClosedSession with per-step output records; expose via GET endpoint; add SessionDetailPage replacing the ARC-1311 placeholder
**Owner:** build
**Date:** 2026-06-29

## Scope & Alignment

This plan covers all three acceptance criteria in ARC-1312:

- **AC 1:** When a past session is opened, all executed steps are listed with status and timestamp.
- **AC 2:** When a step is selected, the full agent output (event stream) is displayed.
- **AC 3:** When a checkpoint in a past session is viewed, supervisor feedback is shown alongside the artifact state at approval/rejection time.

ARC-1311 built `ClosedSession` (the archive entry) and `POST /api/sessions/close`. It did not include per-step output in `ClosedSession`. This plan extends `ClosedSession` with a `steps` array (per-step record including all emitted events and checkpoint feedback). It extends `POST /api/sessions/close` to snapshot that data into the archive. A new `GET /api/sessions/history/:sessionId` endpoint returns the detail. The UI adds a `SessionDetailPage` replacing the placeholder left by ARC-1311.

## Assumptions & Dependencies

- ARC-1311 is merged. `ClosedSession` type, `POST /api/sessions/close`, `GET /api/sessions/history`, `SessionHistoryPage`, `App.tsx` navigation skeleton, and `useSessionHistory` hook all exist.
- Per-step output events are currently captured in `RunResult.events` inside `stepRunner.ts` but are not yet persisted. This plan adds persistence: `stepRunner.ts` must write step events into the active session state so `sessions/close` can snapshot them.
- Checkpoint feedback is currently in `CheckpointQueueEntry` (a queue of pending checkpoints). Approved/rejected checkpoints are removed from the queue by the checkpoint-management stories (ARC-1301–ARC-1305, not yet merged). This plan stores a snapshot of the feedback at approval/rejection time into the step record — **but only populates this field if checkpoint-management stories have added feedback to the queue entry**. For sessions closed before ARC-1303–ARC-1305 land, `checkpointFeedback` will be absent (undefined/null) — acceptable per issue scope.
- A `stepStartedAt` timestamp is not yet stored for each step. This plan adds it to the active session's per-step record when `executeStep` begins, using the `Date.now()` value at the start of execution.
- Steps that were never executed (present in `stepState` with value `false`) appear in AC 1's list with status `"pending"` — no timestamp, no events.
- "Full agent output displayed" (AC 2) means rendering the `events` array. Events contain `type`, `timestamp`, and content fields. The UI renders each event as a structured line: event type + relevant text extracted from the event payload.
- No pagination is required by the issue — all events for a step are rendered at once.

## Implementation Steps

### Step 1: Define `StepRecord` and extend `ClosedSession`

**Files:** `packages/server/src/state/types.ts`

**Action:**
Add a `StepRecord` interface:
- `stepId: string`
- `status: 'success' | 'failed' | 'pending'`
- `startedAt: string | null` — ISO timestamp when execution began; null for never-run steps
- `completedAt: string | null` — ISO timestamp when `executeStep` resolved; null for pending
- `events: OpenCodeEvent[]` — all events emitted during the run, in order; empty array for pending
- `checkpointFeedback: string | null` — supervisor feedback text captured at checkpoint resolution; null if step was not a checkpoint or feedback not yet captured

Import `OpenCodeEvent` from `../runner/types.js` in `types.ts` (currently no import exists; add it).

Extend `ClosedSession` (already defined in ARC-1311) by adding:
- `steps: StepRecord[]` — ordered list of step records snapshotted at session close

Extend `SidecarState` with an in-progress step log:
- `stepLog: Record<string, StepRecord>` — keyed by `stepId`; populated during execution; snapshotted into `ClosedSession.steps` at close time. Optional field (default `{}`).

**Verification:** `npx tsc --noEmit` in `packages/server` passes. Existing tests continue to pass — `stepLog` is optional and defaults to `{}`.

---

### Step 2: Update `executeStep` to write step records into `stepLog`

**Files:** `packages/server/src/runner/stepRunner.ts`

**Action:**
Modify `executeStep` so that:
1. Before calling `runStep`, record `startedAt: new Date().toISOString()` for the step in `state.stepLog`.
2. After `runStep` resolves, update the step's `StepRecord` in `stepLog` with:
   - `status`: `result.outcome` (`'success'` or `'failed'`)
   - `completedAt: new Date().toISOString()`
   - `events: result.events`
   - `checkpointFeedback: null` (to be populated by checkpoint-management when implemented)
3. Persist both the `stepState` update (existing) and the updated `stepLog` via a single `writeSidecar` call (combine into one state object).

The `markRunning` / `markDone` calls are unchanged.

**Verification:** Unit test: after `executeStep` resolves, `getState().stepLog[stepId]` contains the correct `status`, `startedAt`, `completedAt`, and `events`. The existing `stepRunner.test.ts` tests continue to pass.

---

### Step 3: Snapshot `stepLog` into `ClosedSession.steps` in `POST /api/sessions/close`

**Files:** `packages/server/src/routes/sessions.ts`

**Action:**
In the `POST /api/sessions/close` handler (added by ARC-1311):
- Build `steps: StepRecord[]` from `state.stepLog`. Include all step IDs present in `state.stepState` — for step IDs in `stepState` but absent from `stepLog` (never executed), create a `StepRecord` with `status: 'pending'`, `startedAt: null`, `completedAt: null`, `events: []`, `checkpointFeedback: null`.
- Attach `steps` to the `ClosedSession` object before appending to `closedSessions`.
- Clear `stepLog` from the updated state after close (reset to `{}`).

**Verification:** Unit test: close with a `stepLog` containing one entry → `closedSessions[0].steps` contains the record. Close with steps in `stepState` but not in `stepLog` → those steps appear as `status: 'pending'`.

---

### Step 4: Add `GET /api/sessions/history/:sessionId` endpoint

**Files:** `packages/server/src/routes/sessions.ts`

**Action:**
Add a new route handler for `GET /api/sessions/history/:sessionId`:
- Looks up `state.closedSessions.find(s => s.sessionId === sessionId)`.
- Returns `404 { error: "Session not found" }` if no match.
- Returns `200` with the full `ClosedSession` object (including `steps`).

**Verification:** Unit test: session exists → returns full object. Unknown sessionId → 404. Empty `closedSessions` → 404.

---

### Step 5: Add `useSessionDetail` hook in the UI

**Files:** `packages/ui/src/hooks/useSessionDetail.ts` (new file)

**Action:**
Create a hook `useSessionDetail(sessionId: string | null)` that:
- Fetches `GET /api/sessions/history/:sessionId` when `sessionId` is non-null.
- Returns `{ session: ClosedSession | null, loading: boolean, error: string | null }`.
- Uses `AbortController` cleanup on unmount and re-fetch on `sessionId` change.
- Returns `{ session: null, loading: false, error: null }` when `sessionId` is null.

Add `StepRecord` type to `packages/ui/src/types/runbook.ts` as a client-side mirror. Extend `ClosedSession` type (added by ARC-1311) with `steps: StepRecord[]`.

**Verification:** Unit test: mock fetch returning a session → hook returns it. Unknown id (404) → sets error. Null sessionId → returns null session without fetching.

---

### Step 6: Add `SessionDetailPage` UI component

**Files:** `packages/ui/src/pages/SessionDetailPage.tsx` (new file)

**Action:**
Create a `SessionDetailPage` component that accepts:
- `sessionId: string` — the session to display
- `onBack: () => void` — navigates back to history list

The component:
- Calls `useSessionDetail(sessionId)`.
- Shows loading indicator while `loading` is true.
- Shows error alert (matching `Dashboard.tsx` pattern) on error.
- When session is loaded, displays a header with session date, status (via `StatusBadge`), and duration.
- Renders a step list (AC 1): one row per `StepRecord` in `session.steps`, showing `stepId`, status badge, and `startedAt` timestamp formatted as locale time. Steps with `status: 'pending'` show "Never run" in place of timestamp.
- Each step row is expandable (click to toggle): when expanded, shows:
  - Full event stream (AC 2): each `OpenCodeEvent` rendered as a line — `event.type` + extracted text. For `TextEvent`, render `event.part.text`. For `ToolUseEvent`, render tool name and status. For other event types, render the type name only.
  - Checkpoint feedback section (AC 3): visible only when `checkpointFeedback` is non-null; shows label "Supervisor feedback:" followed by the feedback text.
- Expanded state is local component state (`useState<string | null>` for the currently expanded `stepId`).

**Verification:** Renders step list correctly. Clicking a step row expands it and shows events. Steps with no events show the list but an empty events section. Checkpoint feedback section only appears when `checkpointFeedback` is set.

---

### Step 7: Wire `SessionDetailPage` into `App.tsx`

**Files:** `packages/ui/src/App.tsx`

**Action:**
Replace the placeholder `<div>` rendered for `view === 'session-detail'` (added by ARC-1311) with `<SessionDetailPage sessionId={selectedSessionId!} onBack={() => setView('history')} />`.

Import `SessionDetailPage` from `./pages/SessionDetailPage`.

**Verification:** From history list, clicking a session row navigates to the detail page. Clicking "Back" on detail page returns to history list. `selectedSessionId` is correctly passed through from `App.tsx` state (set by `onSelectSession` in `SessionHistoryPage`).

---

### Step 8: Add server-side unit tests

**Files:** `packages/server/src/__tests__/sessions.test.ts`, `packages/server/src/__tests__/stepRunner.test.ts`

**Action:**
In `sessions.test.ts`:
- `POST /api/sessions/close` with `stepLog` → `closedSessions[0].steps` contains the record; `stepLog` is cleared in state.
- `POST /api/sessions/close` with steps in `stepState` not in `stepLog` → those steps present as `status: 'pending'`.
- `GET /api/sessions/history/:sessionId` → returns full session including steps.
- `GET /api/sessions/history/:unknownId` → 404.

In `stepRunner.test.ts`:
- After `executeStep` resolves, `getState().stepLog[stepId]` contains correct fields.

**Verification:** `npm test` in `packages/server` passes all tests.

---

### Step 9: Add UI-side unit tests

**Files:**
- `packages/ui/src/hooks/useSessionDetail.test.ts` (new file)
- `packages/ui/src/pages/SessionDetailPage.test.tsx` (new file)

**Action:**
- `useSessionDetail.test.ts`: mock fetch returning session → hook returns it; 404 → error set; null sessionId → no fetch, null session returned.
- `SessionDetailPage.test.tsx`: renders step list; expanding a step shows events; checkpoint feedback section conditionally present; "Back" calls `onBack`; loading and error states render correctly.

**Verification:** `npm test` in `packages/ui` passes all tests.

## Testing & Validation

**Automated (run from `packages/server` and `packages/ui` with `npm test`):**
- All existing tests continue to pass (no regressions).
- New `stepRunner` test: `stepLog` populated after execution.
- New `POST /api/sessions/close` tests: steps snapshotted, pending steps included, `stepLog` cleared.
- New `GET /api/sessions/history/:sessionId` tests: found and not-found cases.
- `useSessionDetail` hook tests: success, error, null-id no-fetch.
- `SessionDetailPage` render tests: step list, expand/collapse, checkpoint feedback, navigation.

**Manual end-to-end scenario (builds on ARC-1311 manual scenario):**
1. Start server. Select runbooks. Start session. Execute at least one step (via `POST /api/steps/:stepId/run`).
2. Close session via "Close Session" button.
3. Open history. Click the closed session row. Confirm `SessionDetailPage` loads.
4. Verify step list shows the executed step with status and timestamp (AC 1).
5. Click the executed step row. Verify event stream appears (AC 2).
6. If a checkpoint was hit and feedback was provided, verify the "Supervisor feedback" section appears under that step (AC 3).
7. Click "Back". Confirm return to history list.

## Risks & Open Questions

- **Checkpoint feedback population:** AC 3 requires supervisor feedback to be shown. `checkpointFeedback` is defined on `StepRecord` and will be null until checkpoint-management stories (ARC-1303–ARC-1305) write feedback into the step record. This plan defines the storage contract; populating it is a dependency on those stories. Displaying null gracefully (hiding the section) is covered by Step 6.
- **Event volume per step:** A long-running step may emit hundreds or thousands of events. Rendering all events at once in the expanded row may cause performance issues. For the current scope, this is acceptable — the issue does not require virtualization. If performance becomes a problem, virtual scrolling can be added in a future story.
- **`SidecarState` growth:** `stepLog` accumulates events across all steps in the active session. Large event payloads (e.g. lengthy tool output) will grow the sidecar file. Clearing `stepLog` on session close (Step 3) bounds this to one session's worth of data at a time.
- **Import cycle risk:** `types.ts` importing `OpenCodeEvent` from `runner/types.ts` creates a new dependency from `state/` to `runner/`. If this causes a circular import at runtime, `OpenCodeEvent` can be re-declared inline in `state/types.ts` as a `type OpenCodeEvent = Record<string, unknown>` alias for the purposes of storage, avoiding the import entirely. The stored events need not be strongly typed at the state layer.
