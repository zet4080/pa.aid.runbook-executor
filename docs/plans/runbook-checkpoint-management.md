# Runbook: Checkpoint Management — ARC-1295

> Load the `execute-implementation-plan` skill before starting.
> Stop at every checkpoint symbol (🔴 🟡 🟢).
> Resume: find first unchecked box and continue.

**Feature goal:** Supervisors can review checkpoint artifacts as Bitbucket PR inline comments and resume lanes via persistent [Resume] cards in the session UI — with a live priority queue — so no lane proceeds past a gate without human sign-off.
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

- [x] **ARC-1301** — Detect checkpoint markers and pause lane execution
  - [x] 🔒 Claimed: build-agent / 2026-06-30 19:45
  - [x] Read `issues/checkpoint-management/ARC-1301-detect-checkpoint-markers.md`
  - [x] Write `implementation_plans/checkpoint-management/ARC-1301-implementation-plan.md`
  - [x] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [x] Execute plan
  - [x] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1301-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(checkpoint-management): detect checkpoint markers and pause lane execution`

### 🟢 Wave 1 Gate

- [x] 🟢 WAVE 1 GATE — ARC-1301 closed and merged

---

## Wave 2 — PR Creation, Resume Card, and Queue Display

### Batch 2-A: ARC-1302, ARC-1303

- [x] 🔒 Claimed: build-agent / 2026-06-30 00:00
- [x] Read all issues: ARC-1302, ARC-1303
- [x] Write all implementation plans
- [x] 🟡 BATCH PLAN CHECKPOINT
- [x] Execute ARC-1302
  - [x] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1302-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(checkpoint-management): create Bitbucket PR and display persistent Resume card on checkpoint hit`
- [x] Execute ARC-1303
  - [x] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1303-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(checkpoint-management): ARC-1303 checkpoint queue panel with priority scoring`
- [x] ☑ batch 2-A complete

### ARC-1304 — Read PR comment threads and address as required action items 🔴 HIGH

- [ ] **ARC-1304** — Read PR comment threads and address as required action items
  - [x] 🔒 Claimed: checkpoint-management / 2026-07-05 18:33
  - [x] Read `issues/checkpoint-management/ARC-1304-inline-edit-artifacts.md`
  - [x] Write `implementation_plans/checkpoint-management/ARC-1304-implementation-plan.md`
  - [x] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [ ] Execute plan
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1304-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(checkpoint-management): read PR comment threads on Resume and address each as required action item`

### 🟢 Wave 2 Gate

- [ ] 🟢 WAVE 2 GATE — ARC-1302, ARC-1303, ARC-1304 all closed and merged

---

## Wave 3 — Resume Card Persistence and Per-Lane Isolation

### Batch 3-A: ARC-1305

- [ ] 🔒 Claimed: _(fill in: agent / YYYY-MM-DD HH:MM before starting)_
- [ ] Read all issues: ARC-1305
- [ ] Write all implementation plans
- [ ] 🟡 BATCH PLAN CHECKPOINT
- [ ] Execute ARC-1305
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1305-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(checkpoint-management): display persistent Resume card and handle Resume click per lane`
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
