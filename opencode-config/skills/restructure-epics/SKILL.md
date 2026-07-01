---
name: restructure-epics
description: Use when Jira epics need restructuring into feature-driven epics and stories
---

# Skill: restructure-epics

## Purpose

Transform Jira epic organization (layer-based, team-based, phase-based, or ad-hoc) into feature-driven structure: one epic = one user-facing feature, stories = self-contained vertical slices. Stops after Jira and issue files updated.

## When to Use

- Restructuring existing epics into feature-driven layout
- After initial epic creation or when epic organization needs realignment
- Before write-workorder

---

## Overview

Transform any Jira epic organisation (layer-based, team-based, project-phase-based, ad-hoc, or mixed) into a feature-driven structure. Each output epic = one user-facing feature. Each story = a self-contained increment of that feature.

**Stops after Jira is updated and planning repo issue files are written.**

This skill does NOT produce a workorder. It does NOT produce runbooks. It does NOT invoke other skills mid-execution (except loading `create-issue` during Phase 6 to write issue files).

---

## FDD Orientation

- Feature = user-facing capability, not a technical layer
- Stories = vertical slices (touch all layers needed to deliver that slice)
- One epic per feature — not one epic per technical area
- Epics are independent of each other as much as possible
- Cross-feature dependencies are explicit, minimal, and documented
- Every story must be self-contained and independently testable
- **Summary format:** epic = "{PROJ}: {Feature Name}" (noun phrase); story = verb phrase "[action] [result/object]" (e.g. `Display transaction history`, `Export invoice as PDF`)
- **Never use** "As a [role], I want [feature], so that [benefit]" — in summaries, descriptions, or anywhere else

---

## Prerequisites

1. **Jira project key(s)** — provided by user at invocation (supports single key or multiple keys)
2. **Jira MCP access** — read + write permissions required
3. **Git repo access** — skill creates or confirms the planning repo before writing proposal docs

---

## Phase 1 — Ingest

1. Fetch all epics from the Jira project:
   ```
   atlassian_jira_search(jql="issuetype = Epic AND project = {KEY}")
   ```
   For multiple projects:
   ```
   atlassian_jira_search(jql="issuetype = Epic AND project in (KEY1, KEY2)")
   ```

2. For each epic, fetch all child issues:
   ```
   atlassian_jira_search(jql="parent = {EPIC_KEY}")
   ```

3. For each issue, read: summary, description, comments, labels, status, priority

4. Build flat list with schema: `[issue_key, epic_key, summary, description, status]`

---

## Phase 2 — Feature clustering

Analyse the full issue list. For each issue, identify the **user-facing outcome** it contributes to — not its technical layer.

**Analysis rules:**
- Cluster by *what the user can do* after it's done, not *what code is changed*
- Prior organisation (layer, team, component, project phase) is irrelevant — recluster from scratch based on user-facing outcomes
- Issues enabling one end-to-end capability belong in one feature cluster
- Technical plumbing (migrations, infra) belongs with the feature it enables — NOT in a generic "infrastructure" epic
- Exception: shared prerequisites for 3+ features → one "Foundation" epic (Wave 1 only, ≤20% of total issues)
- Existing epics that already map cleanly to a feature: keep them (rename only if summary is misleading)

**Per-cluster output format:**
```
Feature name: {short user-facing name}
Proposed epic summary: {PROJ}: {Feature name}
Rationale: {1-2 sentences why these issues belong together}
Issues: [{KEY1}, {KEY2}, ...]
New? {yes | no — existing epic KEY}
```

---

## Phase 3 — Create Planning Repo

After feature clustering is complete and before writing any proposal document, create or confirm the planning repo.

1. Derive repo name from project/scope: `pa.aid.<topic>` — lowercase, hyphens, no special characters.
2. Propose the repo name and stop for confirmation.
3. If repo already exists, switch to it and verify it is the intended planning repo.
4. If repo does not exist, create it using standard process for your git host, clone it, initialize git, and make an initial commit.
5. Scaffold minimal source-of-truth structure:

```
opencode-config/
  skills/           ← copy from pa.aid.conductor.ts
docs/
  plans/
  specs/
issues/
implementation_plans/
task-completions/
done/
```

Commit: `chore: initialize planning repo`

