# ARC-1310 Completion Summary — Persist Active Session State Across Restarts

**Story:** ARC-1310
**Epic:** ARC-1309 — Session History
**Branch:** `ARC-1310` in `/repos/pa.aid.wsl-setup.sh`
**Plan:** `implementation_plans/session-history/ARC-1310-implementation-plan.md`
**Completed:** 2026-06-29

---

## Acceptance Criteria Evidence

### AC 1: Given active session, when tool restarts, then session restored — runbooks, progress, checkpoints, config all present

**Status:** ✅ Satisfied

**Evidence:**

1. **Pre-existing restart recovery:** `packages/server/src/state/reconcile.ts` — `loadOrCreateState` reads the sidecar at startup and reconciles it against the current markdown. All fields (`selectedRunbooks`, `stepState`, `checkpointQueue`, `config`, `sessionStartedAt`) are present in `SidecarState` and restored.

2. **Write-back gap fixed:** `packages/server/src/routes/runbooks.ts` — `GET /api/runbooks/summary` now calls `writeSidecar(reconciled.repoRoot, reconciled)` after reconciling state, ensuring any mid-session reconcile is persisted to disk before a possible restart.

3. **Test coverage:** `packages/server/src/__tests__/runbooks.summary.test.ts` — test `"ARC-1310: persists reconciled state to sidecar file after summary request"` seeds a stale sidecar on disk, makes a summary request, then reads the sidecar back and asserts the reconciled (markdown-authoritative) stepState was written.

   Test output: 120 tests pass (`npm test` with `PORT=4099`).

---

### AC 2: Given two simultaneous starts on same repo, when second starts, then it detects active session and warns supervisor

**Status:** ✅ Satisfied

**Evidence:**

1. **Lock file helpers:** `packages/server/src/state/sidecar.ts` — three new exported functions:
   - `getLockPath(repoRoot)` — returns path to `.runbook-executor-lock`
   - `acquireLock(repoRoot)` — writes PID; returns `true` if no prior lock, `false` if conflict
   - `releaseLock(repoRoot)` — deletes lock file; ignores `ENOENT`

2. **Startup enforcement:** `packages/server/src/index.ts` — calls `acquireLock(repoRoot)` at startup. If conflict detected, emits console warning: `[ARC-1310] WARNING: Another runbook-executor instance may already be running…`. Sets `conflictingSession: true` in initial state.

3. **Clean shutdown:** `packages/server/src/index.ts` — registers `process.once('SIGINT')` and `process.once('SIGTERM')` handlers that call `releaseLock(repoRoot)` and exit cleanly.

4. **API flag:** `packages/server/src/state/types.ts` — `conflictingSession?: boolean` added to `SidecarState`. `packages/server/src/routes/state.ts` — `conflictingSession` added to `READ_ONLY_FIELDS` (cannot be PATCHed).

5. **Lock file in `.gitignore`:** `runbook-executor/.gitignore` — `.runbook-executor-lock` pattern added.

6. **Test coverage:** 
   - `packages/server/src/__tests__/sidecar.test.ts` — 8 new tests cover `getLockPath`, `acquireLock` (first call returns true, second returns false, PID written, idempotent), `releaseLock` (deletes file, idempotent, re-acquirable after release).
   - `packages/server/src/__tests__/state.test.ts` (new file) — 4 tests cover `GET /api/state` returns `conflictingSession: true` when set; `PATCH /api/state` with `conflictingSession` returns 400.

   Test output: 120 tests pass (`npm test` with `PORT=4099`).

---

## Commit Log (feature branch `ARC-1310`)

| Commit | Description |
|--------|-------------|
| `ec9c555` | ARC-1310: persist reconciled state after GET /api/runbooks/summary |
| `02b3b8f` | ARC-1310: add conflictingSession flag to SidecarState |
| `da32cf5` | ARC-1310: add getLockPath/acquireLock/releaseLock helpers to sidecar.ts |
| `efac0a6` | ARC-1310: acquire/release lock file at startup/shutdown, set conflictingSession flag |
| `64f7873` | ARC-1310: add conflictingSession to READ_ONLY_FIELDS in state route |
| `47537fc` | ARC-1310: add tests for lock helpers, write-back, conflictingSession enforcement |

---

## Test Results

```
Test Files  13 passed (13)
Tests       120 passed (120)
```

TypeScript type-check: `npx tsc --noEmit` — no errors.

---

## Files Changed

| File | Change |
|------|--------|
| `packages/server/src/routes/runbooks.ts` | Added `writeSidecar` call after reconcile in summary handler |
| `packages/server/src/state/types.ts` | Added `conflictingSession?: boolean` to `SidecarState` |
| `packages/server/src/state/sidecar.ts` | Added `getLockPath`, `acquireLock`, `releaseLock` |
| `packages/server/src/index.ts` | Lock acquire/release at startup/shutdown; `conflictingSession` flag |
| `packages/server/src/routes/state.ts` | `conflictingSession` in `READ_ONLY_FIELDS` |
| `runbook-executor/.gitignore` | Added `.runbook-executor-lock` pattern |
| `packages/server/src/__tests__/sidecar.test.ts` | Lock helper tests |
| `packages/server/src/__tests__/runbooks.summary.test.ts` | Write-back persistence test |
| `packages/server/src/__tests__/state.test.ts` | New: `conflictingSession` API tests |
```