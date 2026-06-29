# Runbook: Session Setup & Runbook Selection — ARC-1289

> Load the `execute-implementation-plan` skill before starting.
> Stop at every checkpoint symbol (🔴 🟡 🟢).
> Resume: find first unchecked box and continue.

**Feature goal:** Users can select and configure runbooks for a new session from a dashboard that loads on startup.
**Epic:** ARC-1289
**Lane:** session-setup
**Repos:** pa.aid.runbook-executor
**Skills to load at start:** execute-implementation-plan, write-implementation-plan
**Depends on:** ARC-1286 (core-infrastructure — runbook parser)

## Checkpoint model

| Symbol | Type | When |
|--------|------|------|
| 🔴 | Individual plan checkpoint | Before executing any HIGH story |
| 🟡 | Batch plan checkpoint | Before executing a batch of MEDIUM/LOW stories |
| 🟢 | Wave gate | After all stories in a wave close |

---

## Wave 1 — Session Setup & Runbook Selection

- [ ] Confirm core-infrastructure ARC-1286 merged

### HIGH stories

- [x] **ARC-1313** — Display runbook dashboard at startup
  - [x] 🔒 Claimed: build / 2026-06-28
  - [x] Read `issues/session-setup/ARC-1313-display-runbook-dashboard.md`
  - [x] Write `implementation_plans/session-setup/ARC-1313-implementation-plan.md`
  - [x] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [x] Execute plan
  - [x] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1313-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(session-setup): add runbook dashboard at startup`
  - [x] ✅ Verified: 2026-06-28

### MEDIUM batch — Expand, Select, Configure

- [x] 🔒 Claimed: session-setup / 2026-06-29 13:13
- [x] Read all issues: ARC-1314, ARC-1315, ARC-1316
- [x] Write all implementation plans
- [x] 🟡 BATCH PLAN CHECKPOINT
- [x] Execute ARC-1314
  - [x] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1314-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(session-setup): expand runbook detail on demand`
- [x] Execute ARC-1315
  - [x] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1315-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(session-setup): select runbooks to include in session`
- [x] Execute ARC-1316
  - [x] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1316-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(session-setup): configure session parameters before start`
- [x] ☑ batch complete

### Wave 1 gate
- [x] All Wave 1 stories checked off
- [ ] All PRs merged
- [x] All completion summaries written
- [ ] 🟢 WAVE GATE — notify dependent lanes: none

---

## On-Hold Register

| Story | Blocked by | Last checked | Action if unblocked |
|-------|-----------|--------------|---------------------|
| ARC-1313 | ARC-1286 (core-infrastructure) | 2026-06-27 | Confirm ARC-1286 merged; begin immediately |

---

## Checkpoint Summary

| Wave | HIGH checkpoints | Batch checkpoints | Wave gates | Total |
|------|-----------------|-------------------|------------|-------|
| Wave 1 | 1 | 1 | 1 | 3 |
| **Total** | **1** | **1** | **1** | **3** |
