---
name: generate-lane-runbooks
description: "Use when a workorder exists and you need executable agent runbooks — one per feature lane — with checkpointed workflows and resume capability."
---

# Skill: generate-lane-runbooks

## Purpose

Generate one executable agent runbook per feature lane from workorder. Each runbook is standalone, checkpointed, resumable. Re-invokable whenever workorder changes.

## When to Use

- Workorder exists and executable runbooks needed
- After write-workorder completes
- Before start-execution-session
- Anytime workorder changes (new stories, risk reclassified, wave assignments updated)

---

## Overview

Generate one executable agent runbook per feature lane from `docs/plans/workorder.md`. Each runbook is standalone, checkpointed, and resumable — an agent can execute it with no other context.

- This skill **does not modify** `workorder.md` or any issue files.
- **FDD orientation:** One runbook = one feature lane = one epic. Each story is a self-contained increment: plan → implement → verify → commit. The feature is shippable when all stories in the runbook are checked off.
- **Re-invokable:** Call `generate-lane-runbooks` again whenever `workorder.md` changes (new stories added, risk reclassified, wave assignments updated). Regenerate runbooks whenever the workorder changes.

---

## Prerequisites

1. **`docs/plans/workorder.md`** — must contain:
   - Feature Lane Overview table (lane names, epic keys, wave assignments)
   - Wave sections with story tables (KEY, summary, risk classification: 🔴 HIGH / 🟡 MEDIUM / 🟢 LOW)
   - Cross-Lane Dependencies table (which lanes gate which)
   - (This is the output of the `write-workorder` skill.)

2. **`issues/{feature-name}/`** — one directory per lane, containing one issue file per story. Each issue file must have either:
   - A description body whose first line provides a usable story summary, OR
   - A `summary:` frontmatter field (fallback when description body is sparse).

3. **`execute-implementation-plan` and `write-implementation-plan` skills** accessible in the repo (listed in every generated runbook header so the executing agent loads them at start).

---

## Output

One file per lane:
```
docs/plans/runbook-{feature-name}.md
```

- **One file per lane** in the Feature Lane Overview table.
- **`{feature-name}` normalization rule:** take the lane name from workorder, lowercase it, replace spaces with hyphens, remove any character that is not alphanumeric or a hyphen. Example: `"User Auth"` → `user-auth`.

---

## Runbook structure

Every generated runbook uses this template. Do not omit any block.

### Header block

```markdown
# Runbook: {Feature Name} — {Epic Key}

> Load the `execute-implementation-plan` skill before starting. 
> Stop at every checkpoint symbol (🔴 🟡 🟢). 
> Resume: find first unchecked box and continue.

**Feature goal:** {one sentence — what the user can do when this lane is complete} 
**Epic:** {KEY} 
**Lane:** {feature-name} 
**Wave(s):** {N, M — waves this lane participates in} 
**Repos:** {list of sibling repos this lane touches} 
**Skills to load at start:** execute-implementation-plan, write-implementation-plan 
**Depends on:** {other lane(s) or foundation stories that must complete first — explicit keys, or "none"}
```

### Checkpoint model

```markdown
| Symbol | Type | When |
|--------|------|------|
| 🔴 | Individual plan checkpoint | Before executing any HIGH story |
| 🟡 | Batch plan checkpoint | Before executing a batch of MEDIUM/LOW stories |
| 🟢 | Wave gate | After all stories in a wave close |
```

**HIGH story workflow (9 steps):**
```markdown
1. 🔒 Fill generated claim row in runbook + commit
2. Read issue file
3. Write implementation plan
4. 🔴 INDIVIDUAL PLAN CHECKPOINT — human reviews plan before execution
5. Execute plan
6. Run `local-code-review` — agent self-review with automated lint/tests/type-check
7. 🔴 IMPLEMENTATION REVIEW CHECKPOINT — human reviews implementation before archiving
8. Lint / tests pass (final verification after addressing review comments)
9. Write completion summary
10. Commit + ☑
```

**MEDIUM/LOW batch workflow (9 steps):**
```markdown
1. 🔒 Fill generated claim row in runbook + commit
2. Read all issue files in batch
3. Write all implementation plans
4. 🟡 CHECKPOINT — human reviews all plans
5. Execute all plans
6. Run `local-code-review` per plan — all BLOCKER/ISSUE resolved
7. Lint / tests pass for all
8. Write all completion summaries
9. Commit all + ☑ all
```

### Wave section template

