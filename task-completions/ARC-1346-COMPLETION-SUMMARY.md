# ARC-1346 — Introduce Decision Document Pattern — Completion Summary

**Issue:** `issues/agent-workflow/ARC-1346-decision-document-pattern.md`
**Implementation Plan:** `implementation_plans/agent-workflow/ARC-1346-implementation-plan.md`
**Completed:** 2026-07-01
**Status:** ✅ Done

## Summary

Introduced the `decisions/` folder convention and decision document pattern to the planning repo and the `write-implementation-plan` skill. Agents facing multi-approach HIGH-risk stories now write a structured decision document, commit and push it, and stop at a checkpoint rather than blocking interactively. On resume, agents read the answered document and extract the selected approach automatically.

## Acceptance Criteria Evidence

| AC | Status | Evidence |
|----|--------|----------|
| AC 1: Agent writes `decisions/{KEY}-approach-decision.md`, commits+pushes, stops at checkpoint | ✅ | `write-implementation-plan/SKILL.md` Phase 1 updated with Path A protocol (write → commit → push → stop); verified in live skill and pa.aid.config.md repo (`125d913`) |
| AC 2: On resume, agent reads answered document, extracts `selected_approach`, proceeds without interactive prompt | ✅ | `write-implementation-plan/SKILL.md` Phase 1 updated with Path B protocol (check `status: answered` → extract `selected_approach` → proceed); interactive-only STOP removed |
| AC 3: Decision document moved to `done/decisions/` after plan is written | ✅ | `write-implementation-plan/SKILL.md` Phase 2 updated with conditional archival step (`git mv` → commit → push); `done/decisions/.gitkeep` created |

## Artifacts Created / Modified

| Artifact | Location | Notes |
|----------|----------|-------|
| `decisions/` folder | `pa.aid.runbook-executor/decisions/` | New folder for pending decision documents |
| `decisions/_template.md` | `pa.aid.runbook-executor/decisions/_template.md` | Canonical decision document template (6 frontmatter fields + 4 body sections) |
| `done/decisions/` folder | `pa.aid.runbook-executor/done/decisions/` | Archive for answered decision documents |
| `AGENTS.md` | `pa.aid.runbook-executor/AGENTS.md` | Added `decisions/` row to repo layout table |
| `write-implementation-plan` skill (live) | `/home/zimmermann/.config/opencode/skills/write-implementation-plan/SKILL.md` | Phase 1 + Phase 2 updated |
| `write-implementation-plan` skill (source) | `pa.aid.config.md` repo, branch `ARC-1346` | PR #1 open: https://bitbucket.org/proalpha/pa.aid.config.md/pull-requests/1 |

## Commits

| Repo | Commit | Description |
|------|--------|-------------|
| `pa.aid.runbook-executor` | `8e4eb23` | `docs(agent-workflow): add ARC-1346 decision document pattern artifacts` |
| `pa.aid.config.md` | `125d913` | `feat(skills): add ARC-1346 decision document protocol to write-implementation-plan` (on branch `ARC-1346`, PR #1) |

## Open Items

- `pa.aid.config.md` PR #1 requires review and merge to propagate skill changes to all future skill installations.
- ARC-1347 will extend the resume protocol to `start-execution-session` and other skills.