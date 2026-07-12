# Runbook: Checkpoint Management — ARC-1295

> Load the `execute-implementation-plan` skill before starting.
> Stop at every checkpoint symbol (🔴 🟡 🟢).
> Resume: find first unchecked box and continue.

**Feature goal:** Supervisors review checkpoint artifacts via Bitbucket PR inline comments and resume lanes via persistent Resume cards in the session UI — live priority queue — so no lane proceeds past a gate without human sign-off
**Epic:** ARC-1295
**Lane:** checkpoint-management
**Wave(s):** 5
**Repos:** pa.aid.runbook-executor, pa.aid.conductor.ts
**Skills to load at start:** execute-implementation-plan, write-implementation-plan
**Depends on:** ARC-1291 (parallel-lane-execution — sequential step execution)

**Out-of-checklist stories (in repo, not in active scope):** On-hold register tracks original blocking conditions; Waves 1–4 are fully complete.

## Checkpoint model

| Symbol | Type | When |
|--------|------|------|
| 🔴 | Individual plan checkpoint | Before executing any HIGH story |
| 🟡 | Batch plan checkpoint | Before executing a batch of MEDIUM/LOW stories |
| 🟢 | Wave gate | After all stories in a wave close |

## HIGH story workflow
1. Read issue file
2. Write implementation plan → `implementation_plans/{lane}/{KEY}-implementation-plan.md`
3. 🔴 CHECKPOINT — human reviews plan before execution
4. Execute plan
5. Lint / tests pass
6. Write completion summary → `task-completions/{KEY}-COMPLETION-SUMMARY.md`
7. Commit + ☑

## MEDIUM/LOW batch workflow
1. Read all issue files in batch
2. Write all implementation plans
3. 🟡 CHECKPOINT — human reviews all plans
4. Execute all plans in sequence
5. Lint / tests pass for all
6. Write all completion summaries
7. Commit all + ☑ all

---

## Wave 1 — Checkpoint Detection

> Gate: ARC-1301 closed and merged

- [ ] Confirm Confirm parallel-lane-execution ARC-1291 merged

### 🔴 ARC-1301 — Detect checkpoint markers and pause lane execution

- [x] **ARC-1301** — Detect checkpoint markers and pause lane execution
  - [x] 🔒 Claimed: ✅
  - [x] Read `issues/checkpoint-management/ARC-1301-detect-checkpoint-markers.md`
  - [x] Write `implementation_plans/checkpoint-management/ARC-1301-implementation-plan.md`
  - [x] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [x] Execute plan
  - [x] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1301-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(checkpoint-management): detect checkpoint markers and pause lane execution`
  - [ ] Claimed: build-agent / 2026-06-30 19:45

### Wave 1 gate
- [ ] All active stories in this wave checked off
- [ ] All PRs merged
- [ ] All completion summaries written
- [ ] 🟢 WAVE GATE — notify supervisor; dependent lanes: none

## Wave 2 — PR Creation, Resume Card, and Queue Display

> Gate: ARC-1302, ARC-1303, ARC-1304 all closed and merged

### 🔴 ARC-1304 — Read PR comment threads and address as required action items

- [x] **ARC-1304** — Read PR comment threads and address as required action items
  - [x] 🔒 Claimed: ✅
  - [x] Read `issues/checkpoint-management/ARC-1304-inline-edit-artifacts.md`
  - [x] Write `implementation_plans/checkpoint-management/ARC-1304-implementation-plan.md`
  - [x] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [x] Execute plan
  - [x] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1304-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(checkpoint-management): read PR comment threads on Resume and address required action items`
  - [ ] Claimed: checkpoint-management / 2026-07-05 18:33

### 🟡 Batch — Batch 2-A

- [x] 🔒 Claimed: ✅
- [x] Read all issues: ARC-1302, ARC-1303
  - Files: `issues/checkpoint-management/ARC-1302-browser-notification.md`, `issues/checkpoint-management/ARC-1303-checkpoint-queue-panel.md`
- [x] Write all plans:
  - [x] `implementation_plans/checkpoint-management/ARC-1302-implementation-plan.md`
  - [x] `implementation_plans/checkpoint-management/ARC-1303-implementation-plan.md`
- [x] 🟡 BATCH PLAN CHECKPOINT
- [x] Execute ARC-1302 — Deliver browser notification within 5 seconds of checkpoint hit
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1302-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(checkpoint-management): deliver browser notification on checkpoint hit`
- [x] Execute ARC-1303 — Display checkpoint queue with automatic priority scoring
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1303-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(checkpoint-management): checkpoint queue panel with priority scoring`
- [x] ☑ all complete

### Wave 2 gate
- [ ] All active stories in this wave checked off
- [ ] All PRs merged
- [ ] All completion summaries written
- [ ] 🟢 WAVE GATE — notify supervisor; dependent lanes: none

## Wave 3 — Resume Card Persistence and Per-Lane Isolation

> Gate: ARC-1305 closed and merged

### 🟡 Batch — Batch 3-A

- [x] 🔒 Claimed: ✅
- [x] Read all issues: ARC-1305
  - Files: `issues/checkpoint-management/ARC-1305-approve-reject-checkpoint.md`
- [x] Write all plans:
  - [x] `implementation_plans/checkpoint-management/ARC-1305-implementation-plan.md`
