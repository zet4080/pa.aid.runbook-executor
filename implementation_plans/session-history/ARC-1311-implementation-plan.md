# ARC-1311 Browse Past Sessions List — Implementation Plan
**Issue:** `issues/session-history/ARC-1311-browse-past-sessions.md`
**Completion Summary:** `task-completions/ARC-1311-COMPLETION-SUMMARY.md` (TBD)
**Approach:** Single approach — extend SidecarState with a closed-sessions archive; expose via dedicated GET endpoint; add UI page with hook
**Owner:** build
**Date:** 2026-06-29

## Scope & Alignment

This plan covers both acceptance criteria in ARC-1311:

- **AC 1:** When the history view is opened, all past sessions are listed with date/time, runbooks included, final status, and duration.
- **AC 2:** When a past session entry is selected, the session detail view opens (navigation only — rendering detail is ARC-1312).

The active session is tracked in `SidecarState` (ARC-1287/ARC-1310). Past sessions are not currently stored anywhere. This plan adds a `closedSessions` archive array to `SidecarState` and a `POST /api/sessions/close` endpoint that snapshots the active session into the archive. A new `GET /api/sessions/history` endpoint returns the list. The UI adds a `SessionHistoryPage` and a `useSessionHistory` hook, with navigation wired through the existing single-page layout.

## Assumptions & Dependencies

- ARC-1310 is merged. `SidecarState`, `writeSidecar`, `readSidecar`, `getState`, `setState` are in place.
- A "session close" event is needed to populate the archive. The issue does not specify a trigger; this plan defines `POST /api/sessions/close` as the explicit close action. The UI exposes a "Close Session" button on the dashboard once a session has been started (i.e. `sessionStartedAt` is set).
- `SidecarState.version` is currently `1`. Adding `closedSessions` as an optional field with a default of `[]` does not require a version bump — existing sidecars without the field continue to parse correctly via the `?? []` default in `readSidecar`.
- Duration is computed as the difference between `sessionStartedAt` and the close timestamp; stored as integer seconds.
- Session status on close is either `"complete"` (all selected runbook steps checked) or `"partial"` (some steps remain unchecked). This is derived at close time from `stepState` vs. `totalSteps`.
- No concurrent-write concern for `closedSessions` — close is a single infrequent action, not a hot path.
- The `App.tsx` currently renders only `Dashboard`. Navigation between Dashboard and history will be added as a simple string state toggle in `App.tsx` (no router dependency added — ARC-1311 is scoped to "open the detail view", which can be a page state change within `App.tsx`).

## Implementation Steps

### Step 1: Define `ClosedSession` type and extend `SidecarState`

**Files:** `packages/server/src/state/types.ts`

**Action:**
Add a `ClosedSession` interface:
- `sessionId: string` — copied from active session
- `repoRoot: string`
- `closedAt: string` — ISO timestamp of close action
- `startedAt: string` — copied from `sessionStartedAt`
- `durationSeconds: number` — integer seconds between `startedAt` and `closedAt`
- `selectedRunbooks: string[]` — snapshot at close time
- `status: 'complete' | 'partial'` — derived at close: `complete` if all step values in `stepState` for selected runbooks are `true`, `partial` otherwise
- `stepState: Record<string, boolean>` — snapshot of `stepState` at close

Add `closedSessions?: ClosedSession[]` as an optional field on `SidecarState`. Callers treat absence as `[]`.

**Verification:** `npx tsc --noEmit` in `packages/server` passes. Existing tests using `makeState()` without `closedSessions` continue to compile and pass.

---

### Step 2: Implement `POST /api/sessions/close` in `sessions.ts`

**Files:** `packages/server/src/routes/sessions.ts`

**Action:**
Add a new route handler for `POST /api/sessions/close`:
- Returns `400 { error: "No active session" }` if `sessionStartedAt` is undefined on current state.
- Computes `closedAt` as `new Date().toISOString()`, `durationSeconds` as the difference in seconds between `closedAt` and `sessionStartedAt`.
- Derives `status`: iterate over `stepState` keys; if every value is `true`, status is `complete`, otherwise `partial`.
- Constructs a `ClosedSession` object from the current state.
- Appends it to `closedSessions` (defaulting to `[]` if absent).
- Clears `sessionStartedAt` from the updated state (session is now closed).
- Calls `writeSidecar` and `setState` with the updated state.
- Returns `200 { ok: true, session: ClosedSession }`.

