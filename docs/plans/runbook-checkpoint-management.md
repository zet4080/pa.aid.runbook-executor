# Runbook: Checkpoint Management — ARC-1295

> Load the `execute-implementation-plan` skill before starting.
> Stop at every checkpoint symbol (🔴 🟡 🟢).
> Resume: find first unchecked box and continue.

**Feature goal:** Supervisors can review, edit, approve, or reject execution checkpoints from the browser — with instant notifications and a live priority queue — so no lane proceeds past a gate without human sign-off.
**Epic:** ARC-1295
**Lane:** checkpoint-management
**Repos:** pa.aid.runbook-executor
**Skills to load at start:** execute-implementation-plan, write-implementation-plan
**Depends on:** ARC-1291 (parallel-lane-execution)

## Checkpoint model

| Symbol | Type | When |
|--------|------|------|
| 🔴 | Individual plan checkpoint | Before executing any HIGH story |
| 🟡 | Batch plan checkpoint | Before executing a batch of MEDIUM/LOW stories |
| 🟢 | Wave gate | After all stories in a wave close |

---

## Pre-flight

- [ ] Confirm parallel-lane-execution ARC-1291 merged

---

## Wave 1 — Checkpoint Detection

### ARC-1301 — Detect checkpoint markers and pause lane execution 🔴 HIGH

- [ ] **ARC-1301** — Detect checkpoint markers and pause lane execution
  - [ ] 🔒 Claimed: _(fill in: agent / YYYY-MM-DD HH:MM before starting)_
  - [ ] Read `issues/checkpoint-management/ARC-1301-detect-checkpoint-markers.md`
  - [ ] Write `implementation_plans/checkpoint-management/ARC-1301-implementation-plan.md`
  - [ ] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [ ] Execute plan
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1301-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(checkpoint-management): detect checkpoint markers and pause lane execution`

### 🟢 Wave 1 Gate

- [ ] 🟢 WAVE 1 GATE — ARC-1301 closed and merged

---

## Wave 2 — Notification, Queue Display, and Artifact Editing

### Batch 2-A: ARC-1302, ARC-1303

- [ ] 🔒 Claimed: _(fill in: agent / YYYY-MM-DD HH:MM before starting)_
- [ ] Read all issues: ARC-1302, ARC-1303
- [ ] Write all implementation plans
- [ ] 🟡 BATCH PLAN CHECKPOINT
- [ ] Execute ARC-1302
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1302-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(checkpoint-management): deliver browser notification within 5 seconds of checkpoint hit`
- [ ] Execute ARC-1303
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1303-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(checkpoint-management): display checkpoint queue with automatic priority scoring`
- [ ] ☑ batch 2-A complete

### ARC-1304 — Inline-edit checkpoint artifacts and persist handoff file to disk 🔴 HIGH

- [ ] **ARC-1304** — Inline-edit checkpoint artifacts and persist handoff file to disk
  - [ ] 🔒 Claimed: _(fill in: agent / YYYY-MM-DD HH:MM before starting)_
  - [ ] Read `issues/checkpoint-management/ARC-1304-inline-edit-artifacts.md`
  - [ ] Write `implementation_plans/checkpoint-management/ARC-1304-implementation-plan.md`
  - [ ] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [ ] Execute plan
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1304-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(checkpoint-management): inline-edit checkpoint artifacts and persist handoff file to disk`

### 🟢 Wave 2 Gate

- [ ] 🟢 WAVE 2 GATE — ARC-1302, ARC-1303, ARC-1304 all closed and merged

---

## Wave 3 — Checkpoint Resolution

### Batch 3-A: ARC-1305

- [ ] 🔒 Claimed: _(fill in: agent / YYYY-MM-DD HH:MM before starting)_
- [ ] Read all issues: ARC-1305
- [ ] Write all implementation plans
- [ ] 🟡 BATCH PLAN CHECKPOINT
- [ ] Execute ARC-1305
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1305-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(checkpoint-management): approve or reject checkpoint inline with feedback`
- [ ] ☑ batch 3-A complete

### 🟢 Wave 3 Gate

- [ ] 🟢 WAVE 3 GATE — ARC-1305 closed and merged

---

## On-Hold Register

| Story | Blocked by | Last checked | Action if unblocked |
|-------|-----------|--------------|---------------------|
| ARC-1301 | ARC-1291 (parallel-lane-execution) | — | Begin Wave 1 once ARC-1291 merged |
| ARC-1305 | ARC-1303, ARC-1304 (intra-lane) | — | Begin Batch 3-A once Wave 2 gate passes |

---

## Checkpoint Summary

| Wave | HIGH checkpoints | Batch checkpoints | Wave gates | Total |
|------|-----------------|-------------------|------------|-------|
| Wave 1 | 1 | 0 | 1 | 2 |
| Wave 2 | 1 | 1 | 1 | 3 |
| Wave 3 | 0 | 1 | 1 | 2 |
| **Total** | **2** | **2** | **3** | **7** |