Do not write proposal docs in `pa.aid.conductor.ts`, a product repo, or a temporary folder.

---

## Phase 4 — Propose epic structure (gate 1)

Write `docs/plans/proposed-epic-structure-{YYYY-MM-DD}.md` using this template:

```markdown
# Proposed Epic Structure — {date}

## Summary
{N} existing epics → {M} feature epics + {F} foundation epics
{K} issues re-parented, {J} issues unchanged

## Feature Epics

### {Feature Name}
**Proposed epic:** {PROJ-NEW or existing PROJ-KEY}
**Rationale:** {why these form a feature}
**Stories ({N}):**
| Key | Summary | Current Epic | Action |
|-----|---------|--------------|--------|
| PROJ-123 | ... | PROJ-10 | re-parent |
| PROJ-456 | ... | PROJ-10 | keep |

## Foundation Epics (shared prerequisites)

### {Foundation Name}
**Rationale:** {which features depend on this and why it cannot belong to one}
**Stories ({N}):**
| Key | Summary | Current Epic | Action |

## Unchanged
Issues not moved: {list with reason}

## Epics to close
Epics that become empty after re-parenting: {list}
```

**STOP. Do not proceed until the user has reviewed and approved this document.**

Human edits the file directly. Agent re-reads the approved structure before Phase 4b.

---

## Phase 4b — Analyse issues and propose changes (gate 2)

After epic structure is approved, analyse every issue within each approved feature epic for quality and coverage.

### Analysis dimensions

**1. Structure** — are required sections present?
- Check for: summary, description, acceptance criteria, scope (in-scope + dependencies), out-of-scope
- Flag any missing section

**2. Content quality** — are existing sections well-written?
- AC uses vague language ("works correctly", "as expected", "is visible") → flag for rewrite
- Scope written as implementation steps ("add field to X") instead of outcomes → flag for rewrite
- Description missing constraints or goal → flag for rewrite
- Out-of-scope empty or generic → flag for rewrite

**3. Coverage** — does the feature have the right stories?
- Identify user-facing increments the feature needs but has no story for → propose new issue
- Identify issues that duplicate each other → propose cancellation of the weaker one
- Identify purely technical issues with no user-facing outcome that belong to no feature → propose cancellation or move to Foundation

### Proposal document

Write `docs/plans/proposed-issue-changes-{YYYY-MM-DD}.md` using this template:

```markdown
# Proposed Issue Changes — {date}

## Summary
{N} issues to rewrite · {M} new issues to create · {K} issues to cancel · {J} issues OK

## Issues to Rewrite
| Key | Feature | Problem | Proposed action |
|-----|---------|---------|-----------------|
| PROJ-123 | Auth | AC vague ("login works") · scope lists steps | Rewrite AC as Given/When/Then; rewrite scope as outcomes |

## New Issues to Create

### {KEY}-new-1: {Proposed summary — verb phrase: "[action] [result/object]"}
**Feature:** {feature name} / {epic key}
**Why needed:** {gap this fills — what user capability is missing without it}
**Proposed description:** {goal + constraints, 2-3 sentences}
**Proposed AC:**
1. Given {precondition}, when {action}, then {observable result}.
**Proposed scope:**
- In Scope: {outcome bullets}
- Out of Scope: {explicit exclusions}

## Issues to Cancel
| Key | Feature | Reason |
|-----|---------|--------|
| PROJ-789 | — | Duplicate of PROJ-456 — same outcome, weaker description |

## Issues OK
{List of keys confirmed as good — no changes needed}
```

**STOP. Do not proceed until the user has reviewed and approved this document.**

Human edits the file directly. Agent re-reads before Phase 5.

---

## Phase 5 — Execute all Jira changes

Read both approved documents (`proposed-epic-structure-{date}.md` and `proposed-issue-changes-{date}.md`) before executing.

Execute in this exact order (dependencies flow top to bottom):

1. **Create new epics** — `atlassian_jira_create_issue(project_key, summary, issue_type="Epic")`
2. **Re-parent issues** — `atlassian_jira_update_issue(issue_key, fields={"parent": new_epic_key})`
   - Note: if API rejects re-parenting for issues already assigned to an epic, use `atlassian_jira_link_to_epic` as fallback
