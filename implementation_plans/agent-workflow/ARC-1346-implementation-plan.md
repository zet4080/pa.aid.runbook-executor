# ARC-1346 Introduce decision document pattern for async human decisions - Implementation Plan

**Issue:** `issues/agent-workflow/ARC-1346-decision-document-pattern.md`
**Completion Summary:** `task-completions/ARC-1346-COMPLETION-SUMMARY.md` (not yet written)
**Approach:** Option 1 — Repo-root `decisions/` folder + frontmatter `status` field + in-place skill edit
**Owner:** build
**Date:** 2026-07-01

---

## Scope & Alignment

This plan introduces the decision document pattern to the planning repo and bakes it into the `write-implementation-plan` skill. No application code is involved; all changes are planning-repo artifacts and a shared skill file.

| Acceptance Criterion | Covered by |
|---|---|
| AC 1 — Agent writes `decisions/{KEY}-approach-decision.md`, commits+pushes, stops at checkpoint | Steps 1–2 (folder + template), Step 4 (Phase 1 skill update) |
| AC 2 — On resume, agent reads answered document, extracts `selected_approach`, proceeds without interactive prompt | Step 4 (resume protocol in Phase 1 skill update) |
| AC 3 — Decision document moved to `done/decisions/` after plan is written | Step 3 (`done/decisions/` folder), Step 5 (Phase 2 skill update) |

---

## Assumptions & Dependencies

- The `decisions/` folder lives at the root of `pa.aid.runbook-executor` alongside `issues/`, `implementation_plans/`, etc.
- Frontmatter in decision documents uses YAML-style key: value lines at the top of the file, consistent with how issue files embed metadata tables today.
- The `write-implementation-plan` SKILL.md at `/home/zimmermann/.config/opencode/skills/write-implementation-plan/SKILL.md` is the single file to update; no other skill is in scope for this story (ARC-1347 handles the broader sweep).
- `done/decisions/` follows the same archival convention as `done/issues/`, `done/implementation_plans/`, and `done/task-completions/`.
- A `.gitkeep` in each empty folder is sufficient to persist it in git; no additional README is required.
- The agent executing this plan has write access to both the planning repo and the skill file path.

---

## Implementation Steps

### Step 1: Create `decisions/` folder with `.gitkeep` and decision document template

**Files:**
- `decisions/.gitkeep` (new)
- `decisions/_template.md` (new)

**Action:** Create the `decisions/` directory at the repo root. Add `.gitkeep` to make the empty folder trackable. Add `_template.md` defining the canonical structure for decision documents. The template must include:

- A frontmatter block (YAML key-value lines, not a table) with these fields:
  - `key:` — Jira issue key, e.g. `ARC-1346`
  - `story:` — one-line human-readable title
  - `status:` — one of `pending` or `answered`
  - `selected_approach:` — left blank until the human fills it in; expected value is `Option N` (e.g. `Option 1`)
  - `decided_by:` — name or handle of the person who answered
  - `decided_at:` — ISO date when answered
- A body with four sections: `## Decision Context`, `## Options`, `## Recommendation`, `## Notes`
- `## Options` uses the same approach format already defined in the skill (Option N / How / Pro / Con)
- Comments inside the template (HTML comment blocks) instruct the human on how to fill it in

**Verification:** `ls decisions/` returns both `.gitkeep` and `_template.md`; the template file opens with a valid frontmatter block containing all six fields.

---

### Step 2: Create `done/decisions/` folder with `.gitkeep`

**Files:**
- `done/decisions/.gitkeep` (new)

**Action:** Create `done/decisions/` under the existing `done/` directory. Add `.gitkeep`. No other content needed.

**Verification:** `ls done/` shows a `decisions/` entry alongside the existing `issues/`, `implementation_plans/`, and `task-completions/` directories.

---

### Step 3: Update `AGENTS.md` — add `decisions/` to the repo layout table

**Files:**
- `AGENTS.md`

**Action:** In the "What This Repo Contains" table, insert a new row for `decisions/` after the `issues/` row:

```
| `decisions/` | Pending and answered decision documents |
```

Also add a row for `done/decisions/` context (already covered by the `done/` row — no separate row needed; the `done/` row description should mention decisions). Update the `done/` row description from "Archived completed items" to "Archived completed items (issues, plans, completions, decisions)".

**Verification:** `AGENTS.md` repo layout table has a `decisions/` row and the `done/` row explicitly names decisions.

---

### Step 4: Update `write-implementation-plan/SKILL.md` — Phase 1 decision-document protocol

**Files:**
- `/home/zimmermann/.config/opencode/skills/write-implementation-plan/SKILL.md`

**Action:** Replace the Phase 1 "Process" subsection. The current process says: propose options → STOP — wait for user approval. Replace it with the decision-document protocol as follows.

The new **Process** subsection must describe two sub-paths:

**Path A — No existing decision document (first run):**
1. Check for a file matching `decisions/{KEY}-approach-decision.md` in the planning repo. If absent, proceed with Path A.
2. Write `decisions/{KEY}-approach-decision.md` using the `_template.md` schema: populate `key`, `story`, `status: pending`; leave `selected_approach`, `decided_by`, `decided_at` blank; fill in all options, pros/cons, and the recommendation in the body.
3. Commit and push the decision document to the planning repo main branch using the commit message format `docs({lane}): add {KEY} approach decision`.
4. Stop at a 🔴 checkpoint. Surface the message: "Decision document written and pushed. Review and answer: `decisions/{KEY}-approach-decision.md`. Set `status: answered` and `selected_approach: Option N`, then re-trigger the session."

