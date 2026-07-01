# ARC-1310 Persist Active Session State Across Restarts — Implementation Plan
**Issue:** `issues/session-history/ARC-1310-persist-session-state.md`
**Completion Summary:** `task-completions/ARC-1310-COMPLETION-SUMMARY.md` (TBD)
**Approach:** File-lock sentinel for single-session enforcement; sidecar write-back gap fix for summary reconcile
**Owner:** agent
**Date:** 2026-06-29

## Scope & Alignment

This plan covers two acceptance criteria in ARC-1310:

- **AC 1:** On tool restart, the active session is fully restored from sidecar — runbooks, progress (stepState), checkpoint queue, config, and `sessionStartedAt` are all present.
- **AC 2:** When a second server instance starts against the same repo, it detects the running instance and warns the supervisor.

The sidecar primitives (`writeSidecar`, `readSidecar`, `loadOrCreateState`) and the `SidecarState` type already exist (ARC-1287). The restart-recovery path (`loadOrCreateState` in `index.ts`) already reads and reconciles sidecar state. What is missing:

1. **Write-back gap:** `GET /api/runbooks/summary` calls `reconcileState` and `setState` but does not call `writeSidecar`, so a reconcile triggered mid-session is not persisted. If the server restarts before the next explicit mutation (PATCH /api/state, POST /api/sessions/start), the last reconciled state is lost.
2. **Single-session enforcement (AC 2):** No lock mechanism exists. Two simultaneous server instances pointing at the same repo can start without any detection or warning.

## Assumptions & Dependencies

- ARC-1287 is merged; `sidecar.ts`, `store.ts`, `reconcile.ts`, and `types.ts` are in place at `packages/server/src/state/`.
- The `writeSidecar` atomic write-then-rename is already crash-safe for this single-writer scenario (ARC-1294 will later serialize concurrent lane writes).
- The lock file lives alongside the sidecar in the repo root. Its presence at startup is the detection signal.
- The server process is expected to release the lock on clean shutdown (`SIGINT`/`SIGTERM`). Stale locks (crash without cleanup) are a known limitation; the supervisor warning message will advise manual deletion.
- "Warn supervisor" for AC 2 means: log a visible warning to the server console AND expose a flag in `GET /api/state` that the UI can surface.

## Implementation Steps

### Step 1: Fix write-back gap in `GET /api/runbooks/summary`

**Files:** `packages/server/src/routes/runbooks.ts`

**Action:** After calling `reconcileState` and `setState` in the `GET /api/runbooks/summary` handler, also call `writeSidecar(reconciled.repoRoot, reconciled)` to persist the reconciled state. Import `writeSidecar` from `../state/sidecar.js`.

**Verification:** Unit test: after a `GET /api/runbooks/summary` call, the sidecar file on disk reflects the reconciled stepState (not the stale pre-reconcile value).

---

### Step 2: Add `conflictingSession` flag to `SidecarState`

**Files:** `packages/server/src/state/types.ts`

**Action:** Add an optional field `conflictingSession?: boolean` to the `SidecarState` interface. This flag is set to `true` at startup when a lock file is detected. The UI reads this via `GET /api/state` and can surface a warning banner.

**Verification:** TypeScript compiles without error. Existing tests that construct `SidecarState` without the field continue to pass (field is optional).

---

### Step 3: Implement lock file helpers in `sidecar.ts`

**Files:** `packages/server/src/state/sidecar.ts`

**Action:** Add three exported functions:

- `getLockPath(repoRoot: string): string` — returns `path.join(repoRoot, '.runbook-executor-lock')`.
- `acquireLock(repoRoot: string): boolean` — writes current PID to lock file using `writeFileSync`. Returns `true` if the file did not already exist before the write, `false` if it did (conflict detected). Does not throw; warns to console on any write error and returns `false`.
- `releaseLock(repoRoot: string): void` — deletes the lock file with `unlinkSync`. Ignores `ENOENT`. Warns to console on other errors.

Lock file content: `String(process.pid)` — aids human diagnosis of stale locks.