3. **Update epic summaries** — where proposed summary differs from current
4. **Close empty epics** — transition to Done with comment `Restructured: issues moved to {KEY}`
5. **Rewrite issues** — update description, AC, scope per proposals (structure + content fixes)
6. **Create new issues** — `atlassian_jira_create_issue(...)` with full proposed content
7. **Cancel issues** — transition to Won't Do / Cancelled (project-specific — check available transitions first using `atlassian_jira_get_transitions`) with comment stating reason

Log every action (key → action → result) to `docs/plans/restructure-log-{date}.md`.

Commit: `docs: add restructure log after Jira epic and issue restructure`

---

## Phase 6 — Write Issue Files

The planning repo already exists from Phase 3. Ensure directories:
```
opencode-config/
  skills/           ← copy from pa.aid.conductor.ts
docs/
  plans/
  specs/
issues/
  {feature-name}/   ← one folder per active feature epic
```

**Load the `create-issue` skill** (`opencode-config/pa.aid.config.md/skills/create-issue/SKILL.md`) at the start of Phase 6 before writing any issue file. Use its template and quality checklist for every file.

Write issue files for every active story:
- Path: `issues/{feature-name}/{KEY}-{kebab-title}.md`
- Content: pull from Jira post-execution (already rewritten/complete at this point)
- New issues created in Phase 5 get their full file written here
- **Exclude cancelled issues** — do not write files for issues transitioned to Cancelled
- **Do NOT add ⚠️ flags** — Phase 4b ensures all AC is resolved before reaching this phase; no flags needed

Do not overwrite existing issue files — add missing ones only. Note: if `issues/` already contains stale folders from a prior structure, clean them up manually.

Commit: `docs: scaffold repo and write issue files after epic restructure`

---

## Quality Checks

### After Phase 2 (clustering)
- [ ] Every issue assigned to exactly one cluster
- [ ] No cluster is purely technical (e.g. "database layer") — must be user-facing
- [ ] Foundation epic contains only genuine shared prerequisites
- [ ] Foundation epic has ≤20% of total issues

### After Phase 3 (planning repo)
- [ ] Planning repo exists before proposal docs are written
- [ ] Current working directory is the planning repo
- [ ] Minimal source-of-truth structure exists

### After Phase 4 (epic proposal doc)
- [ ] Every existing issue appears in exactly one proposed epic
- [ ] Every existing epic either maps to a proposed epic or is listed for closure
- [ ] Rationale is written in user-facing language, not technical

### After Phase 4b (issue proposal doc)
- [ ] Every issue in every approved epic is covered (rewrite / new / cancel / OK)
- [ ] Every new issue has full proposed content (description, AC, scope, out-of-scope)
- [ ] Every cancellation has an explicit reason
- [ ] No feature has zero stories after cancellations

### After Phase 5 (Jira execution)
- [ ] Restructure log lists every action with result
- [ ] No issue left orphaned (parentless)
- [ ] Empty epics closed with comment
- [ ] All rewritten issues updated in Jira
- [ ] All new issues created and parented to correct epic
- [ ] All cancelled issues transitioned with comment

### After Phase 6 (issue files)
- [ ] One folder per active feature epic under `issues/`
- [ ] Every active issue has a file (no cancelled issues)
- [ ] No ⚠️ flags in issue files — all AC resolved before reaching this phase

---

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Cluster named after a technical layer ("API layer") | Rename to the user capability it enables |
| Foundation epic growing large | Split: only genuinely shared prerequisites belong there |
| Executing Jira before human approves epic structure | Hard stop at Phase 4 gate — do not proceed without explicit approval |
| Executing Jira before human approves issue changes | Hard stop at Phase 4b gate — do not proceed without explicit approval |
| Writing issue files before Jira is updated | Phase 6 reads from updated Jira state — execute Phase 5 first |
| Repo already exists but has old structure | Do not overwrite existing issue files — add missing ones only |
| New issue proposed without full AC | Phase 4b requires complete proposed content (description, AC, scope, out-of-scope) for all new issues |
| Cancelling issue without comment in Jira | Always add reason comment before transitioning to Cancelled |
| Writing proposal docs before creating repo | Create/enter the planning repo first; it is the source of truth |
| Story summary uses "As a…" user story format | Rewrite as FDD verb phrase: "[action] [result/object]" |
