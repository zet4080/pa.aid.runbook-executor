# ARC-1316 Configure Session Parameters Before Start — Implementation Plan

**Issue:** `issues/session-setup/ARC-1316-configure-session-parameters.md`
**Completion Summary:** `task-completions/ARC-1316-COMPLETION-SUMMARY.md` (TBD)
**Approach:** Config panel in Dashboard + PATCH /api/state for config persistence + Start Session button
**Owner:** Build agent
**Date:** 2026-06-29

---

## Scope & Alignment

This plan adds a session configuration panel on the Dashboard where the supervisor sets concurrency, retry count, escalation sequence, and queue policy before starting. Defaults are pre-populated from `DEFAULT_SESSION_CONFIG`. Configuration is persisted via `PATCH /api/state`. A "Start Session" button initiates the session only after config is confirmed.

| AC | Step(s) that satisfy it |
|----|-----------------------|
| AC-1: Concurrency set to N → at most N lanes run simultaneously | Steps 1, 2 (persisted to `config.maxConcurrentLanes`; enforced by ARC-1292 executor) |
| AC-2: Defaults pre-populated → session runs with sensible defaults without changes | Steps 1, 2 (`DEFAULT_SESSION_CONFIG` pre-fills the form) |
| AC-3: Escalation sequence configured → each retry uses next agent in sequence | Steps 1, 2 (persisted to `config` field; consumed by ARC-1307 retry escalation) |

**Note on AC-1 and AC-3:** This story persists the config values. Actual enforcement of concurrency (AC-1) and escalation (AC-3) is implemented in ARC-1292 and ARC-1307 respectively. This story's acceptance is satisfied by ensuring the correct values are written to state and are readable by those stories.

---

## Assumptions & Dependencies

- ARC-1313 is complete. `Dashboard` and `useSessionState` (added in ARC-1315) exist.
- `SidecarState.config: SessionConfig` exists with `maxConcurrentLanes`, `retryLimit`, `queuePolicy` (see `packages/server/src/state/types.ts`).
- `DEFAULT_SESSION_CONFIG` is exported from `packages/server/src/state/types.ts`: `{ maxConcurrentLanes: 3, retryLimit: 3, queuePolicy: 'checkpoint-first' }`.
- `PATCH /api/state` already accepts `{ config: Partial<SessionConfig> }` and merges it shallowly.
- The issue mentions "escalation sequence" as a configurable parameter. In the current `SidecarState`, there is no `escalationSequence` field — only `retryLimit`. The escalation sequence (junior → senior → human) is defined in ARC-1307. For this story, the configurable value is `retryLimit` (number of retries before escalation). If a dedicated `escalationSequence` field is needed, it must be added to `SessionConfig` and `SidecarState` as part of this story.
- The worktree for this batch is `/repos/ARC-1314`.
- All file paths below are relative to the repo root of `pa.aid.conductor.ts`.

---

## Implementation Steps

### Step 1: Extend `SessionConfig` if needed for escalation sequence

**Files:**
- `packages/server/src/state/types.ts` (modify if `escalationSequence` field is required)
- `packages/server/src/__tests__/sidecar.test.ts` (verify — no change expected if field added with default)

**Action:**

Review the issue's AC-3: "given escalation sequence configured, when step fails and retries, then each retry uses next agent in sequence." The existing `retryLimit` covers retry count. The escalation sequence (which agent tier to use on each retry) is a separate concern.

Add `escalationSequence: string[]` to `SessionConfig` with a default of `['junior', 'senior-coder', 'human']`. Update `DEFAULT_SESSION_CONFIG` accordingly.

If this field already exists (check `types.ts` before implementing), skip this step.

**Verification:** `npm run test --workspace=packages/server` passes — no existing tests broken by the additive field. `sidecar.test.ts` confirms round-trip serialisation includes the new field.

---

### Step 2: Add client-side `SessionConfig` type mirror and config form

**Files:**
- `packages/ui/src/types/runbook.ts` (modify — add `SessionConfig`, `QueuePolicy` mirror types)
- `packages/ui/src/components/SessionConfigPanel.tsx` (new)
- `packages/ui/src/components/SessionConfigPanel.test.tsx` (new)

**Action:**

**`src/types/runbook.ts`** — append (additive):
- `QueuePolicy: 'checkpoint-first' | 'lane-first' | 'balanced' | 'wave-gate-first'`
- `SessionConfig: { maxConcurrentLanes: number; retryLimit: number; queuePolicy: QueuePolicy; escalationSequence: string[] }`

**`SessionConfigPanel.tsx`** — new component.

Props:
```
interface SessionConfigPanelProps {
  config: SessionConfig;
  onChange: (updated: Partial<SessionConfig>) => void;
}
```

