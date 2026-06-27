---
# Runbook: Queue & Scheduling Policy — ARC-1296

> Load the `execute-implementation-plan` skill before starting.
> Stop at every checkpoint symbol (🔴 🟡 🟢).
> Resume: find first unchecked box and continue.

**Feature goal:** Supervisors can view, reorder, and policy-switch the checkpoint queue so the right checkpoints are actioned first without restarting the session.
**Epic:** ARC-1296
**Lane:** queue-scheduling-policy
**Repos:** pa.aid.runbook-executor
**Skills to load at start:** execute-implementation-plan, write-implementation-plan
**Depends on:** ARC-1303 (checkpoint-management lane)

## Checkpoint model

| Symbol | Type | When |
|--------|------|------|
| 🔴 | Individual plan checkpoint | Before executing any HIGH story |
| 🟡 | Batch plan checkpoint | Before executing a batch of MEDIUM/LOW stories |
| 🟢 | Wave gate | After all stories in a wave close |

---

## Wave 1 — Queue Panel, Manual Reorder & Policy Selection

**Start condition:**
- [ ] Confirm checkpoint-management ARC-1303 merged

### Batch 1-A — ARC-1297, ARC-1298, ARC-1299 (MEDIUM / MEDIUM / MEDIUM)

- [ ] 🔒 Claimed: _(fill in: agent / YYYY-MM-DD HH:MM before starting)_
- [ ] Read all issues: ARC-1297, ARC-1298, ARC-1299
  - [ ] `issues/queue-scheduling-policy/ARC-1297-queue-panel.md`
  - [ ] `issues/queue-scheduling-policy/ARC-1298-manual-queue-reorder.md`
  - [ ] `issues/queue-scheduling-policy/ARC-1299-queue-policy.md`
- [ ] Write all implementation plans
  - [ ] `implementation_plans/queue-scheduling-policy/ARC-1297-implementation-plan.md`
  - [ ] `implementation_plans/queue-scheduling-policy/ARC-1298-implementation-plan.md`
  - [ ] `implementation_plans/queue-scheduling-policy/ARC-1299-implementation-plan.md`
- [ ] 🟡 BATCH PLAN CHECKPOINT
- [ ] Execute ARC-1297 — Display ordered checkpoint queue with priority scores
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1297-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(queue-scheduling-policy): display ordered checkpoint queue with priority scores`
- [ ] Execute ARC-1298 — Reorder checkpoint queue manually
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1298-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(queue-scheduling-policy): reorder checkpoint queue manually`
- [ ] Execute ARC-1299 — Select and switch queue policy mid-session
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1299-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(queue-scheduling-policy): select and switch queue policy mid-session`
- [ ] ☑ batch complete

### 🟢 Wave 1 Gate

- [ ] All three stories (ARC-1297, ARC-1298, ARC-1299) committed and reviewed
- [ ] 🟢 WAVE 1 COMPLETE

---

## On-Hold Register

| Story | Blocked by | Last checked | Action if unblocked |
|-------|-----------|--------------|---------------------|
| ARC-1297 | ARC-1303 not merged | — | Confirm merge, then start Wave 1 |
| ARC-1298 | ARC-1297 not done | — | Complete ARC-1297 first |
| ARC-1299 | ARC-1297 not done | — | Complete ARC-1297 first |

---

## Checkpoint Summary

| Wave | HIGH checkpoints | Batch checkpoints | Wave gates | Total |
|------|-----------------|-------------------|------------|-------|
| Wave 1 | 0 | 1 | 1 | 2 |
| **Total** | **0** | **1** | **1** | **2** |
