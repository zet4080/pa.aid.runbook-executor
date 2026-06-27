---
name: add-story-to-epic
description: Use when adding one new Jira story or issue to an existing epic, especially when a planning repo, workorder, or runbooks may already exist
---

# Add Story To Epic

## Purpose

Add one story to existing epic as atomic planning change: Jira, issue file, workorder, and runbooks stay consistent. Prevents optional-Jira and overwritten-progress failures.

## When to Use

- Adding story or issue to existing epic
- Creating Jira story under epic when planning repo may exist
- Missing story found after workorder or runbooks already exist
- Not for greenfield epic creation (use create-epics) or broad epic cleanup (use restructure-epics)

---

## Overview

Add one story to an existing epic as one atomic planning change: Jira, issue file, workorder, and runbooks must stay consistent.

Baseline failures this prevents: treating Jira creation as optional, treating planning repo updates as optional, and overwriting active runbook progress.

## When To Use

- User asks to add a story, issue, or task to an existing epic
- User asks to create a Jira story under an epic and a planning repo may exist
- A missing story is found after `workorder.md` or lane runbooks already exist

Do not use for greenfield epic creation; use `create-epics`. Do not use for broad epic cleanup; use `restructure-epics`.

## Required Skills

Load `create-issue` before drafting the story. Use its sections and quality checklist.

If runbooks must be regenerated from a changed workorder and no execution progress exists, load `generate-lane-runbooks` after updating `workorder.md`.

## Process

1. **Orient**
   - Confirm epic key and planning repo path. If repo is unclear, ask.
   - Read Jira epic and existing child stories.
   - Read planning artifacts if present: `issues/`, `docs/plans/workorder.md`, `docs/plans/runbook-*.md`, `docs/plans/HOW-THIS-WORKS.md`.

2. **Draft story**
   - Use `create-issue` format: summary, description, scope, dependencies, Given/When/Then AC, out-of-scope.
   - If story content is too thin for observable AC, ask for missing requirement before Jira write.

3. **Impact plan**
   - Identify target lane from epic key or `issues/{feature}/`.
   - Classify risk: 🔴 HIGH / 🟡 MEDIUM / 🟢 LOW using `write-workorder` criteria.
   - Decide insertion point: default = end of lane; if user says "next", insert before first unchecked story only if dependencies allow.
   - Determine workorder changes: story table, estimates, supervision total, intra-lane deps, cross-lane deps, on-hold items.
   - Determine runbook changes: affected lane runbook plus any dependent runbooks needing new gate checks.

4. **Approval gate**
   - Present story draft and impact plan.
   - Stop until user approves before Jira write or schedule/runbook changes.

5. **Execute Jira**
   - Create Story in same project as epic, parented to epic.
   - Prefer `atlassian_jira_create_issue(..., issue_type="Story", additional_fields={"parent": "EPIC-KEY"})`.
   - If Jira rejects parent field, create story then call `atlassian_jira_link_to_epic`.
   - Use returned story key everywhere. Do not keep `TBD` in committed files.

6. **Update issue file**
   - Write `issues/{feature}/{STORY-KEY}-{kebab-title}.md`.
   - Include Parent Epic metadata if surrounding issue files do.

7. **Update workorder if present**
   - If `docs/plans/workorder.md` does not exist, stop after issue file and report: next step is `write-workorder`.
   - If it exists, patch it manually. Do not invoke `write-workorder`; that skill is for first creation only.
   - Verify every story still appears exactly once.

8. **Update runbooks if present**
   - If no `docs/plans/runbook-*.md` files exist, report: next step is `generate-lane-runbooks`.
   - Before editing runbooks, snapshot checked boxes and in-progress notes.
   - If no boxes are checked, regenerate runbooks from updated workorder.
   - If any boxes are checked, update surgically or regenerate then restore progress. Never lose checked state.
   - New story starts unchecked and includes an unchecked `🔒 Claimed:` row. Do not insert it before completed checked stories.

9. **Verify**
   - Jira story parent = target epic.
   - Issue file exists and passes `create-issue` checklist.
   - Workorder story count, estimates, risk table, dependencies, supervision totals updated.
   - Runbook contains story exactly once and checkpoint summary counts match body.
   - Existing checked boxes remain unchanged.

## Active Execution Rules

If execution already started, runbook progress is source of truth for done/in-progress state.

- Completed stories stay checked.
- New story cannot invalidate a closed wave without explicit user approval.
- If new story belongs in a closed wave, add it to the next open wave unless user explicitly reopens the wave.
- If user requests "make it next", place it before the current first unchecked story only after dependency check.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| "User only asked for Jira" | Still check for planning repo; keep artifacts consistent if one exists. |
| "No Jira unless explicit" | Adding to an epic means Jira parent relationship unless user says local draft only. |
| Running `write-workorder` over existing workorder | Patch existing workorder; `write-workorder` is initial creation only. |
| Blind runbook regeneration | Preserve checked boxes and in-progress notes. |
| Story file uses `TBD` key | Create Jira first after approval; use returned key. |
| New story added to issue file only | Also update workorder and runbooks when present. |
