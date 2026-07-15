# Runbook: Story Lifecycle — ARC-1373

> Load the `execute-implementation-plan` skill before starting.
> Stop at every checkpoint symbol (🔴 🟡 🟢).
> Resume: find first unchecked box and continue.

**Feature goal:** Replace ~25 scattered implicit state flags and 5 competing status enums with one canonical StoryLifecycleState state machine, closing the gap where code changes are never gated by human review before archiving.
**Epic:** ARC-1373
**Lane:** story-lifecycle
**Wave(s):** 1,2,3,4
**Repos:** pa.aid.conductor.ts
**Skills to load at start:** write-design-spec, execute-implementation-plan, local-code-review
**Depends on:** none

**Out-of-checklist stories (in repo, not in active scope):** None.

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

## Wave 1 — Canonical State Model

### 🔴 ARC-1374 — Add canonical StoryLifecycleState enum and per-story state field

- [x] **ARC-1374** — Add canonical StoryLifecycleState enum and per-story state field
  - [x] 🔒 Claimed: story-lifecycle / 2026-07-12 12:27
  - [x] Read `issues/story-lifecycle/ARC-1374-add-canonical-story-lifecycle-state-enum.md`
  - [x] Write `implementation_plans/story-lifecycle/ARC-1374-implementation-plan.md`
  - [x] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [x] Execute plan
  - [x] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1374-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(story-lifecycle): add canonical StoryLifecycleState enum and per-story state field`

### Wave 1 gate
- [x] All active stories in this wave checked off
- [x] All PRs merged
- [x] All completion summaries written
- [x] 🟢 WAVE GATE — notify supervisor; dependent lanes: none

## Wave 2 — Sub-Machine Extraction and Silent-Failure Fixes

### 🟡 Batch — Batch 2-A

- [x] 🔒 Claimed: story-lifecycle / 2026-07-12 14:00
- [x] Read all issues: ARC-1375, ARC-1377, ARC-1379, ARC-1380
  - Files: `issues/story-lifecycle/ARC-1375-extract-generic-pr-review-loop-sub-machine.md`, `issues/story-lifecycle/ARC-1377-track-agent-self-review-state.md`, `issues/story-lifecycle/ARC-1379-fix-worktree-provisioning-silent-failure.md`, `issues/story-lifecycle/ARC-1380-fix-archive-guard-silent-failure.md`
- [ ] Write all plans:
  - [x] `implementation_plans/story-lifecycle/ARC-1375-implementation-plan.md`
  - [x] `implementation_plans/story-lifecycle/ARC-1377-implementation-plan.md`
  - [x] `implementation_plans/story-lifecycle/ARC-1379-implementation-plan.md`
  - [x] `implementation_plans/story-lifecycle/ARC-1380-implementation-plan.md`
- [x] 🟡 BATCH PLAN CHECKPOINT
- [x] Execute ARC-1375 — Extract generic PR-review-loop sub-machine parametrized by artifact kind
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1375-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(story-lifecycle): ARC-1375 implementation`
- [x] Execute ARC-1377 — Track agent self-review state explicitly
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1377-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(story-lifecycle): ARC-1377 implementation`
- [x] Execute ARC-1379 — Fix worktree-provisioning silent failure
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1379-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(story-lifecycle): ARC-1379 implementation`
- [x] Execute ARC-1380 — Fix archive-guard silent failure
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1380-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(story-lifecycle): ARC-1380 implementation`
- [x] ☑ all complete

### Wave 2 gate
- [x] All active stories in this wave checked off
- [x] All PRs merged
- [x] All completion summaries written
- [x] 🟢 WAVE GATE — notify supervisor; dependent lanes: none

## Wave 3 — Gate Migration and New Review Gate

> Gate: Dependent lanes (checkpoint-management, parallel-lane-execution) should watch for Wave 3 completion before touching checkpointGit.ts/laneRunner.ts concurrently.

### 🔴 ARC-1376 — Migrate planning-artifact gate onto shared sub-machine (supersedes ARC-1372)

- [x] **ARC-1376** — Migrate planning-artifact gate onto shared sub-machine (supersedes ARC-1372) Depends on ARC-1375 (shared sub-machine mechanism)
  - [x] 🔒 Claimed: story-lifecycle / 2026-07-13 15:50
  - [x] Read `issues/story-lifecycle/ARC-1376-migrate-planning-artifact-gate-onto-shared-sub-machine.md`
  - [x] Write `implementation_plans/story-lifecycle/ARC-1376-implementation-plan.md`
  - [x] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [x] Execute plan
  - [x] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1376-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(story-lifecycle): migrate planning-artifact gate onto shared PR-review sub-machine`