**Verification:** Unit test: state with `sessionStartedAt` set → `POST /api/sessions/close` returns 200, `writeSidecar` called with state containing new entry in `closedSessions` and no `sessionStartedAt`. Unit test: no `sessionStartedAt` → returns 400.

---

### Step 3: Implement `GET /api/sessions/history` in `sessions.ts`

**Files:** `packages/server/src/routes/sessions.ts`

**Action:**
Add a new route handler for `GET /api/sessions/history`:
- Returns `200` with the `closedSessions` array from current state (defaulting to `[]`).
- Response shape: `ClosedSession[]` — the full array, ordered as stored (append order = chronological).

**Verification:** Unit test: state with two `closedSessions` entries → GET returns both in order. Unit test: empty `closedSessions` → GET returns `[]`.

---

### Step 4: Register `history` and `close` routes in `index.ts`

**Files:** `packages/server/src/index.ts`

**Action:**
The `sessionsRouter` is already registered on `/api` via `app.use('/api', sessionsRouter)`. No change needed in `index.ts` — new routes added to `sessionsRouter` are automatically mounted.

**Verification:** `curl -X POST http://localhost:4000/api/sessions/close` returns a JSON response (400 or 200 depending on state). `curl http://localhost:4000/api/sessions/history` returns a JSON array.

---

### Step 5: Add `useSessionHistory` hook in the UI

**Files:** `packages/ui/src/hooks/useSessionHistory.ts` (new file)

**Action:**
Create a hook `useSessionHistory()` that:
- Fetches `GET /api/sessions/history` on mount.
- Returns `{ sessions: ClosedSession[], loading: boolean, error: string | null }`.
- Uses `AbortController` cleanup on unmount (matches the pattern in `useRunbookSummary` and `useSessionState`).
- No polling — history is read-once per page open (AC 1 says "when supervisor opens history view").

Add a `ClosedSession` type to `packages/ui/src/types/runbook.ts` mirroring the server type (same fields, same shape). Comment it as "client-side mirror of server ClosedSession".

**Verification:** Unit test: mock `fetch` returning two sessions → hook returns them in `sessions`. Mock `fetch` returning non-2xx → hook sets `error`. AbortController is called on unmount.

---

### Step 6: Add `SessionHistoryPage` UI component

**Files:** `packages/ui/src/pages/SessionHistoryPage.tsx` (new file)

**Action:**
Create a `SessionHistoryPage` component that:
- Calls `useSessionHistory()`.
- Displays a heading "Session History".
- Shows a loading indicator while `loading` is true.
- Shows an error alert (matching the pattern in `Dashboard.tsx`) when `error` is set.
- Shows "No past sessions." when `sessions` is empty and not loading.
- Renders a list — one row per `ClosedSession` — displaying:
  - Date/time: formatted from `closedAt` as a locale date-time string.
  - Runbooks: count and first filename of each path (e.g. "2 runbooks: runbook-core-infrastructure.md, runbook-session-setup.md").
  - Status: a `StatusBadge` (existing component) with the session `status` value.
  - Duration: `durationSeconds` formatted as `Xm Ys` (e.g. "3m 42s").
- Each row has an `onClick` prop that calls `onSelectSession(session.sessionId)` — a prop passed in from `App.tsx`.
- A "Back" button calls `onBack` prop to return to the dashboard.

**Verification:** Renders the list correctly when sessions are provided. Each row is keyboard-accessible (button or role="button"). "No past sessions." appears when sessions array is empty.

---

### Step 7: Wire navigation in `App.tsx` and add "History" + "Close Session" buttons to `Dashboard`

**Files:**
- `packages/ui/src/App.tsx`
- `packages/ui/src/pages/Dashboard.tsx`

**Action:**

