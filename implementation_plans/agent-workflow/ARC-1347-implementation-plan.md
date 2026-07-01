# ARC-1347 Update planning skills for non-interactive checkpoint model - Implementation Plan

**Issue:** `issues/agent-workflow/ARC-1347-update-skills-noninteractive.md`
**Completion Summary:** `task-completions/ARC-1347-COMPLETION-SUMMARY.md` (TBD)
**Approach:** Single approach — replace inline STOP/ask patterns with decision document protocol per ARC-1346 convention
**Owner:** build
**Date:** 2026-07-01

---

## Scope & Alignment

This plan updates three skill files and one planning-repo document to remove inline interactive question patterns and replace them with the decision document protocol introduced in ARC-1346.

| Acceptance Criterion | Covered by |
|---|---|
| AC 1 — Any planning skill with a "STOP — ask human" pattern now writes a decision document, commits it, surfaces the file path as a checkpoint, and does not emit an inline question | Steps 1 (start-execution-session), 2 (execute-implementation-plan) |
| AC 2 — HOW-THIS-WORKS.md Human Escalation Triggers table documents decision document protocol as the response, not interactive questions | Step 3 (HOW-THIS-WORKS.md update) |
| AC 3 — No updated skill contains inline "STOP — answer X, Y, Z" question patterns | Steps 1–3 + Step 4 (self-check scan) |

---

## Assumptions & Dependencies

- ARC-1346 is complete: the `decisions/` folder, `decisions/_template.md`, and `done/decisions/` exist in `pa.aid.runbook-executor`.
- The decision document frontmatter schema (6 fields: `key`, `story`, `status`, `selected_approach`, `decided_by`, `decided_at`) is stable as established in ARC-1346.
- The live skill files at `/home/zimmermann/.config/opencode/skills/` are the authoritative copies. Source copies may exist in `pa.aid.config.md` — that repo is out of scope for this story (will need a follow-on PR there, same as ARC-1346 did).
- `write-completion-summary` has no inline STOP patterns (confirmed by inspection); no update needed.
- `write-implementation-plan` is already updated by ARC-1346; no further changes needed here.
- The `HOW-THIS-WORKS.md` file lives in `pa.aid.runbook-executor/docs/plans/` and is a planning-repo artifact, not a skill file.

---

## Implementation Steps

### Step 1: Update `start-execution-session/SKILL.md` — replace inline STOP/ask patterns with decision document protocol

**Files:**
- `/home/zimmermann/.config/opencode/skills/start-execution-session/SKILL.md`

**Action:** Three sub-changes in this file:

**1a — "Which lane?" ask pattern (line ~33):**
Replace the interactive "ask" fallback:
```
If not, ask:
> "Which lane (feature) are you working on? ..."
```
With a decision document trigger: if the lane is not specified at invocation, write `decisions/{session-date}-lane-selection.md` with the available lanes listed as options, commit+push it, and stop at a 🔴 checkpoint instructing the human to fill in `selected_approach` with the lane name and set `status: answered`.

**1b — Human Escalation Triggers table:**
Replace every inline "What to report" cell that contains an interactive question with a decision document protocol:
- For `Scope change discovered`: instruct agent to write `decisions/{KEY}-scope-change.md` with context and options (proceed / defer / open new issue), commit+push, stop at checkpoint.
- For `Architectural decision required`: instruct agent to write `decisions/{KEY}-architecture-decision.md`, commit+push, stop at checkpoint.
- For `Security-sensitive change identified`: instruct agent to write `decisions/{KEY}-security-review.md`, commit+push, stop at checkpoint.
- For `Product behavior change (unplanned)`: instruct agent to write `decisions/{KEY}-behavior-change.md`, commit+push, stop at checkpoint.
- Keep `local-code-review exhausted 3 iterations`, `Agent disagreement between plan and review findings`, and `Cross-lane dependency check fails` as direct escalation stops (no decision document needed — these are operational blocks, not decisions).

**1c — "Which wave?" early-stop message:**
The wave gate not-closed stop message is already an operational stop with a clear message (not an interactive question). No change needed.

**Verification:** After editing, `grep -n "ask\|answer X\|STOP — answer" SKILL.md` returns zero hits for inline question patterns. The Escalation Triggers table still has 7 rows but the "What to report" cells now describe decision document writes instead of inline questions.

---

### Step 2: Update `execute-implementation-plan/SKILL.md` — replace STOP/ask patterns with decision document protocol

**Files:**
- `/home/zimmermann/.config/opencode/skills/execute-implementation-plan/SKILL.md`

**Action:** Two sub-changes:

**2a — Preconditions "ask" line (line ~21):**
Replace:
```
If anything is missing or unclear, ask — don't guess.
```
With: if a required artifact (issue or plan) is missing or a plan step is critically ambiguous, write `decisions/{KEY}-plan-gap.md` documenting the gap and options, commit+push, stop at a 🔴 checkpoint.

**2b — STOP Rules section:**
Current text says "Stop and ask the user when: [list]". Replace with:
- For `Blocked by missing context` and `Plan has a critical gap discovered during implementation` and `Instruction is genuinely ambiguous`: write `decisions/{KEY}-implementation-gap.md` with the specific question as context, commit+push, stop at 🔴 checkpoint.
- For `Verification fails repeatedly with no clear path forward`: write `decisions/{KEY}-verification-failure.md` describing the failure mode and possible recovery paths, commit+push, stop at 🔴 checkpoint.
- Keep `Review loop hit 3 iterations` as a direct escalation (same operational-block rationale as in Step 1).