### 🔴 ARC-1378 — Add new human implementation-review gate after self-review passes

- [x] **ARC-1378** — Add new human implementation-review gate after self-review passes Depends on ARC-1375 (shared sub-machine) and ARC-1377 (self-review state tracking)
  - [x] 🔒 Claimed: story-lifecycle / 2026-07-14 09:26
  - [x] Read `issues/story-lifecycle/ARC-1378-add-implementation-review-gate.md`
  - [x] Write `implementation_plans/story-lifecycle/ARC-1378-implementation-plan.md`
  - [x] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [x] Execute plan
  - [x] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1378-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(story-lifecycle): add human implementation-review gate`

### Wave 3 gate
- [x] All active stories in this wave checked off
- [x] All PRs merged
- [x] All completion summaries written
- [x] 🟢 WAVE GATE — notify supervisor; dependent lanes: none

## Wave 4 — Rollups, Docs, and Regression Coverage

### 🟡 Batch — Batch 4-A

- [x] 🔒 Claimed: story-lifecycle / 2026-07-15 10:15
- [x] Read all issues: ARC-1381, ARC-1382, ARC-1383
  - Files: `issues/story-lifecycle/ARC-1381-derive-legacy-status-enums-as-rollups.md`, `issues/story-lifecycle/ARC-1382-add-implementation-review-checkpoint-to-template.md`, `issues/story-lifecycle/ARC-1383-end-to-end-lifecycle-regression-tests.md`
- [x] Write all plans:
  - [x] `implementation_plans/story-lifecycle/ARC-1381-implementation-plan.md`
  - [x] `implementation_plans/story-lifecycle/ARC-1382-implementation-plan.md`
  - [x] `implementation_plans/story-lifecycle/ARC-1383-implementation-plan.md`
- [x] 🟡 BATCH PLAN CHECKPOINT
- [x] Execute ARC-1381 — Derive legacy status enums as computed rollups from canonical state Depends on ARC-1374, ARC-1376, ARC-1378
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1381-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(story-lifecycle): ARC-1381 implementation`
- [x] Execute ARC-1382 — Add implementation-review checkpoint recognition to runbook template/docs/parser Depends on ARC-1378
  - [x] Lint / tests pass
  - [x] Write `task-completions/ARC-1382-COMPLETION-SUMMARY.md`
  - [x] Commit: `feat(story-lifecycle): ARC-1382 implementation`
- [ ] Execute ARC-1383 — End-to-end lifecycle regression tests Depends on all prior stories in this epic (ARC-1374 through ARC-1382)
  - [ ] Lint / tests pass
  - [x] Write `task-completions/ARC-1383-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat(story-lifecycle): ARC-1383 implementation`
- [ ] ☑ all complete

### Wave 4 gate
- [ ] All active stories in this wave checked off
- [ ] All PRs merged
- [ ] All completion summaries written
- [ ] 🟢 WAVE GATE — notify supervisor; dependent lanes: none

## On-Hold Register

No items on hold.

## Checkpoint Summary

| Wave | HIGH checkpoints | Batch checkpoints | Wave gates | Total |
|------|-----------------|-------------------|------------|-------|
| 1 | 1 | 0 | 1 | 2 |
| 2 | 0 | 1 | 1 | 2 |
| 3 | 2 | 0 | 1 | 3 |
| 4 | 0 | 1 | 1 | 2 |
