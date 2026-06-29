# ARC-1311 — Browse past sessions list

## Completed: 2026-06-29

### Acceptance Criteria and Evidence

**AC 1:** Given past sessions exist, when supervisor opens history view, then all listed with date, runbooks, status, duration.

Evidence:
- `packages/server/src/routes/sessions.ts` — `GET /api/sessions/history` returns `closedSessions[]` with `closedAt`, `selectedRunbooks`, `status`, `durationSeconds` fields
- `packages/server/src/state/types.ts` — `ClosedSession` interface defines all required fields
- `packages/ui/src/hooks/useSessionHistory.ts` — fetches `GET /api/sessions/history` on mount; returns `{ sessions, loading, error }`
- `packages/ui/src/pages/SessionHistoryPage.tsx` — renders session list with date (`closedAt`), runbooks (count + filenames), `StatusBadge`, and formatted duration
- Test: `packages/server/src/__tests__/sessions.test.ts` — "returns closed sessions in order" passes
- Test: `packages/ui/src/hooks/useSessionHistory.test.ts` — "returns sessions on successful fetch" passes
- Test: `packages/ui/src/pages/SessionHistoryPage.test.tsx` — "renders a row for each session" passes

**AC 2:** Given past session entry, when supervisor selects it, then session detail view opens.

Evidence:
- `packages/ui/src/pages/SessionHistoryPage.tsx` — each session row is a `<button>` that calls `onSelectSession(session.sessionId)`
- `packages/ui/src/App.tsx` — `onSelectSession` handler sets `selectedSessionId` and transitions `view` to `'session-detail'`, rendering `SessionDetailPage`
- Test: `packages/ui/src/pages/SessionHistoryPage.test.tsx` — "calls onSelectSession with session id when row is clicked" passes

### Files Changed

| File | Change |
|------|--------|
| `packages/server/src/state/types.ts` | Added `StoredEvent`, `StepRecord`, `ClosedSession` interfaces; added `closedSessions?` and `stepLog?` to `SidecarState` |
| `packages/server/src/routes/sessions.ts` | Added `POST /api/sessions/close`, `GET /api/sessions/history`, `GET /api/sessions/history/:sessionId` |
| `packages/ui/src/types/runbook.ts` | Added `StoredEvent`, `StepRecord`, `ClosedSession` client-side mirror types |
| `packages/ui/src/hooks/useSessionHistory.ts` | New file — hook for fetching session history list |
| `packages/ui/src/pages/SessionHistoryPage.tsx` | New file — session list UI page |
| `packages/ui/src/App.tsx` | Added view state, SessionHistoryPage/SessionDetailPage routing |
| `packages/ui/src/pages/Dashboard.tsx` | Added `onOpenHistory` and `onCloseSession` props, "View History" and "Close Session" buttons |
| `packages/server/src/__tests__/sessions.test.ts` | Extended with close and history endpoint tests |
| `packages/ui/src/hooks/useSessionHistory.test.ts` | New file — hook tests |
| `packages/ui/src/pages/SessionHistoryPage.test.tsx` | New file — page render tests |

### Test Results

```
Server: 135 tests passed (13 test files)
UI: 89 tests passed (12 test files)
TypeScript: no errors (both packages)
```

### No deviations from plan.