```markdown
## Wave N — {Milestone Name}

> Gate: {what must be true before this wave's stories start — e.g. "Foundation epic Wave 1 closed"}

### 🔴 HIGH stories

- [ ] **{KEY}** — {summary}
  - [ ] 🔒 Claimed: _(fill in: lane / YYYY-MM-DD HH:MM before starting)_
  - [ ] Read `issues/{feature}/{KEY}-{title}.md`
  - [ ] Write `implementation_plans/{feature}/{KEY}-implementation-plan.md`
  - [ ] 🔴 INDIVIDUAL PLAN CHECKPOINT
  - [ ] Execute plan
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] 🔴 IMPLEMENTATION REVIEW CHECKPOINT
  - [ ] Lint / tests pass
  - [ ] Write `task-completions/{KEY}-COMPLETION-SUMMARY.md`
  - [ ] Commit: `feat({feature}): {short description}`

### 🟡/🟢 Batch — {group description}

- [ ] 🔒 Claimed: _(fill in: lane / YYYY-MM-DD HH:MM before starting)_
- [ ] Read all issues: {KEY1}, {KEY2}, {KEY3}
- [ ] Write all plans
- [ ] 🟡 BATCH PLAN CHECKPOINT
- [ ] Execute {KEY1}
  - [ ] Run `local-code-review` — all BLOCKER/ISSUE resolved
  - [ ] Lint / tests pass
  - [ ] Write completion summary
  - [ ] Commit
- [ ] Execute {KEY2}
  - [ ] ...
- [ ] ☑ all complete

### Wave N gate
- [ ] All stories in this wave checked off
- [ ] All PRs merged
- [ ] All completion summaries written
- [ ] 🟢 WAVE GATE — notify supervisor of dependent lanes: {list or "none"}
```

Claim rows are generated, not added during execution. The executing agent checks and fills the existing `🔒 Claimed:` row; a missing row means the runbook is malformed and must be repaired or regenerated before execution starts.

### On-Hold Register

```markdown
| Story | Blocked by | Last checked | Action if unblocked |
|-------|-----------|--------------|---------------------|
```

### Checkpoint summary table

```markdown
| Wave | HIGH checkpoints | Batch checkpoints | Wave gates | Total |
|------|-----------------|-------------------|------------|-------|
```

---

## Generation process

Execute these 9 steps in order for every invocation:

1. **Read Feature Lane Overview table** from `workorder.md` — extract: lane names, epic keys, wave assignments, cross-lane dependencies.

2. **For each lane:** read story list per wave; record risk classification (🔴 HIGH / 🟡 MEDIUM / 🟢 LOW) per story from workorder story tables.

3. **Read each issue file** in `issues/{feature-name}/` — extract story summary from first line of description body. If description body is empty or sparse, fall back to the `summary:` frontmatter field.

4. **Generate header block:** populate Feature goal (user-facing sentence — "User can do X"), Epic key, Lane, Waves, Repos, Skills, Depends-on from step 1–2 data.

5. **For each wave this lane participates in:** generate wave section:
   - 🔴 HIGH stories: one individual workflow block per story (9 steps), including exactly one unchecked `🔒 Claimed:` row as the first sub-item.
   - 🟡/🟢 MEDIUM/LOW stories: group into batches, **max 5 stories per batch**. Each batch block includes exactly one unchecked `🔒 Claimed:` row before any read/plan rows. If the wave has more than 5 MEDIUM/LOW stories, split into two (or more) batches with descriptive group labels.

6. **On-Hold stories:** generate conditional block — "Check {blocker} — if resolved: [full HIGH or batch workflow], if not: skip with note in On-Hold Register".

7. **Cross-lane dependencies** (from Cross-Lane Dependencies table): insert explicit gate check as the first checkbox before the first story that depends on another lane:
   ```
   - [ ] Confirm {Lane} Wave N gate passed
   ```

8. **Generate checkpoint summary table** by counting: HIGH workflow blocks per wave → HIGH checkpoints; batch blocks per wave → batch checkpoints; wave sections → wave gates. Fill the table at the bottom of the runbook.

9. **After writing all runbooks**, commit (runbooks only — ops docs and session log have their own commits):
   ```
git commit -m "docs: generate lane runbooks from workorder"
   ```

---

## After Runbooks — Operational Documentation

After all runbooks are generated, produce three operational documents if they do not already exist.

### docs/plans/HOW-THIS-WORKS.md

Single entry point for all meta-operational information. Required sections:

