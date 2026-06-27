---
name: sync-runbook
description: Use when reconciling runbook/workorder state to actual work done — when an agent forgot to check off steps, or a human did manual work outside the agent workflow
---
# sync-runbook

## Purpose
Reconcile `docs/plans/runbook-*.md` and `docs/plans/workorder.md` to the actual current state of work. Use this when:
- An agent completed work but forgot to check off runbook steps
- A human did implementation work manually (outside agent workflow)
- The runbook shows steps unchecked that are clearly already done
- You need to audit what's really done vs. what the runbook thinks is done

## When to invoke
Load this skill when human supervisor says any of:
- "sync runbook"
- "update runbook"
- "runbook is out of date"
- "I did some work manually"
- "agent forgot to check off"
- "sync workorder"
- "reconcile runbook"

---

## Workflow

### Step 1 — Gather Current State
1. Read all runbook files: `docs/plans/runbook-*.md` (one per lane)
2. Read `docs/plans/workorder.md`
3. List all stories/tasks with their current checkbox state:
   - `[ ]` = unchecked (runbook thinks not done)
   - `[x]` = checked (runbook thinks done)

### Step 2 — Human Supervisor Input
Ask the human supervisor:
> "Which stories/tasks are actually done? Give me issue IDs or descriptions, and I'll check them off."

Accept input in any format:
- "ABC-123 is done"
- "The first three stories in lane 1 are done"
- "Everything up to ABC-456 is complete"
- A list of issue IDs

### Step 3 — Verify Artifacts Exist
For each story the supervisor marks as done, verify these artifacts exist:
- Issue file: `issues/{EPIC}/{TRACKER-ID}-*.md` — must exist
- Implementation plan: `implementation_plans/{EPIC}/{TRACKER-ID}-implementation-plan.md` — must exist OR note it's missing
- Completion summary: `task-completions/{EPIC}/{TRACKER-ID}-*-COMPLETION-SUMMARY.md` — must exist OR create a stub

If a completion summary is missing, create a minimal stub:
```markdown
# {TRACKER-ID} Completion Summary

## Status
✅ Completed (manual sync — {DATE})

## Notes
Marked complete by human supervisor sync. Implementation artifacts may be partial.
```

### Step 4 — Verify Code Evidence (Optional but Recommended)
For each story marked done, optionally check:
- Does a git commit exist referencing the issue ID? (`git log --oneline --all | grep <TRACKER-ID>`)
- Is the feature branch merged or does it no longer exist as active work?
- Are relevant files changed since the story's start date?

Report any stories where "done" is claimed but no code evidence exists — flag these as ⚠️ UNVERIFIED.

### Step 5 — Update Runbook Checkboxes
For each verified-done story:
1. Find the corresponding line in the runbook: `- [ ] <description> (<TRACKER-ID>)` or similar
2. Change to: `- [x] <description> (<TRACKER-ID>)`
3. Also update the `🔒 Claimed:` row if present — set to `✅ Done`
4. If a wave gate is now fully complete (all stories in wave checked), check off the wave gate too

### Step 6 — Update Workorder (if applicable)
If workorder.md tracks story status, update it to reflect completed stories.

### Step 7 — Commit to Main
Commit ALL runbook/workorder changes directly to `main` (never a feature branch):
```
chore: sync runbook to actual work state

Manually reconciled runbook checkboxes after human supervisor review.
Stories marked done: {list of IDs}
```

### Step 8 — Report
Output a summary table:
| Story | Was | Now | Artifacts | Code Evidence |
|-------|-----|-----|-----------|---------------|
| ABC-123 | [ ] | [x] | ✅ all present | ✅ commit found |
| ABC-124 | [ ] | [x] | ⚠️ no completion summary (stub created) | ⚠️ no commit found |

---

## Rules
- **Never uncheck a box** — this skill only marks things done, never reverts to undone. If you need to revert, do it manually.
- **Always commit to main** — runbook is a planning artifact, not code. No feature branches.
- **Flag unverified** — if supervisor says "done" but no evidence exists, mark `⚠️ UNVERIFIED` in both the runbook comment and the report.
- **Preserve runbook structure** — only change checkbox state, do not reformat or rewrite runbook content.
- **Create missing completion summary stubs** — a stub is better than silence.
