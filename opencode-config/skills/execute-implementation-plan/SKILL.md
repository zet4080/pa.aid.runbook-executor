---
name: execute-implementation-plan
description: Use when converting an approved implementation plan into code changes, tests, and a completion summary
---

# Execute Implementation Plan

## Purpose

Convert approved implementation plan into committed code, verified against each step's acceptance criteria. Scope enforcement: implement exactly what the plan specifies, no bonus refactors or expansion.

## When to Use

- Converting approved implementation plan into code changes
- After issue and plan are written and approved
- Before local-code-review and PR creation

---

## Overview

Convert an approved plan into committed code, verified against each step's acceptance criteria.

**Core principle:** Implement exactly what the plan specifies. No bonus refactors, no scope expansion.

## Pre-flight Checklist

Before starting, verify all of the following. Stop immediately if any item fails.

**Artifact verification (REQUIRED):**
- [ ] Issue file exists at `issues/{EPIC}/{TRACKER-ID}-*.md` in the planning repo — if missing, stop and report
- [ ] Implementation plan exists at `implementation_plans/{EPIC}/{TRACKER-ID}-implementation-plan.md` — if missing, stop and report
- [ ] Active runbook identified at `docs/plans/runbook-*.md` in the planning repo

**Readiness:**
- [ ] Implementation plan is approved
- [ ] Any open questions in the plan are resolved

If any artifact is missing, stop and report exactly which file is absent. Do not guess paths or proceed without them.

## Execution Steps (in order)

### 1. Review Plan Critically

Before touching code:
- Identify anything unclear, contradictory, or missing
- Raise concerns with user and resolve them
- Do NOT start implementing with unresolved critical questions

### 2. Reconcile Plan vs Repository Reality

- Verify referenced files and modules exist at stated paths
- Note any discrepancies; ask user if significant
- Confirm test framework and build commands

### 3. Implement in Small Increments

- Implement one plan step at a time
- Commit after each completed step (see commit format below)
- Run step verification after each step before proceeding
- **After each step:** check off the corresponding item in the active runbook (see Runbook Update Protocol below) and commit that update directly to `main`

### 4. Update Docs and Artifacts

- Create completion summary when all steps done
- Update issue Completion Tracking section
- **After ALL steps complete:** verify every corresponding runbook checkbox is marked done; commit any remaining runbook updates directly to `main`

## The Iron Law (Verification)

**NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE.**

Gate function:
1. IDENTIFY what command proves the claim
2. RUN it fresh (don't rely on earlier output)
3. READ full output
4. VERIFY result matches claim
5. ONLY THEN make the claim

Never say "tests pass" without running them and reading the output.

## Scope Rules

- Implement exactly what the plan specifies
- No bonus refactors
- No "while I'm here" improvements
- Preserve existing APIs unless plan explicitly changes them
- If you discover something that needs changing beyond plan scope → stop, document, ask

## Runbook Update Protocol

The active runbook lives at `docs/plans/runbook-*.md` in the planning repo.

**After completing each implementation step:**
1. Locate the checkbox for this story/step in the runbook — e.g., `- [ ] Implement ABC-123`
2. Mark it done: `- [x] Implement ABC-123`
3. Commit the change **directly to `main`** (not the feature branch)

**After ALL steps are done:**
- Verify every checkbox corresponding to this story is marked `[x]`
- If any were missed, update and commit to `main` now

**After creating the completion summary:**
- If the runbook has a completion-summary checkbox (e.g., `- [ ] Write completion summary for ABC-123`), mark it done and commit to `main`

**Artifact commit rule:**
Planning and tracking artifacts — runbook updates, completion summaries, issue Completion Tracking updates — are ALWAYS committed directly to `main`, never to the feature branch.

Commit format for runbook/artifact updates:
```
chore: check off {TRACKER-ID} step in runbook
```

---

## Commit Format

```
{TRACKER-ID}: {description} - {AI Agent name}, {duration}, {cost}
```

Commit after each completed plan step — not every file change, not all at end.

## STOP Rules

Stop and ask the user when:
- Blocked by missing context
- Plan has a critical gap discovered during implementation
- Instruction is genuinely ambiguous
- Verification fails repeatedly with no clear path forward
- Review loop hit 3 iterations and BLOCKER/ISSUE findings remain (escalate, don't keep fixing)

Ask, don't guess.

## Definition of Done

- [ ] All plan steps implemented
- [ ] Each step's verification passed (evidence collected)
- [ ] All existing tests pass
- [ ] Completion summary created
- [ ] Issue Completion Tracking updated
- [ ] Commits follow required format
- [ ] All runbook checkboxes for this story marked `[x]` and committed to `main`

## Review and Fix Loop

After all plan steps are complete and the Definition of Done checklist passes, run `local-code-review` before proceeding to `finishing-a-development-branch`.

The loop:
1. Run `local-code-review` (tests + lint + type-check + diff vs AC)
2. If BLOCKER or ISSUE findings → fix → commit → re-run (max 3 iterations)
3. If iteration 3 still has BLOCKER/ISSUE → **STOP. Escalate to human.** Do not create PR.
4. If clean (no BLOCKER, no ISSUE) → proceed to `finishing-a-development-branch`

SUGGESTION and NIT findings are non-blocking — they may remain open.

**Human escalation triggers (stop and ask):**
- Scope change discovered during review
- Architectural decision required to fix a finding
- Security-sensitive finding
- Review loop hit 3 iterations with unresolved blockers

## Agent Self-Check

Before claiming done, verify:
- [ ] Fresh test run output reviewed (not cached)
- [ ] No TODO/FIXME left from implementation
- [ ] Completion summary maps criteria to evidence
- [ ] No scope creep introduced
- [ ] All runbook checkboxes for this story marked `[x]` and committed to `main`

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Starting with unresolved plan questions | Ask first, implement after |
| Starting without verifying issue/plan artifacts exist | Run pre-flight checklist; stop if artifact missing |
| Committing everything at end | Commit after each plan step |
| Claiming tests pass from memory | Run fresh, read output |
| Adding improvements not in plan | Scope rules — stay on plan |
| Guessing when blocked | STOP and ask |
| Proceeding to PR without running local-code-review | Run local-code-review after all steps; fix blockers/issues before pushing |
| Forgetting to check off runbook after each step | Runbook Update Protocol — mark `[x]` and commit to `main` after every step |
| Committing runbook/artifact updates to feature branch | Artifacts go to `main` directly, never feature branch |
