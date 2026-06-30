# Runbook: Failure Handling & Escalation — ARC-1300

> Load the `execute-implementation-plan` skill before starting.
> Stop at every checkpoint symbol (🔴 🟡 🟢).
> Resume: find first unchecked box and continue.

**Feature goal:** Automatically retry failed agent steps with escalating agent levels and notify the supervisor when all retries are exhausted, so runbooks recover from transient failures without human intervention.
**Epic:** ARC-1300
**Lane:** failure-handling
**Repos:** pa.aid.runbook-executor
**Skills to load at start:** execute-implementation-plan, write-implementation-plan
**Depends on:** ARC-1288 (core-infrastructure), ARC-1302 (checkpoint-management)

## Checkpoint model

| Symbol | Type | When |
|--------|------|------|
| 🔴 | Individual plan checkpoint | Before executing any HIGH story |
| 🟡 | Batch plan checkpoint | Before executing a batch of MEDIUM/LOW stories |
| 🟢 | Wave gate | After all stories in a wave close |

---

## Wave 1 — Retry Engine & Escalation

**Start condition:**
- [x] Confirm ARC-1288 (opencode CLI subprocess) merged

### ARC-1306 — Retry failed agent steps up to configurable limit 🔴 HIGH

- [ ] **ARC-1306** — Retry failed agent steps up to configurable limit
  - [ ] 🔒 Claimed: _(fill in: agent / YYYY-MM-DD HH:MM before starting)_
  - [ ] Read `issues/failure-handling/ARC-1306-retry-failed-steps.md`
  - [ ] Write `implementation_plans/failure-handling/ARC-1306-implementation-plan.md`
  - [ ] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [ ] Execute plan
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1306-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(failure-handling): retry failed agent steps up to configurable limit`

### ARC-1307 — Escalate agent level on each retry 🟡 MEDIUM

- [ ] 🔒 Claimed: _(fill in: agent / YYYY-MM-DD HH:MM before starting)_
- [ ] Read all issues: ARC-1307
- [ ] Write all implementation plans
- [ ] 🟡 BATCH PLAN CHECKPOINT
- [ ] Execute ARC-1307
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1307-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(failure-handling): escalate agent level on each retry`
- [ ] ☑ batch complete

🟢 **WAVE 1 GATE** — ARC-1306 and ARC-1307 closed; ARC-1302 merged → proceed to Wave 2

---

## Wave 2 — Failure-as-Checkpoint Notification

**Start condition:**
- [ ] Confirm ARC-1302 (browser notification) merged

### ARC-1308 — Surface exhausted retries as supervisor checkpoint notification 🔴 HIGH

- [ ] **ARC-1308** — Surface exhausted retries as supervisor checkpoint notification
  - [ ] 🔒 Claimed: _(fill in: agent / YYYY-MM-DD HH:MM before starting)_
  - [ ] Confirm ARC-1302 (browser notification, checkpoint-management lane) is merged before starting
  - [ ] Read `issues/failure-handling/ARC-1308-surface-retries-as-checkpoint.md`
  - [ ] Write `implementation_plans/failure-handling/ARC-1308-implementation-plan.md`
  - [ ] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [ ] Execute plan
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1308-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(failure-handling): surface exhausted retries as supervisor checkpoint notification`

🟢 **WAVE 2 GATE** — ARC-1308 closed; failure-handling lane complete

---

## On-Hold Register

| Story | Blocked by | Last checked | Action if unblocked |
|-------|-----------|--------------|---------------------|
| ARC-1308 | ARC-1302 (browser notification) | — | Begin Wave 2 once ARC-1302 merged |

---

## Checkpoint Summary

| Wave | HIGH checkpoints | Batch checkpoints | Wave gates | Total |
|------|-----------------|-------------------|------------|-------|
| Wave 1 | 1 (ARC-1306) | 1 (ARC-1307) | 1 | 3 |
| Wave 2 | 1 (ARC-1308) | 0 | 1 | 2 |
| **Total** | **2** | **1** | **2** | **5** |
