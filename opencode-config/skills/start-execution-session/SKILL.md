---
name: start-execution-session
description: Use when starting work on a planning repo — first time or resuming — to orient the agent, identify the active runbook and wave, find the next unchecked story, and load the right execution skills before touching any code.
---

# Start Execution Session

## Purpose

Bootstrap agent session on planning repo: load context, identify active runbook/wave, find next unchecked story, and load execution skills before touching code. Works for first-time start and resume.

## When to Use

- Starting work on planning repo (first time or resuming)
- Before executing any stories or implementation
- After runbook and workorder are generated

---

## Step 1 — Load Context

Read these files in order:

1. `AGENTS.md` — repo purpose, folder layout, sibling repos, commit format
2. `docs/plans/HOW-THIS-WORKS.md` — checkpoint model, agent workflows, wave gate checklists
3. `docs/plans/workorder.md` — lane overview, wave structure, current state

If any of these files are missing, stop and report:
> "Cannot start session — `{file}` is missing. Run `generate-lane-runbooks` first."

---

## Step 2 — Identify Active Work

### Which lane?

If the user specified a lane at invocation, use it.
If not, ask:
> "Which lane (feature) are you working on? Available lanes: {list from workorder Feature Lane Overview table}"

### Which wave?

In the workorder, find the lane's current wave:
- Wave is active if previous wave gate is closed (all boxes checked) or this is Wave 1
- If previous wave gate is NOT closed, stop:
  > "Wave {N-1} gate is not closed for lane {lane}. Complete Wave {N-1} before starting Wave {N}. Outstanding items: {list unchecked boxes}"

### Which story?

Open the lane's runbook: `docs/plans/runbook-{lane}.md`

Find the **first unchecked AND unclaimed story checkbox** in the active wave. A story is claimed if it has a checked `- [x] 🔒 Claimed:` sub-item. Skip claimed stories — another agent or session owns them.

Report:
> "Next story: **{KEY}** — {summary}
> Risk: {🔴 HIGH / 🟡 MEDIUM / 🟢 LOW}
> Checkpoint: {individual plan checkpoint / batch with {KEY2}, {KEY3}}"

### Claim the Story

**Before writing any plan or code**, mark the story or batch as claimed in the runbook:

Update the existing unchecked `- [ ] 🔒 Claimed:` row; do not add a second Claimed row. Use the row's current indentation:
```
  - [x] 🔒 Claimed: {lane} / {YYYY-MM-DD HH:MM}
```

If the runbook has no existing `🔒 Claimed:` row for the story or batch, stop: the generated runbook is malformed. Repair or regenerate it before starting work.

Commit immediately:
```bash
git add docs/plans/runbook-{lane}.md
git commit -m "chore({lane}): claim {KEY}"
```

This prevents a parallel agent from picking the same story.

### Check Live Worktrees

```bash
git worktree list
```

Report active worktrees to the user. If the target branch already exists as a worktree → resume that worktree instead of creating a new one.

> **Branch Naming Rule:** When creating a feature branch for a story, the branch name MUST be exactly `<issueid>` — no prefix (`fix/`, `feature/`), no suffix, no description.
> Example: `git worktree add ../<issueid> -b <issueid>`
> ✅ Correct: `ARC-123`
> ❌ Wrong: `ARC-123-add-login`, `feature/ARC-123`

> **Runbook artifact commits go to main:** All runbook updates (checking off steps, wave gates) and planning artifacts (implementation plans, completion summaries) are committed directly to `main`, never to a feature branch.

---

## Step 3 — Check Cross-Lane Dependencies

Before starting, verify any gate checks preceding this story are satisfied:
- Look for `- [ ] Confirm {Lane} Wave N gate passed` checkboxes before this story
- If any are unchecked, stop:
  > "Cross-lane dependency not met: {Lane} Wave {N} gate must pass before this story can start. Check with supervisor."

---

## Step 4 — Load Execution Skills

Load in this order:
1. `write-implementation-plan` — for writing the plan
2. `execute-implementation-plan` — for executing it

Report:
> "Skills loaded. Ready to start {KEY}."

---

## Step 5 — Execute Story

Follow the checkpoint model from HOW-THIS-WORKS.md:

**If 🔴 HIGH story:**
1. Read `issues/{lane}/{KEY}-*.md`
2. Write `implementation_plans/{lane}/{KEY}-implementation-plan.md`
3. 🔴 STOP — present plan to supervisor for review before proceeding
4. On approval: execute plan
5. Run `local-code-review` — fix all BLOCKER/ISSUE findings (max 3 iterations; escalate if not clean)
6. Lint / tests pass
7. Write `task-completions/{KEY}-COMPLETION-SUMMARY.md`
8. Commit: `{type}({lane}): {description} ({KEY})`
9. Check off story in runbook

**If 🟡 MEDIUM or 🟢 LOW story (batch):**
1. Identify all stories in the batch (from runbook)
2. Read all issue files
3. Write all implementation plans
4. 🟡 STOP — present all plans to supervisor for review before proceeding
5. On approval: execute each plan in sequence
6. Run `local-code-review` after each plan — fix all BLOCKER/ISSUE findings (max 3 iterations; escalate if not clean)
7. Lint / tests pass for each
8. Write completion summary for each
9. Commit each
10. Check off all stories in runbook

---

## Step 6 — Wave Gate

After all stories in the wave are checked off:

1. Verify wave gate checklist from runbook:
   - [ ] All stories in this wave checked off
   - [ ] All PRs merged (or branch ready)
   - [ ] All completion summaries written
2. 🟢 WAVE GATE — notify supervisor:
   > "Wave {N} gate closed for lane {lane}. Dependent lanes: {list or 'none'}."
3. Check off wave gate in runbook

---

## Resume Protocol

If resuming a session mid-way:

1. Run Steps 1–3 (context + active story identification)
2. Check story state:
   - Issue file exists, no plan yet → start at plan writing
   - Plan exists, not executed → present plan for checkpoint approval, then execute
   - Plan executed, no completion summary → write completion summary
   - Completion summary written, not committed → commit
   - All done, not checked off in runbook → check off, check wave gate

Report current state before proceeding:
> "Resuming. {KEY} is in progress. State: {plan written / awaiting checkpoint / executing / summary needed}. Continuing from there."

---

## Human Escalation Triggers

Stop immediately and present to supervisor when ANY of these occur. Do not proceed without explicit human instruction.

| Trigger | What to report |
|---------|---------------|
| Scope change discovered during implementation | "Scope change found: {description}. Out of current issue scope. Proceed, defer, or open new issue?" |
| Architectural decision required | "Architecture decision needed: {options with trade-offs}. Cannot proceed without direction." |
| Security-sensitive change identified | "Security-sensitive change found: {description}. Requires human sign-off before proceeding." |
| Product behavior change (unplanned) | "This change alters observable product behavior: {description}. Not in AC. Human approval required." |
| Local review loop hit 3 iterations with unresolved BLOCKER/ISSUE | "local-code-review exhausted 3 iterations. Unresolved findings: {list}. Cannot auto-resolve." |
| Agent disagreement between plan and review findings | "Plan specifies X but review flags X as BLOCKER/ISSUE. Conflict requires human resolution." |
| Cross-lane dependency check fails | "Lane {name} Wave {N} gate is not closed. Cannot start dependent story." |

**Do not rationalize past these triggers.** If in doubt → escalate.

---

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Starting implementation without reading HOW-THIS-WORKS.md | Always Step 1 first |
| Picking a story without checking wave gate of previous wave | Check wave gate before identifying active story |
| Starting a HIGH story without stopping at checkpoint | 🔴 is a hard stop — present plan, wait for approval |
| Executing batch without presenting all plans first | 🟡 covers the whole batch — write all plans, then stop |
| Forgetting to check off story in runbook | Final step of every story — mark it done |
| Skipping cross-lane dependency check | Step 3 is mandatory — blocked stories cannot start |
| Starting a story without claiming it first | Claim before planning — prevents parallel agent collision |
| Adding a second `🔒 Claimed:` row | Fill the generated row instead; duplicate claim rows make ownership ambiguous. |
| Not checking `git worktree list` before creating worktree | Another session may already have the branch checked out |
| Naming branch with prefix/suffix (e.g. `feature/ARC-123`) | Branch name MUST be exactly `<issueid>` — no prefix, no suffix, no description |
| Committing runbook/plan artifacts to feature branch | Runbook checkoffs, implementation plans, completion summaries → commit to `main` |