- [x] 🟡 BATCH PLAN CHECKPOINT
- [x] Execute ARC-1305 — Approve or reject checkpoint inline with feedback
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1305-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(checkpoint-management): display persistent Resume card and handle Resume click per lane`
- [x] ☑ all complete

### Wave 3 gate
- [ ] All active stories in this wave checked off
- [ ] All PRs merged
- [ ] All completion summaries written
- [ ] 🟢 WAVE GATE — notify supervisor; dependent lanes: none

## Wave 4 — Agent Last Message on Checkpoint Resume Card

> Gate: ARC-1365 closed and merged

### 🟡 Batch — Batch 4-A

- [x] 🔒 Claimed: ✅
- [x] Read all issues: ARC-1365
  - Files: `issues/checkpoint-management/ARC-1365-agent-last-message-on-checkpoint.md`
- [x] Write all plans:
  - [x] `implementation_plans/checkpoint-management/ARC-1365-implementation-plan.md`
- [x] 🟡 BATCH PLAN CHECKPOINT
- [x] Execute ARC-1365 — Display agent last message on checkpoint Resume card
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1365-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(checkpoint-management): display agent last message on checkpoint Resume card`
- [x] ☑ all complete

### Wave 4 gate
- [x] All active stories in this wave checked off
- [x] All PRs merged
- [x] All completion summaries written
- [x] 🟢 WAVE GATE — notify supervisor; dependent lanes: none

## Wave 5 — PR Approval Gate for Planning Artifacts

> Gate: ARC-1371 closed and merged; planning artifact review is fully PR-based ARC-1372 depends on ARC-1371 + ARC-1304 (both already merged).

- [x] Confirm Confirm Wave 4 gate passed
- [x] Confirm Confirm ARC-1366 (resume_checkpoint) merged

### 🔴 ARC-1371 — Conductor: open PR for implementation plans and decision documents as approval gate

- [x] **ARC-1371** — Conductor: open PR for implementation plans and decision documents as approval gate Depends on ARC-1366 (resume_checkpoint) for PR-merge detection; depends on ARC-1369 (auto-commit artifacts) for branch+commit infrastructure
  - [x] 🔒 Claimed: checkpoint-management / 2026-07-07 19:41
  - [x] Read `issues/checkpoint-management/ARC-1371-conductor-pr-approval-gate-for-plans-and-decisions.md`
  - [x] Write `implementation_plans/checkpoint-management/ARC-1371-implementation-plan.md`
  - [x] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [x] Execute plan
  - [x] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1371-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(checkpoint-management): PR approval gate for implementation plans and decision documents`

### 🔴 ARC-1372 — Unify planning-artifact checkpoint gate with PR comment-resolution loop

- [ ] **ARC-1372** — Unify planning-artifact checkpoint gate with PR comment-resolution loop Depends on ARC-1371 (planning-artifact PR creation and pause mechanism this issue unifies); depends on ARC-1304 (comment-resolution loop this issue reuses)
  - [ ] 🔒 Claimed: _(fill in: lane / YYYY-MM-DD HH:MM before starting)_
  - [ ] Read `issues/checkpoint-management/ARC-1372-unify-planning-artifact-gate-with-comment-loop.md`
  - [ ] Write `implementation_plans/checkpoint-management/ARC-1372-implementation-plan.md`
  - [ ] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [ ] Execute plan
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/ARC-1372-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(checkpoint-management): unify planning-artifact gate with comment loop`

### On-Hold story — ARC-1301

If ARC-1291 (parallel-lane-execution) all complete:
- [ ] 🔒 Claimed: _(fill in: lane / YYYY-MM-DD HH:MM before starting)_
- [ ] Read `issues/checkpoint-management/ARC-1301.md`
- [ ] Write `implementation_plans/checkpoint-management/ARC-1301-implementation-plan.md`
- [ ] 🟡 BATCH PLAN CHECKPOINT
- [ ] Execute plan
- [ ] Lint / tests pass
- [ ] Write `task-completions/ARC-1301-COMPLETION-SUMMARY.md`
- [ ] Commit: `feat(checkpoint-management): ARC-1301 implementation`

If not: log in On-Hold Register below and skip.

### On-Hold story — ARC-1305

If ARC-1303 (intra-lane), ARC-1304 (intra-lane) all complete:
- [ ] 🔒 Claimed: _(fill in: lane / YYYY-MM-DD HH:MM before starting)_
- [ ] Read `issues/checkpoint-management/ARC-1305.md`
- [ ] Write `implementation_plans/checkpoint-management/ARC-1305-implementation-plan.md`
- [ ] 🟡 BATCH PLAN CHECKPOINT
- [ ] Execute plan
- [ ] Lint / tests pass
- [ ] Write `task-completions/ARC-1305-COMPLETION-SUMMARY.md`
- [ ] Commit: `feat(checkpoint-management): ARC-1305 implementation`

If not: log in On-Hold Register below and skip.

### Wave 5 gate
- [ ] All active stories in this wave checked off
- [ ] All PRs merged
- [ ] All completion summaries written
- [ ] 🟢 WAVE GATE — notify supervisor; dependent lanes: none

## On-Hold Register

| Story | Blocked by | Last checked | Action if unblocked |
|-------|-----------|--------------|---------------------|
| ARC-1301 | ARC-1291 (parallel-lane-execution) | — | Begin Wave 1 once ARC-1291 merged |
| ARC-1305 | ARC-1303 (intra-lane), ARC-1304 (intra-lane) | — | Begin Batch 3-A once Wave 2 gate passes |

## Checkpoint Summary

| Wave | HIGH checkpoints | Batch checkpoints | Wave gates | Total |
|------|-----------------|-------------------|------------|-------|
| 1 | 1 | 0 | 1 | 2 |
| 2 | 1 | 1 | 1 | 3 |
| 3 | 0 | 1 | 1 | 2 |
| 4 | 0 | 1 | 1 | 2 |
| 5 | 2 | 2 | 1 | 5 |
