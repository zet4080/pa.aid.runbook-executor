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

- [ ] **ARC-1313** — Display runbook dashboard at startup
  - [ ] 🔒 Claimed: _(fill in: agent / YYYY-MM-DD HH:MM before starting)_
  - [ ] Read `issues/session-setup/ARC-1313-display-runbook-dashboard.md`
  - [ ] Write `implementation_plans/session-setup/ARC-1313-implementation-plan.md`
  - [ ] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [ ] Execute plan
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1313-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(session-setup): add runbook dashboard at startup`

### MEDIUM batch — Expand, Select, Configure

- [ ] 🔒 Claimed: _(fill in: agent / YYYY-MM-DD HH:MM before starting)_
- [ ] Read all issues: ARC-1314, ARC-1315, ARC-1316
- [ ] Write all implementation plans
- [ ] 🟡 BATCH PLAN CHECKPOINT
- [ ] Execute ARC-1314
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1314-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(session-setup): expand runbook detail on demand`
- [ ] Execute ARC-1315
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1315-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(session-setup): select runbooks to include in session`
- [ ] Execute ARC-1316
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1316-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(session-setup): configure session parameters before start`
- [ ] ☑ batch complete

### Wave 1 gate
- [ ] All Wave 1 stories checked off
- [ ] All PRs merged
- [ ] All completion summaries written
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
