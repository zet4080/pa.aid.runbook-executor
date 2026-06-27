# Runbook: Session History — ARC-1309

> Load the `execute-implementation-plan` skill before starting.
> Stop at every checkpoint symbol (🔴 🟡 🟢).
> Resume: find first unchecked box and continue.

**Feature goal:** Supervisors can view, restore, and browse the full history of past runbook execution sessions without losing any progress across restarts.
**Epic:** ARC-1309
**Lane:** session-history
**Repos:** pa.aid.runbook-executor
**Skills to load at start:** execute-implementation-plan, write-implementation-plan
**Depends on:** ARC-1287 (core-infrastructure — sidecar state file)

## Checkpoint model

| Symbol | Type | When |
|--------|------|------|
| 🔴 | Individual plan checkpoint | Before executing any HIGH story |
| 🟡 | Batch plan checkpoint | Before executing a batch of MEDIUM/LOW stories |
| 🟢 | Wave gate | After all stories in a wave close |

---

## Pre-check

- [ ] Confirm core-infrastructure ARC-1287 merged

---

## Wave 1 — Session Persistence Foundation

### ARC-1310 — Persist active session state across restarts

- [ ] **ARC-1310** — Persist active session state across restarts
  - [ ] 🔒 Claimed: _(fill in: agent / YYYY-MM-DD HH:MM before starting)_
  - [ ] Read `issues/session-history/ARC-1310-persist-session-state.md`
  - [ ] Write `implementation_plans/session-history/ARC-1310-implementation-plan.md`
  - [ ] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [ ] Execute plan
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1310-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(session-history): persist active session state across restarts`

🟢 WAVE 1 GATE — ARC-1310 closed and merged before Wave 2 starts

---

## Wave 2 — Session Browse & Step History

- [ ] 🔒 Claimed: _(fill in: agent / YYYY-MM-DD HH:MM before starting)_
- [ ] Read all issues: ARC-1311, ARC-1312
- [ ] Write all implementation plans
- [ ] 🟡 BATCH PLAN CHECKPOINT
- [ ] Execute ARC-1311
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1311-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(session-history): browse past sessions list`
- [ ] Execute ARC-1312
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1312-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(session-history): view full step output history for any past session`
- [ ] ☑ batch complete

🟢 WAVE 2 GATE — ARC-1311 and ARC-1312 closed and merged

---

## On-Hold Register

| Story | Blocked by | Last checked | Action if unblocked |
|-------|-----------|--------------|---------------------|
| ARC-1310 | ARC-1287 (sidecar state file) | 2026-06-27 | Confirm ARC-1287 merged; begin Wave 1 |

---

## Checkpoint Summary

| Wave | HIGH checkpoints | Batch checkpoints | Wave gates | Total |
|------|-----------------|-------------------|------------|-------|
| Wave 1 | 1 | 0 | 1 | 2 |
| Wave 2 | 0 | 1 | 1 | 2 |
| **Total** | **1** | **1** | **2** | **4** |
