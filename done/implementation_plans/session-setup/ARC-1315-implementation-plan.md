# ARC-1315 Select Runbooks to Include in Session — Implementation Plan

**Issue:** `issues/session-setup/ARC-1315-select-runbooks-for-session.md`
**Completion Summary:** `task-completions/ARC-1315-COMPLETION-SUMMARY.md` (TBD)
**Approach:** Checkbox multi-select on RunbookCard + PATCH /api/state to persist selectedRunbooks
**Owner:** Build agent
**Date:** 2026-06-29

---

## Scope & Alignment

This plan adds runbook selection to the dashboard: supervisors tick checkboxes on runbook cards to include them in the current session. Selection is persisted immediately to the sidecar via the existing `PATCH /api/state` endpoint. Unselected runbooks remain visible but are excluded when the session starts.

| AC | Step(s) that satisfy it |
|----|-----------------------|
| AC-1: Selected subset → only selected runbooks execute when session starts | Steps 1, 2, 3 |
| AC-2: Deselected runbook stays idle when session starts | Steps 2, 3 (deselected path absent from `selectedRunbooks` in state) |

**Out of scope:** adding/removing runbooks from the repo, mid-session selection changes.

---

## Assumptions & Dependencies

- ARC-1313 is complete. `RunbookCard` summary view and `useRunbookSummary` hook exist.
- `SidecarState.selectedRunbooks: string[]` already exists in `packages/server/src/state/types.ts` and is patchable via `PATCH /api/state` (field is not in `READ_ONLY_FIELDS` in `packages/server/src/routes/state.ts`).
- `selectedRunbooks` holds absolute runbook file paths. `RunbookSummary.path` is the absolute path — same key.
- The existing `PATCH /api/state` endpoint in `packages/server/src/routes/state.ts` already accepts and persists `selectedRunbooks`. No server changes are required for basic selection persistence.
- The UI must initialise selection state from `GET /api/state` (or from `useRunbookSummary` polling) so that a page refresh retains the previous selection.
- The worktree for this batch is `/repos/ARC-1314` (shared worktree for the batch; all three stories execute there).
- All file paths below are relative to `runbook-executor/`.

---

## Implementation Steps

### Step 1: Add `useSessionState` hook (or extend existing state access)

**Files:**
- `packages/ui/src/hooks/useSessionState.ts` (new)
- `packages/ui/src/hooks/useSessionState.test.ts` (new)

**Action:**

Create a hook:
```
useSessionState(): { selectedRunbooks: string[]; toggleRunbook: (path: string) => void; loading: boolean; error: string | null }
```

Behaviour:
- On mount, fetch `GET /api/state` and extract `selectedRunbooks`.
- `toggleRunbook(path)`: add or remove `path` from the local set, then immediately call `PATCH /api/state` with `{ selectedRunbooks: updatedPaths }`.
- Optimistic update: update local state before the PATCH resolves; revert on error.
- `loading` is `true` during the initial GET only.
- Clean up with `AbortController` on unmount.

**Verification:** Hook test: initial fetch returns `selectedRunbooks: ["/a.md"]` → hook returns correct value; `toggleRunbook("/b.md")` adds it; `toggleRunbook("/a.md")` removes it; PATCH is called with correct body.

---

### Step 2: Add selection checkbox to `RunbookCard`

**Files:**
- `packages/ui/src/components/RunbookCard.tsx` (modify)
- `packages/ui/src/components/RunbookCard.test.tsx` (modify — add selection tests)

**Action:**

Add a props extension to `RunbookCard`:
```
interface RunbookCardProps {
  summary: RunbookSummary;
  selected: boolean;
  onToggleSelect: (path: string) => void;
}
```

In `CardHeader`, add a checkbox element with:
- `checked={selected}`
- `onChange={() => onToggleSelect(summary.path)}`
- `aria-label={`Select ${summary.metadata.title}`}`

The checkbox is placed left of the title (or in a row before the header). Visual convention: checked = full border/ring (selected); unchecked = subdued.

**Verification:** `RunbookCard.test.tsx`: checkbox renders; clicking it calls `onToggleSelect` with the card's path; checked prop reflects the `selected` value.

---

### Step 3: Wire selection into `Dashboard`

**Files:**
- `packages/ui/src/pages/Dashboard.tsx` (modify)
- `packages/ui/src/pages/Dashboard.test.tsx` (modify — add selection tests)

**Action:**

In `Dashboard.tsx`:
1. Call `useSessionState()` to get `selectedRunbooks`, `toggleRunbook`, `loading` (selection loading), and `error` (selection error).
2. Pass `selected={selectedRunbooks.includes(summary.path)}` and `onToggleSelect={toggleRunbook}` to each `<RunbookCard>`.
3. Show a selection summary row above the grid: "N of M selected" where N = `selectedRunbooks.length` and M = `summaries.length`.
4. When `useSessionState` loading is true and no selection is loaded yet, show a brief "Loading selection…" (distinct from the runbook-list loading state — they are independent).

**Verification:** `Dashboard.test.tsx`: mock `useSessionState` returning 1 of 2 selected; confirm "1 of 2 selected" text renders; confirm selected card receives `selected=true` and the other `selected=false`.

---

## Testing & Validation

| Layer | Command | What it covers |
|-------|---------|----------------|
| UI unit | `npm run test --workspace=packages/ui` | `useSessionState` hook, `RunbookCard` selection checkbox, `Dashboard` selection wiring |
| Full suite | `npm test` (root) | All of the above |

Manual verification:
1. Start server: `REPO_ROOT=/repos/pa.aid.runbook-executor npm run dev`
2. Navigate to `http://localhost:5173`; confirm all runbook cards show checkboxes.
3. Tick 3 runbooks; confirm "3 of 7 selected" appears.
4. Refresh the page; confirm the 3 selected runbooks are still ticked (persisted via sidecar).
5. Call `GET /api/state` and confirm `selectedRunbooks` contains exactly the 3 selected paths.
6. Untick one; confirm it disappears from `/api/state` on next GET.

---

## Risks & Open Questions

| # | Risk | Likelihood | Mitigation |
|---|------|-----------|-----------|
| 1 | RunbookCard prop interface change (adding `selected` / `onToggleSelect`) may break ARC-1314 tests | Low | ARC-1314 and ARC-1315 execute in the same worktree; update RunbookCard tests in ARC-1315 step to pass the new props |
| 2 | Optimistic update on PATCH failure could cause flicker | Low | On PATCH error, revert local state and show a toast/error message; acceptable for MVP |
| 3 | `selectedRunbooks` initialises empty on fresh sidecar | Low | Empty = all runbooks selectable; user must explicitly select before starting; acceptable default |