**Verification:** Unit tests cover: `getLockPath` returns correct path; `acquireLock` returns `true` on first call (no prior lock file) and returns `false` when lock file already exists; `releaseLock` removes the file and is idempotent (no error when file absent).

---

### Step 4: Register lock acquisition in `index.ts` startup

**Files:** `packages/server/src/index.ts`

**Action:**
1. After computing `repoRoot`, call `acquireLock(repoRoot)`.
2. If `acquireLock` returns `false` (conflict): log a clearly visible warning — `[ARC-1310] WARNING: Another runbook-executor instance may already be running against this repo. Lock file exists at <lockPath>. If no other instance is running, delete it manually.` — and set `conflictingSession: true` on the initial state before calling `setState`.
3. Register a shutdown handler that calls `releaseLock(repoRoot)` on `SIGINT` and `SIGTERM`. Use `process.once` for each signal. After releasing the lock, call `process.exit(0)`.

**Verification:** Manual test scenario — start server, verify lock file exists; stop server (Ctrl-C), verify lock file is deleted. Automated: the lock integration test (Step 6) covers acquire/release within a single process; signal handler is tested manually only.

---

### Step 5: Expose `conflictingSession` in `GET /api/state`

**Files:** `packages/server/src/routes/state.ts`

**Action:** Add `conflictingSession` to the `READ_ONLY_FIELDS` array so callers cannot PATCH it. The existing `GET /api/state` handler already calls `getState()` and returns it — no body change needed. The field will appear in the JSON response whenever `conflictingSession: true` is set.

**Verification:** Unit test: construct state with `conflictingSession: true`, verify `GET /api/state` returns it; verify `PATCH /api/state` with `{ conflictingSession: true }` returns 400.

---

### Step 6: Add/extend tests for new behaviour

**Files:**
- `packages/server/src/__tests__/sidecar.test.ts` — add `getLockPath`, `acquireLock`, `releaseLock` test suite
- `packages/server/src/__tests__/reconcile.test.ts` — add test: `loadOrCreateState` calls `writeSidecar` after reconcile (spy on `writeSidecar`)
- `packages/server/src/__tests__/sessions.test.ts` or a new `runbooks.summary.test.ts` companion — add test: `GET /api/runbooks/summary` calls `writeSidecar` with the reconciled state
- `packages/server/src/__tests__/state.test.ts` (new) — test `conflictingSession` in `READ_ONLY_FIELDS` and `GET /api/state` response

**Verification:** `npm test` in `packages/server` passes all tests with no failures.

---

## Testing & Validation

**Unit test suite (`npm test` in `packages/server`):**
- All existing tests continue to pass (no regressions).
- New `acquireLock`/`releaseLock` tests in `sidecar.test.ts` pass.
- `GET /api/runbooks/summary` write-back test passes.
- `conflictingSession` read-only enforcement test passes.

**Manual restart recovery (AC 1):**
1. Start server, select runbooks, start session.
2. Kill and restart server.
3. `GET /api/state` returns `sessionStartedAt`, `selectedRunbooks`, `stepState`, and `config` matching pre-restart values.

**Manual conflict detection (AC 2):**
1. Start server (process A). Verify lock file exists at `<repoRoot>/.runbook-executor-lock`.
2. Start a second server instance (process B) against the same `repoRoot`.
3. Process B console log shows `[ARC-1310] WARNING: Another runbook-executor instance…`.
4. `GET /api/state` on process B returns `conflictingSession: true`.
5. Stop process A. Verify lock file is deleted.

## Risks & Open Questions

- **Stale lock on crash:** If the server crashes without running the SIGINT/SIGTERM handler (e.g. SIGKILL), the lock file persists. The warning message advises manual deletion. This is an accepted known limitation — no auto-stale-detection in scope for ARC-1310.
- **Lock file in `.gitignore`:** `.runbook-executor-lock` should be added to the root `.gitignore` of `pa.aid.conductor.ts` to prevent accidental commits. Verify the existing `.gitignore` or add the entry.
- **`conflictingSession` in UI:** This plan adds the server-side flag only. A UI warning banner consuming `conflictingSession` is out of scope for ARC-1310 (no UI AC in the issue). The field is available in `GET /api/state` for any future UI story to consume.