**2c — Human escalation triggers section (lines ~110–114):**
Replace the four inline escalation trigger descriptions with decision document protocol matching Step 1b (scope change → `decisions/{KEY}-scope-change.md`, architectural decision → `decisions/{KEY}-architecture-decision.md`, security-sensitive → `decisions/{KEY}-security-review.md`, review loop exhausted → direct stop).

**Verification:** After editing, `grep -n "ask — don't guess\|Stop and ask\|Ask, don't guess" SKILL.md` returns zero hits. All question-triggering patterns are replaced with decision document writes.

---

### Step 3: Update `docs/plans/HOW-THIS-WORKS.md` — Human Escalation Triggers section

**Files:**
- `/repos/pa.aid.runbook-executor/docs/plans/HOW-THIS-WORKS.md`

**Action:** Section 5 ("Agent workflows") does not explicitly list Human Escalation Triggers, but Section 11 ("Cross-supervisor coordination") lists events and actions. Update Section 11's "What to do" column for:
- `Scope change discovered` → "Write `decisions/{KEY}-scope-change.md`; commit+push; stop at 🔴 checkpoint. Do not implement out-of-scope."
- `Security-sensitive change found` → "Write `decisions/{KEY}-security-review.md`; commit+push; stop at 🔴 checkpoint. Human sign-off required before proceeding."

Also add a new Section 16 titled `Decision Document Protocol` that:
1. States the decision document pattern replaces all inline "STOP — ask human" patterns in planning skills.
2. References `decisions/_template.md` as the canonical format.
3. Documents the two agent paths: Path A (write pending document → stop) and Path B (resume after `status: answered`).
4. States that answered documents are archived to `done/decisions/` by the `write-implementation-plan` skill after the plan is written.

**Verification:** `grep -n "ask interactively\|inline question\|STOP — answer" docs/plans/HOW-THIS-WORKS.md` returns zero hits. Section 16 is present with the decision document protocol summary.

---

### Step 4: Scan for any remaining inline STOP/ask patterns across all updated files

**Action:** After completing Steps 1–3, run a scan across the three modified skill files and HOW-THIS-WORKS.md for residual inline question patterns:

```bash
grep -rn "STOP.*answer\|ask.*user\|ask — don't guess\|Stop and ask" \
  /home/zimmermann/.config/opencode/skills/start-execution-session/SKILL.md \
  /home/zimmermann/.config/opencode/skills/execute-implementation-plan/SKILL.md \
  /repos/pa.aid.runbook-executor/docs/plans/HOW-THIS-WORKS.md
```

Any hit that is not in a "Common Mistakes" row or an explanatory prose context (i.e., describing what NOT to do) is a finding that must be resolved before marking this step complete.

**Verification:** The scan returns zero hits for interactive question patterns in a normative (instructional) context.

---

### Step 5: Commit all changes to pa.aid.runbook-executor and push

**Files:** `docs/plans/HOW-THIS-WORKS.md` and this implementation plan file

**Action:** Stage and commit all planning-repo artifact changes (HOW-THIS-WORKS.md + implementation plan) to `pa.aid.runbook-executor` main, then push.

Commit message:
```
docs(agent-workflow): add ARC-1347 implementation plan and HOW-THIS-WORKS update
```

**Note:** Skill file changes (Steps 1–2) are live edits to `/home/zimmermann/.config/opencode/skills/`. These are not in a git repo tracked here — they take effect immediately. A follow-on PR to `pa.aid.config.md` will be needed to persist them (same pattern as ARC-1346).

**Verification:** `git -C /repos/pa.aid.runbook-executor log --oneline -3` shows the commit; `git status` is clean.

---

## Testing & Validation

### Scenario 1 — AC 1: no inline question emitted at a decision point

Given `start-execution-session` is loaded and the agent hits an architectural decision trigger during session startup, the skill instructs the agent to write a decision document and stop at a checkpoint. The human receives a file path, not a question.

### Scenario 2 — AC 1: execute-implementation-plan gap handling

Given `execute-implementation-plan` is loaded and the agent discovers a critical plan gap, the skill instructs the agent to write `decisions/{KEY}-implementation-gap.md`, commit+push, and stop. The session can be resumed after the human answers.

### Scenario 3 — AC 2: HOW-THIS-WORKS.md escalation table updated

Read `docs/plans/HOW-THIS-WORKS.md` Section 11 — the "What to do" cells for scope change and security-sensitive triggers reference decision document writes, not interactive stops.

### Scenario 4 — AC 3: no inline STOP/ask patterns remain

Run the grep scan from Step 4. Zero normative hits across all three modified files.

---

## Risks & Open Questions

- **Skill files are shared and live**: edits to `/home/zimmermann/.config/opencode/skills/` take effect immediately for all sessions. An error in the skill text will affect all concurrent planning sessions. Mitigate: edit only the specific STOP/ask pattern sections; leave all other sections intact; run Step 4 scan before declaring done.
- **ARC-1346 decision document convention must match**: the decision document format used in the skill updates must match `decisions/_template.md` exactly (same 6 frontmatter fields, same `status: answered` string). If ARC-1346's template is modified after this story executes, both skill files will need to be re-synced.
- **`pa.aid.config.md` follow-on PR not in scope**: the live skill files at `/home/zimmermann/.config/opencode/skills/` are updated here, but the source-of-truth copies in `pa.aid.config.md` will be out of sync until a follow-on PR is raised (same open item as ARC-1346 PR #1).
- **`write-completion-summary` confirmed clean**: no inline STOP/ask patterns found during exploration. No update needed — but if the skill is edited after this story, any new patterns should be checked.
