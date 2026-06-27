---
name: create-issue
description: Use when authoring a new engineering issue — defines WHAT to build, scope, constraints, and acceptance criteria
---

# Create Issue

## Purpose

Write engineering issues defining WHAT to deliver (not HOW). Issues are source of truth for requirements; implementation plan covers execution. Enforces FDD: verb-phrase summaries, Given/When/Then AC, black-box observable outcomes.

## When to Use

- Authoring new engineering issue with scope and AC
- Writing story for feature epic
- Before write-implementation-plan

---

## Overview

Write engineering issues that define WHAT to deliver — not HOW. The issue is the source of truth for requirements; the implementation plan covers execution.

**Core principle:** Issues define outcomes and boundaries. Implementation order belongs in the plan.

## Story Style — FDD

Issues use Feature-Driven Development (FDD) style — NOT user story format.

| Format | Rule |
|--------|------|
| **Summary** | Verb phrase: "[action] [result/object]" — e.g. `Export report as PDF`, `Display account balance` |
| **Description** | Goal + constraints in plain prose |
| **Acceptance Criteria** | Given/When/Then — observable black-box behavior |

**Never use:** "As a [role], I want [feature], so that [benefit]" — in any field, ever.

## Sections (exact order)

1. **Summary** — TRACKER-ID + short title (one line)
2. **Related** — links to issue, implementation plan (TBD), completion summary (TBD)
3. **Description** — goal + constraints (1-3 sentences)
4. **Scope** — In Scope (outcome bullets) + Dependencies
5. **Acceptance Criteria** — Given/When/Then, black-box observable
6. **Out of Scope** — explicit exclusions
7. **Completion Tracking** — NOT written at authoring time; appended when work begins

**Output:** Save to `issues/{EPIC}/{TRACKER-ID}-{kebab-title}.md`

## Quality Checklist

Before finalizing, verify ALL:

- [ ] Summary includes TRACKER-ID and concise title
- [ ] Description includes goal plus explicit constraints
- [ ] Scope defines outcomes and boundaries (NOT implementation steps)
- [ ] Dependencies explicit or "None beyond existing ..."
- [ ] Acceptance criteria are Given/When/Then and testable
- [ ] Out-of-scope list is explicit and aligned with constraints
- [ ] No implementation-order detail embedded in the issue

## Refactor Issues

If behavior-preserving, explicitly state:
- No public method signature changes
- Preserve event contracts
- Preserve error handling
- No new persistence paths
- No added performance cost

## Minimal Template

```markdown
**Summary**
{TRACKER-ID}: {Title}

**Related**
- Issue: `issues/{TRACKER-ID}-{short-kebab-title}.md`
- Implementation Plan: `implementation_plans/{TRACKER-ID}-implementation-plan.md` (TBD)
- Completion Summary: `task-completions/{TRACKER-ID}-{short-kebab-title}-COMPLETION-SUMMARY.md` (TBD)

**Description**
{Goal statement}. {Constraints}. {Important exclusions if needed}.

**Scope**
- **In Scope:**
  - {Outcome 1}
- **Dependencies:**
  - {None beyond existing ... | explicit list}

**Acceptance Criteria**
1. Given {precondition}, when {action}, then {observable result}.

**Out of Scope**
- {Explicit exclusion 1}
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Acceptance criteria describe internals | Rewrite as observable black-box behavior |
| Scope lists implementation steps | Replace with outcome statements |
| Dependencies left vague | List explicitly or state "None beyond existing X" |
| Out of Scope missing | Required — prevents scope creep |
| Completion Tracking filled at authoring | Leave blank; fill when work begins |
| User story format ("As a…") | Rewrite as FDD verb phrase in summary; plain goal prose in description |
