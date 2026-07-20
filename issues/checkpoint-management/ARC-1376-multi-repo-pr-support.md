| Field | Value |
|-------|-------|
| Type | Story |
| Priority | High |
| Status | To Do |
| Assignee | — |
| Reporter | — |
| Created | 2026-07-20 |
| Parent Epic | ARC-1295 — ARC: Checkpoint Management |
| Jira | ARC-1376 |

# ARC-1376: Support multi-repo PR creation and review for implementation checkpoint

## Goal

Extend the implementation review checkpoint (Checkpoint #2) to detect when a story's changes span multiple repositories and open one PR per affected repository, rather than assuming a single repo. The review loop must track all PRs and gate resume on ALL of them being resolved (zero unresolved comments across all PRs), not just one.

## Acceptance Criteria

**Given** A story branch has uncommitted or committed changes in multiple repositories,
**When** The implementation review checkpoint fires,
**Then** One PR is opened per affected repository.

---

**Given** Multiple PRs are open for a single story,
**When** The checkpoint is resumed via POST /api/checkpoints/:stepId/resume,
**Then** The resume handler checks for unresolved comments across ALL PRs, not just one.

---

**Given** At least one PR has unresolved comments,
**When** The checkpoint is resumed,
**Then** The checkpoint re-queues (loops) and the agent addresses comments across all affected PRs.

---

**Given** All PRs have zero unresolved comments,
**When** The checkpoint is resumed,
**Then** The lane proceeds without re-queuing.

---

**Given** All changes are complete,
**When** Tests run,
**Then** There is explicit test coverage proving multi-repo detection, one-PR-per-repo creation, and all-PRs-resolved gating.

## In Scope

- Detect which repositories have uncommitted or committed changes for a story branch
- Open one PR per affected repository (iterate over affected repos, not assume single repo)
- Track all PRs associated with a story/checkpoint
- Modify resume handler to check for unresolved comments across ALL PRs, not just one
- Gate resume on zero unresolved comments across ALL PRs
- Update createPlanningArtifactPR (laneRunner.ts ~402-477) and createCheckpointBranchAndPR (checkpointGit.ts ~94-176) to iterate over affected repos
- Add test coverage proving multi-repo detection and all-PRs-resolved gating

## Out of Scope

- Changes to Checkpoint #1 (plan review) — that checkpoint is single-repo only (implementation plans live in the planning repo)
- Changes to PR creation logic beyond supporting iteration over multiple repos
- Changes to the generate-lane-runbooks skill or runbook template — those are separate skill-maintainer tasks
- Cross-repo dependency ordering (e.g. repo A's PR must merge before repo B's PR) — not required by current spec

## Constraints

- Current PR creation functions are createPlanningArtifactPR (laneRunner.ts ~402-477) and createCheckpointBranchAndPR (checkpointGit.ts ~94-176)
- These functions each take a single repoRoot string and shell out `git -C {repoRoot} ...` — no iteration over multiple repos/remotes exists currently
- The resume handler is resumeArtifactPRReviewLoop (checkpointGit.ts:785-820) — it must be extended to iterate over all PRs associated with a checkpoint
- Multi-repo detection must be based on which repos have uncommitted or committed changes for the story branch, not a static configuration

## Dependencies

- Issue C (ARC-1375) must be complete — this issue extends the implementation review checkpoint that Issue C moves/fixes
- The generate-lane-runbooks skill and generate_runbook MCP tool currently emit the implementation review checkpoint marker BEFORE the Commit line in runbook templates — this ordering is wrong per new requirements. A separate skill-maintainer task will update the template; this issue's implementation should be coordinated with that change but does not own it.

## Completion Tracking

| Artifact | Path | Status |
|----------|------|--------|
| Implementation plan | `implementation_plans/agent-tools/ARC-1376-implementation-plan.md` | TBD |
| Completion summary | `task-completions/ARC-1376-COMPLETION-SUMMARY.md` | TBD |