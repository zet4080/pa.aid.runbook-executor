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
If not, write a decision document to request the selection:

1. Write `decisions/{session-date}-lane-selection.md` using the `_template.md` schema:
   - `key`: `lane-selection`
   - `story`: `Select lane for this execution session`
   - `status: pending`
   - List each available lane from the workorder Feature Lane Overview table as an option in the body
2. Commit and push the decision document:
   ```bash
   git add decisions/{session-date}-lane-selection.md
   git commit -m "docs(session): add lane selection decision"
   git push
   ```
3. 🔴 STOP at checkpoint. Surface:
   > "Lane not specified. Decision document written: `decisions/{session-date}-lane-selection.md`. Set `status: answered` and `selected_approach: {lane-name}`, then re-trigger the session."

On resume: read the answered document, extract `selected_approach`, use that as the lane. Do not ask interactively.

### Which wave?

In the workorder, find the lane's current wave:
- Wave is active if previous wave gate is closed (all boxes checked) or this is Wave 1
- If previous wave gate is NOT closed, stop:
  > "Wave {N-1} gate is not closed for lane {lane}. Complete Wave {N-1} before starting Wave {N}. Outstanding items: {list unchecked boxes}"

### Which story?

Call `runbook_find_next_story(runbook_path="docs/plans/runbook-{lane}.md")` — this returns the first unchecked AND unclaimed story in the active wave, or null if none remain. A story is claimed if it has a checked `- [x] 🔒 Claimed:` sub-item. Skip claimed stories — another agent or session owns them.

Report:
> "Next story: **{KEY}** — {summary}
> Risk: {🔴 HIGH / 🟡 MEDIUM / 🟢 LOW}
> Checkpoint: {individual plan checkpoint / batch with {KEY2}, {KEY3}}"

### Claim the Story

**Before writing any plan or code**, mark the story or batch as claimed in the runbook:

Call `runbook_claim_story(runbook_path="docs/plans/runbook-{lane}.md", story_key="{KEY}", lane="{lane}", repo_path="/path/to/planning/repo")` — this updates the claimed checkbox, stages, commits, and pushes the change.

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
3. The conductor detects the new plan file via git status, opens a Bitbucket PR (branch `plan/{lane}/{KEY}`, PR titled `Plan review: {KEY} — {story title}`), and pauses the lane automatically (ARC-1371). No manual commit/push is needed — the agent's step is complete once the file is written to disk.
4. 🔴 The lane is now paused — the supervisor reviews the plan on the PR and merges to approve
5. On PR merge: the conductor detects it, resumes the lane, and re-dispatches the agent to execute the plan (now on `main`)
6. Run `local-code-review` — fix all BLOCKER/ISSUE findings (max 3 iterations; escalate if not clean)
7. Lint / tests pass
8. Write `task-completions/{KEY}-COMPLETION-SUMMARY.md`
9. Commit: `{type}({lane}): {description} ({KEY})`
10. Check off story in runbook

**If 🟡 MEDIUM or 🟢 LOW story (batch):**
1. Identify all stories in the batch (from runbook)
2. Read all issue files
3. Write all implementation plans
4. Each plan file triggers its own conductor-managed PR approval gate (ARC-1371) as it is written — no manual commit/push needed
5. 🟡 Each plan pauses at its own checkpoint until its PR is merged by the supervisor
6. On approval: execute each plan in sequence
7. Run `local-code-review` after each plan — fix all BLOCKER/ISSUE findings (max 3 iterations; escalate if not clean)
8. Lint / tests pass for each
9. Write completion summary for each
10. Commit each
11. Check off all stories in runbook

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

Stop immediately and write a decision document when ANY of these occur. Do not proceed without an answered decision document or explicit human instruction.

| Trigger | Decision document action |
|---------|--------------------------|
| Scope change discovered during implementation | Write `decisions/{KEY}-scope-change.md`: describe the out-of-scope change, list options (proceed / defer / open new issue), include recommendation. Commit+push. 🔴 Stop at checkpoint. |
| Architectural decision required | Write `decisions/{KEY}-architecture-decision.md`: describe the decision point, list 2–3 options with trade-offs, include recommendation. Commit+push. 🔴 Stop at checkpoint. |
| Security-sensitive change identified | Write `decisions/{KEY}-security-review.md`: describe the sensitive change, list options (proceed with mitigations / defer / reject), note security implications. Commit+push. 🔴 Stop at checkpoint. |
| Product behavior change (unplanned) | Write `decisions/{KEY}-behavior-change.md`: describe the observable behavior change, list options (accept / revert / open new issue), note AC impact. Commit+push. 🔴 Stop at checkpoint. |
| Local review loop hit 3 iterations with unresolved BLOCKER/ISSUE | Stop at checkpoint. Report: "local-code-review exhausted 3 iterations. Unresolved findings: {list}. Cannot auto-resolve." No decision document needed — this is an operational block. |
| Agent disagreement between plan and review findings | Stop at checkpoint. Report: "Plan specifies X but review flags X as BLOCKER/ISSUE. Conflict requires human resolution." No decision document needed — this is an operational block. |
| Cross-lane dependency check fails | Stop at checkpoint. Report: "Lane {name} Wave {N} gate is not closed. Cannot start dependent story." No decision document needed — this is an operational block. |

On resume after an answered decision document: read the document, verify `status: answered`, extract `selected_approach`, and proceed accordingly.

**Do not rationalize past these triggers.** If in doubt → write a decision document and stop.

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