In `App.tsx`:
- Add a `view` state: `'dashboard' | 'history' | 'session-detail'` (string union). Default `'dashboard'`.
- Add a `selectedSessionId` state: `string | null`. Default `null`.
- Conditionally render `SessionHistoryPage` when `view === 'history'`, passing `onBack={() => setView('dashboard')}` and `onSelectSession={(id) => { setSelectedSessionId(id); setView('session-detail'); }}`.
- Keep rendering `Dashboard` when `view === 'dashboard'`.
- Render a placeholder `<div>` when `view === 'session-detail'` (ARC-1312 fills this in next story).
- Pass `onOpenHistory={() => setView('history')}` and `onCloseSession` to `Dashboard`.

In `Dashboard.tsx`:
- Accept two new optional props: `onOpenHistory?: () => void` and `onCloseSession?: () => void`.
- Add a "View History" button in the header area that calls `onOpenHistory` when clicked. Disabled and hidden if `onOpenHistory` is not provided.
- Add a "Close Session" button next to "Start Session" that is visible only when `sessionStartedAt` is truthy in the state response. Clicking it calls `POST /api/sessions/close` then calls `onCloseSession?.()`.
- `useSessionState` already fetches `GET /api/state` which includes `sessionStartedAt` — use the presence of that field to show/hide the Close button.

**Verification:** Clicking "View History" navigates to `SessionHistoryPage`. Clicking "Back" returns to Dashboard. Clicking "Close Session" posts to the close endpoint; on success the session list updates on next history open.

---

### Step 8: Add server-side unit tests

**Files:** `packages/server/src/__tests__/sessions.test.ts`

**Action:**
Extend the existing test file with:
- `POST /api/sessions/close` — returns 400 with no active session, returns 200 and correct `ClosedSession` shape with active session, calls `writeSidecar` with `closedSessions` appended and `sessionStartedAt` cleared.
- `GET /api/sessions/history` — returns `[]` when none, returns array when entries present, preserves order.

**Verification:** `npm test` in `packages/server` passes all tests including new ones.

---

### Step 9: Add UI-side unit tests

**Files:**
- `packages/ui/src/hooks/useSessionHistory.test.ts` (new file)
- `packages/ui/src/pages/SessionHistoryPage.test.tsx` (new file)

**Action:**
- `useSessionHistory.test.ts`: mock `fetch`; assert sessions returned on success; assert error set on non-2xx; assert abort on unmount.
- `SessionHistoryPage.test.tsx`: render with mocked hook; assert row count; assert "No past sessions." when empty; assert `onSelectSession` called on row click; assert `onBack` called on Back button click.

**Verification:** `npm test` in `packages/ui` passes all tests including new ones.

## Testing & Validation

**Automated (run from `packages/server` and `packages/ui` with `npm test`):**
- All existing tests continue to pass (no regressions).
- New `POST /api/sessions/close` tests: 400 path, 200 path, `writeSidecar` called correctly.
- New `GET /api/sessions/history` tests: empty and non-empty cases.
- `useSessionHistory` hook tests: success, error, abort.
- `SessionHistoryPage` render tests: list, empty state, navigation callbacks.

**Manual end-to-end scenario:**
1. Start server. Open dashboard. Select runbooks, start session.
2. Click "Close Session". Confirm 200 response and `closedSessions` appears in the sidecar file.
3. Click "View History". Confirm session row appears with correct date, runbooks, status, duration.
4. Click the row. Confirm navigation to `session-detail` view (placeholder for ARC-1312).
5. Click "Back". Confirm return to dashboard.
6. Restart server. Open history again. Confirm closed session persists across restart (read from sidecar).

## Risks & Open Questions

- **Session close trigger:** The issue does not define when a session closes (user action vs. automatic on completion). This plan uses an explicit "Close Session" button as the simplest unambiguous mechanism. If the supervisor requires automatic close on runbook completion, that is a scope addition requiring a new issue.
- **`stepState` snapshot size:** For repos with many runbook steps, embedding `stepState` in each `ClosedSession` entry grows the sidecar file. With the current scale (26 steps max), this is not a concern. If the repo grows significantly, a separate history file would be appropriate — but that is premature optimization here.
- **Status derivation at close time:** "complete" requires all `stepState` values to be `true`. Steps that were never run (value `false`) count against completion. This matches the natural interpretation: a partial session has uninitiated steps.