```
What this repo is
The three planning documents (table)
Roles (coding agent, review agent, supervisor — explicit responsibility lists)
  → What the review agent checks (5-item checklist)
Checkpoint model (canonical — 3-tier table + risk classification)
Agent workflows (batch 9-step + individual 9-step)
Per-story artifact chain (issue → plan → code → summary)
How to start a session (first time + resume)
How to review an implementation plan (4-item checklist)
How to review a wave close (5-item checklist)
Failure and recovery (agent fails mid-story / story blocked / wave fails gate / emergency stop)
Cross-supervisor coordination (signal table: event → who signals → who receives → what to do)
On-Hold items protocol
Commit message format with example
Authoring skills reference (pointer to opencode-config/pa.aid.config.md/skills/)
```

### AGENTS.md

Agent-oriented context file. Required sections:
```
What this repo contains (folder table)
Sibling repositories (WSL paths, repo names, purposes)
Domain-specific architecture notes (if applicable)
Key configuration patterns (if applicable)
Workflow: issue → plan → execute → complete (4-step table)
Commit message format
Issue authoring conventions (summary of create-issue skill)
Implementation gap protocol
Execution planning documents (pointer to HOW-THIS-WORKS.md as entry point)
MCP endpoints
WSL setup (if applicable)
Key contacts
```

### README.md

Human-oriented overview. Required sections:
```
What is this? (1-2 paragraphs)
Quick navigation (table: section → path → contents)
Project planning documents (pointer to HOW-THIS-WORKS.md + table of 3 planning docs)
Execution model summary (bullets + supervision budget table)
Parallel lane diagram (ASCII)
Issue scope and status (tracker prefixes + status counts)
Sibling repositories
Developer setup (quick sequence + MCP endpoints)
Authoring standards (skills cross-reference, pointer to opencode-config/pa.aid.config.md/skills/)
Key contacts
```

**Rule:** Canonical operational content lives in HOW-THIS-WORKS.md only. AGENTS.md and README.md reference it — they do not duplicate it.

Commit: `"docs: add HOW-THIS-WORKS.md; update AGENTS.md and README.md"`

---

## Session Log

After operational docs, write `docs/plans/session-log-{YYYY-MM-DD}.md`:

```markdown
# Session Log — {date}

**Participants:** {names}
**Outcome:** [one paragraph summary]

## Starting state
## What was built, in order
  [Restructure: issue rewrite — batches, counts, key decisions]
  [Workorder: design decisions, wave structure]
  [Runbooks: checkpoint model, lane assignments]
  [Ops docs: what was missing, what was added]
## Final repository state
  [Files created/changed table]
  [Commit sequence table with hashes]
## Design decisions and rationale
## What was deliberately not done
## How to continue from here
```

Commit: `"docs: add session log"`

---

## Quality checks

After generating all runbooks, verify each one:

- [ ] One runbook per lane in workorder
- [ ] Every story from workorder appears in exactly one runbook
- [ ] Every story/batch execution block has exactly one unchecked `🔒 Claimed:` row when generated.
- [ ] Every 🔴 HIGH story has individual plan + checkpoint
- [ ] Every cross-lane dependency has an explicit gate check
- [ ] On-Hold stories have conditional logic (check/skip structure)
- [ ] Wave gate checkboxes include cross-lane notification where needed
- [ ] Checkpoint summary table count matches actual count in body
- [ ] Feature goal sentence is user-facing ("User can do X"), not a technical task ("Implement X service")

After operational docs:
- [ ] HOW-THIS-WORKS.md covers all 15 topics listed above
- [ ] AGENTS.md references HOW-THIS-WORKS.md as entry point
- [ ] README.md references HOW-THIS-WORKS.md as entry point
- [ ] No operational content duplicated across files (canonical = HOW-THIS-WORKS.md)

After session log:
- [ ] Starting state accurately describes repo before the session
- [ ] All phases documented with key decisions
- [ ] Commit sequence table lists all commits with hashes
- [ ] "How to continue" section gives clear next steps

---

## Common mistakes

| Mistake | Fix |
|---------|-----|
| Feature goal written as technical task (e.g. "Implement X service") | Rewrite as user capability: "User can do X" |
| Cross-lane dependency missing gate check | Add explicit `- [ ] Confirm {Lane} Wave N gate passed` before first dependent story |
| MEDIUM/LOW batch has >5 stories | Split into two batches with descriptive labels |
| On-Hold story has no conditional logic | Add: "Check {blocker} — if resolved: [full workflow], if not: skip with note in On-Hold Register" |
| Checkpoint summary table doesn't match body | Recount HIGH blocks, batch blocks, wave gates after writing — update table |
| Runbook missing a story from workorder | Run coverage check: every workorder row must appear in a runbook |
| HOW-THIS-WORKS.md content duplicated in runbooks | Remove from runbooks, point to HOW-THIS-WORKS.md |
| AGENTS.md has stale content from a different project | Full rewrite — do not append |
