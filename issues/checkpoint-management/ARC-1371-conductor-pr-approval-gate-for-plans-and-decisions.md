| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | In Review |
| Completion | 90% |
| Last Updated | 2026-07-09 |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-06 |
| Parent Epic | ARC-1295 — ARC: Checkpoint Management |
| Jira | ARC-1371 |

# ARC-1371: Conductor: open PR for implementation plans and decision documents as approval gate

## Goal

When the conductor writes an implementation plan or a decision document it should commit the artifact to a short-lived branch, open a Bitbucket PR targeting `main`, pause the lane at a 🔴 checkpoint, and resume execution only after the PR is merged. PR merge is the canonical approval signal — no manual `status: answered` editing required. Supervisors review and annotate the artifact inline on Bitbucket; merging the PR unblocks the lane.

## Problem

Implementation plans and decision documents currently land directly on `main` and are reviewed by the supervisor by reading a raw markdown file. Approval is signalled informally (verbally or by editing `status: answered` in the decision document). This gives supervisors no inline comment capability, no structured approval signal, and no audit trail. The conductor already creates Bitbucket PRs for execution checkpoints — the same mechanism should be extended to planning artifacts so every human approval gate is PR-based and traceable.

## Acceptance Criteria

**Given** The conductor writes an implementation plan file for a HIGH story,
**When** The file is ready to commit,
**Then** The conductor creates branch `plan/{lane}/{KEY}`, commits the file there, opens a Bitbucket PR titled `Plan review: {KEY} — {story title}` targeting `main`, and pauses the lane at a 🔴 checkpoint.

---

**Given** The conductor writes a decision document file,
**When** The file is ready to commit,
**Then** The conductor creates branch `decision/{KEY}-{type}`, commits the file there, opens a Bitbucket PR titled `Decision: {KEY} — {type}` targeting `main`, and pauses the lane at a 🔴 checkpoint.

---

**Given** A plan or decision PR is open and the supervisor merges it on Bitbucket,
**When** The conductor detects the merge (via poll or webhook),
**Then** The lane resumes from the checkpoint; for decision documents the conductor reads the merged file from `main` to extract `selected_approach` before continuing.

---

**Given** A plan PR is merged,
**When** The conductor resumes the lane,
**Then** The implementation plan file is now on `main` and the agent proceeds to execute the plan.

---

**Given** A decision PR is merged,
**When** The conductor resumes the lane,
**Then** The decision document's `status` is read from the merged file; if `status: answered` the lane continues with `selected_approach`; if still `pending` the conductor re-pauses and logs a warning.

---

**Given** PR creation fails (Bitbucket unavailable),
**When** The conductor tries to open the PR,
**Then** The artifact is committed directly to `main` as a fallback, the checkpoint is still raised as a 🔴 pause, and the supervisor is notified via the checkpoint queue that no PR was created.

---

**Given** The feature is implemented,
**When** An agent writes an implementation plan or decision document,
**Then** The `write-implementation-plan` and `start-execution-session` skills no longer instruct agents to commit artifacts manually or to edit `status: answered` in decision files.

## In Scope

- Extend `checkpointGit.ts` (or a new `planningArtifactGit.ts` module) with `createPlanPR()` and `createDecisionPR()` functions reusing the existing PR-creation pattern
- Branch naming: `plan/{lane}/{KEY}` for implementation plans; `decision/{KEY}-{type}` for decision documents
- PR titles: `Plan review: {KEY} — {story title}` and `Decision: {KEY} — {type}`
- PR description: include artifact path, lane, story key, and a short summary of what the supervisor must decide
- Pause mechanism: reuse the existing `CheckpointQueueEntry` + `laneStatusMap` pause pattern
- Resume trigger: conductor polls the Bitbucket PR status (reuse or extend `resume_checkpoint` mechanism from ARC-1366) and resumes when PR state is `MERGED`
- Decision document post-merge: conductor reads merged file from `main`, extracts `selected_approach`, passes it as context to the next agent prompt
- Fallback: if PR creation fails, commit directly to `main` and raise checkpoint without PR URL
- Update `write-implementation-plan` skill to remove manual commit + stop instructions
- Update `start-execution-session` skill to remove `status: answered` editing instructions for decision documents
- Unit tests: PR branch created with correct name; pause fires; resume-on-merge path; fallback-to-main path

## Out of Scope

- Completion summaries — these require no approval gate and continue to be committed directly to `main` (ARC-1369)
- Changing the Bitbucket PR review process for application code branches — this only covers planning repo artifacts
- Webhook infrastructure — polling is sufficient for MVP; webhook support is a future enhancement
- Auto-populating `status: answered` in the decision file — the supervisor writes the decision in the PR description or directly in the file before merging
- Blocking PR creation on missing Bitbucket credentials — the fallback path handles this gracefully

## Constraints

- Must reuse the existing `checkpointGit.ts` PR-creation logic — do not introduce a second Bitbucket API client
- Branch must be created in the planning repo (`pa.aid.runbook-executor`), not in `pa.aid.conductor.ts`
- PR merge detection polling interval must be configurable (default: 30 s); do not hard-code
- The lane must remain fully paused (not retrying steps) while awaiting PR merge

## Dependencies

- ARC-1366 (resume_checkpoint — the PR-merge detection and lane-resume mechanism builds on this)
- ARC-1369 (auto-commit planning artifacts — this issue replaces the direct-to-main commit path for plans and decisions with a branch+PR path)

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-tools/ARC-1371-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1371-COMPLETION-SUMMARY.md` | TBD |