**Path B — Decision document exists with `status: answered` (resume):**
1. Check for `decisions/{KEY}-approach-decision.md`. If present and frontmatter `status` is `answered`, proceed with Path B.
2. Read the document. Extract `selected_approach` from frontmatter.
3. Proceed directly to Phase 2 using the extracted approach. Do not ask interactively.

The existing approach format block (Option 1 / Option 2 / Option 3 / Recommendation) is retained — it is the format used when populating the decision document body. Move it into this section as the prescribed body format for the Options section of the document.

Remove the line "**Do not write the plan until the user selects an approach.**" — it is superseded by Path B.

**Verification:** The Phase 1 section of the skill file contains the two-path structure (Path A: write+commit+push+stop; Path B: read+extract+proceed) and no longer contains the bare "STOP — wait for user approval" instruction as the sole gate.

---

### Step 5: Update `write-implementation-plan/SKILL.md` — Phase 2 archival step

**Files:**
- `/home/zimmermann/.config/opencode/skills/write-implementation-plan/SKILL.md`

**Action:** In the Phase 2 "Commit and Push" subsection, after the existing git commands for the implementation plan, add an archival step. The archival step must:

1. Check whether `decisions/{KEY}-approach-decision.md` exists in the planning repo.
2. If it exists, move it to `done/decisions/{KEY}-approach-decision.md` using `git mv`.
3. Commit the move with message `docs({lane}): archive {KEY} approach decision`.
4. Push.

Express this as a prose instruction followed by a short command block (≤ 5 lines). The step is conditional — it applies only when a decision document was written for this story.

**Verification:** The Phase 2 section of the skill file contains an archival paragraph referencing `done/decisions/` and a `git mv` command block no longer than 5 lines.

---

### Step 6: Commit and push all planning-repo changes

**Files:** All new and modified files in `pa.aid.runbook-executor`

**Action:** Stage and commit `decisions/.gitkeep`, `decisions/_template.md`, `done/decisions/.gitkeep`, `AGENTS.md`, and this implementation plan file together under the message format `docs(agent-workflow): add ARC-1346 decision document pattern artifacts`. Push to `main`.

**Verification:** `git log --oneline -3` on the planning repo shows the commit; `git status` is clean; remote `main` is up to date.

---

## Testing & Validation

### Scenario 1 — Path A: first run on a HIGH-risk story

Given a HIGH-risk story key (e.g. `ARC-1350`) with no existing decision document, when the agent reaches Phase 1 of `write-implementation-plan`, then:
- File `decisions/ARC-1350-approach-decision.md` is created with `status: pending` and a populated options/recommendation body.
- A git commit exists on `main` containing only that file.
- The agent outputs a 🔴 checkpoint message referencing the exact file path and instructs the human to set `status: answered` and `selected_approach`.
- No implementation plan is written yet.

### Scenario 2 — Path B: resume after human answers

Given `decisions/ARC-1350-approach-decision.md` exists with `status: answered` and `selected_approach: Option 2`, when the agent resumes the session and enters Phase 1:
- The agent reads the file, finds `status: answered`, extracts `selected_approach: Option 2`.
- The agent proceeds directly to Phase 2 without asking any interactive question.
- The implementation plan is written using Option 2 as the recorded approach.

### Scenario 3 — AC 3: decision document archival

Given `decisions/ARC-1350-approach-decision.md` exists with `status: answered` and Phase 2 has completed (plan written and committed):
- The archival step runs `git mv decisions/ARC-1350-approach-decision.md done/decisions/ARC-1350-approach-decision.md`.
- The commit is pushed to `main`.
- `decisions/` no longer contains `ARC-1350-approach-decision.md`; `done/decisions/` does.

### Scenario 4 — Template integrity check

Manually open `decisions/_template.md` and verify:
- All six frontmatter fields are present (`key`, `story`, `status`, `selected_approach`, `decided_by`, `decided_at`).
- The `status` placeholder shows the accepted values (`pending` | `answered`).
- The Options body section matches the approach format prescribed in the skill.

### Scenario 5 — AGENTS.md table consistency

Read `AGENTS.md` and confirm:
- A `decisions/` row is present in the repo layout table.
- The `done/` row references decisions in its description.

---

## Risks & Open Questions

- **Skill file is shared**: `/home/zimmermann/.config/opencode/skills/write-implementation-plan/SKILL.md` is a global file used across all planning sessions. An edit that introduces a syntax error or structural ambiguity will break every future `write-implementation-plan` invocation. Mitigate: edit only the Phase 1 Process and Phase 2 Commit subsections; leave all other sections and the quality checklist intact.
- **Agent may not detect `status: answered` reliably**: If frontmatter parsing is purely string-based (grep for `status: answered`), whitespace or case variations could cause Path A to re-run and overwrite a filled-in document. Mitigate: the skill instruction should specify exact match on the string `status: answered` and caution against overwriting an existing file.
- **`done/decisions/` folder absent on first archival**: If the `done/decisions/` folder does not exist when `git mv` runs, the command will fail. Steps 2 and 6 create the folder with `.gitkeep` before any story using the pattern is executed, so this should not occur — but the skill instruction should note the prerequisite.
- **ARC-1347 dependency**: ARC-1347 will extend this pattern to other skills. The template and folder structure defined here must remain stable. Any schema change to the frontmatter fields after ARC-1347 begins would require re-editing multiple skill files.