Fields rendered:
- **Max concurrent lanes**: number input, min=1, max=10, step=1. Label: "Max concurrent lanes".
- **Retry limit**: number input, min=0, max=10, step=1. Label: "Retry limit per step".
- **Queue policy**: `<select>` with options: `checkpoint-first`, `lane-first`, `balanced`, `wave-gate-first`. Label: "Queue policy".
- **Escalation sequence**: read-only display of `config.escalationSequence.join(' → ')` (e.g. "junior → senior-coder → human"). Editing the sequence is out of scope for this story — display only, with a note "Sequence is fixed for MVP".

Each field calls `onChange({ fieldName: newValue })` on change. The parent (`Dashboard`) calls `PATCH /api/state` with `{ config: updatedPartial }` via `useSessionState`.

**Verification:** `SessionConfigPanel.test.tsx`: renders all four fields with correct default values; changing the concurrency input calls `onChange` with `{ maxConcurrentLanes: newValue }`; changing queue policy calls `onChange` with `{ queuePolicy: newValue }`.

---

### Step 3: Integrate config panel and Start Session button into Dashboard

**Files:**
- `packages/ui/src/pages/Dashboard.tsx` (modify)
- `packages/ui/src/hooks/useSessionState.ts` (modify — add `updateConfig` function)
- `packages/ui/src/pages/Dashboard.test.tsx` (modify — add config panel and start button tests)

**Action:**

**`useSessionState.ts`** additions:
- Read `config` from the initial `GET /api/state` response.
- Expose `config: SessionConfig` from the hook return value.
- Add `updateConfig(partial: Partial<SessionConfig>): void` — calls `PATCH /api/state` with `{ config: partial }`, optimistic update, revert on error.
- Add `startSession(): void` — calls `POST /api/sessions/start` (see note below on new endpoint).

**New endpoint `POST /api/sessions/start`:**
For this MVP, the "Start Session" action sets a flag in state indicating the session is active. Add a new route handler in a new file `packages/server/src/routes/sessions.ts`:
- `POST /api/sessions/start`: reads current `selectedRunbooks` from state; if empty, return `400 { error: "No runbooks selected" }`; otherwise set `state.sessionStartedAt = new Date().toISOString()` (requires adding `sessionStartedAt?: string` to `SidecarState`) and persist. Return `200 { ok: true }`.
- Register the router in `packages/server/src/index.ts` under `/api`.

**`Dashboard.tsx`** additions:
1. Render `<SessionConfigPanel config={config} onChange={updateConfig} />` in a collapsible "Session Configuration" section below the runbook grid.
2. Add a "Start Session" button at the bottom of the page. On click, call `startSession()`. Disable the button if `selectedRunbooks.length === 0`.
3. Show "No runbooks selected — select at least one to start." helper text when `selectedRunbooks.length === 0`.

**Verification:**
- `Dashboard.test.tsx`: mock `useSessionState` returning a config; confirm `SessionConfigPanel` receives the correct props; confirm Start Session button is disabled when `selectedRunbooks` is empty; confirm it is enabled when `selectedRunbooks.length > 0`.
- Integration: `PATCH /api/state` with `{ config: { maxConcurrentLanes: 5 } }` returns updated state with `config.maxConcurrentLanes === 5`.

---

## Testing & Validation

| Layer | Command | What it covers |
|-------|---------|----------------|
| Server unit | `npm run test --workspace=packages/server` | New `POST /api/sessions/start` endpoint; `SessionConfig` sidecar round-trip |
| UI unit | `npm run test --workspace=packages/ui` | `SessionConfigPanel` rendering and onChange callbacks; Dashboard config integration and Start button state |
| Full suite | `npm test` (root) | All of the above |

Manual verification:
1. Start server: `REPO_ROOT=/repos/pa.aid.runbook-executor npm run dev`
2. Navigate to `http://localhost:5173`; confirm session config panel shows defaults (3 concurrent, 3 retries, checkpoint-first).
3. Change concurrency to 2; call `GET /api/state` — confirm `config.maxConcurrentLanes === 2`.
4. Without selecting any runbook, confirm Start Session is disabled.
5. Select 2 runbooks; confirm Start Session is enabled.
6. Click Start Session; confirm `POST /api/sessions/start` returns 200.
7. Refresh; confirm config changes persisted.

---

## Risks & Open Questions

| # | Risk | Likelihood | Mitigation |
|---|------|-----------|-----------|
| 1 | `escalationSequence` not in current `SidecarState` — requires a schema addition | Medium | Add field with default in Step 1; no migration needed since sidecar is local and reconcile seeds from markdown on restart |
| 2 | `POST /api/sessions/start` is a placeholder for the full lane-execution start (ARC-1291) | Medium | MVP start just sets a flag. ARC-1291 will hook into this endpoint to trigger real execution. Document the contract clearly in the route handler. |
| 3 | `sessionStartedAt` field added to `SidecarState` — bumping schema version | Low | Additive optional field; no version bump required (version bump only for breaking changes per the type comment) |
