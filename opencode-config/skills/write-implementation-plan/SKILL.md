---
name: write-implementation-plan
description: Use when translating an approved issue into a concrete HOW-to-execute plan with ordered technical steps
---

# Write Implementation Plan

## Purpose

Translate approved issue into concrete execution plan with ordered technical steps, file references, and step-level verification. Explore-then-approach discipline: no TBDs, no placeholders.

## When to Use

- Converting approved issue into how-to-execute plan
- After requirements and design are approved
- Before execute-implementation-plan

---

## Overview

Translate an approved issue into an execution plan. HOW, not WHAT — the issue stays the source of truth for requirements.

**Core principle:** Explore first. Approach before plan. No placeholders.

Three phases — in order, no skipping:
1. **Explore** — understand the codebase before writing anything
2. **Approach** *(when needed)* — propose options, get approval before committing to a direction
3. **Plan** — write concrete steps with file references and step-level verification

## Output

`implementation_plans/{EPIC}/{TRACKER-ID}-implementation-plan.md`

---

## Phase 0 — Explore

**Always run this phase before writing anything.**

1. Read `AGENTS.md` — architecture notes, key patterns, sibling repos, commit format
2. Read the issue file in full — extract: goal, constraints, AC, scope, out-of-scope
3. Explore relevant source — find files/modules the change will touch:
   - Search for existing implementations of similar patterns
   - Read files named or implied by the issue
   - Check sibling repos if the issue has cross-repo scope
4. Identify the existing pattern — how are similar things done today?
5. List unresolved unknowns — anything exploration couldn't answer

**Hard gate:** If unknowns remain after exploration that would produce a TBD in the plan — ask the user before proceeding. Do not write a partial plan.

---

## Phase 1 — Approach *(when needed)*

**Skip this phase only if** exploration made one approach obviously correct AND the issue is 🟢 LOW or 🟡 MEDIUM risk with an established pattern in the codebase.

**Run this phase when:**
- Multiple valid architectural approaches exist
- The approach has downstream impact (other lanes or stories depend on this)
- Issue is 🔴 HIGH risk
- Exploration revealed no clear existing pattern to follow
- The choice involves a trade-off the user should own

**Process:**
1. Propose 2-3 approaches with trade-offs
2. Lead with your recommended option and explain why
3. **STOP — wait for user approval before writing the plan**

Approach format:
```
Option 1 (recommended): {name}
  How: {1-2 sentences}
  Pro: {key advantage}
  Con: {key trade-off}

Option 2: {name}
  How: {1-2 sentences}
  Pro: {key advantage}
  Con: {key trade-off}

Option 3: {name}
  ...

Recommendation: Option {N} because {reason}.
```

**Do not write the plan until the user selects an approach.**

---

## Phase 2 — Write Plan

Write `implementation_plans/{EPIC}/{TRACKER-ID}-implementation-plan.md`.

### Traceability Header (required)

```markdown
# {TRACKER-ID} {Title} - Implementation Plan
**Issue:** `issues/{TRACKER-ID}-{short-kebab-title}.md`
**Completion Summary:** `task-completions/{TRACKER-ID}-{short-kebab-title}-COMPLETION-SUMMARY.md` (TBD)
**Approach:** {name of chosen approach, or "single approach — pattern established"}
**Owner:** {name}
**Date:** {YYYY-MM-DD}
```

### Required Sections

1. **Traceability Header**
2. **Scope & Alignment** — what the plan covers, how it maps to AC
3. **Assumptions & Dependencies** — explicit prerequisites, not "should work"
4. **Implementation Steps** — ordered, concrete, with step-level verification
5. **Testing & Validation** — how to verify each step and end-to-end
6. **Risks & Open Questions** — known unknowns that could block

### Implementation Steps

Each step must:
- Reference specific files, modules, or components (from Phase 0 exploration)
- Include step-level verification (how to confirm this step is done)
- Be ordered by dependency (earlier steps enable later ones)
- Be executable without guessing missing context

### Scope Check

Does the plan span multiple independent subsystems? If yes → split into separate plans. Each plan covers one coherent area of change.

### No Placeholders Rule

These are FAILURES in a plan:
- "TBD"
- "TODO"
- "implement later"
- "similar to Task N"
- "see above"

If you don't know a detail after exploration — ask, don't defer.

---

## Abstraction Level — What Belongs in a Plan

> A plan is reviewed without the codebase open. If a reviewer needs the full repo to validate the plan, the plan is too detailed. Code belongs in the PR, not the plan.

### Allowed

- Intent statement: what the change achieves
- Files to modify (paths, not line numbers)
- Interface contracts: function/method/type signatures ONLY — no body
- Props/parameter names and types
- Behavioral constraints and edge cases in prose
- Test scenarios: what to verify, not how to write the test

### Not Allowed

- Code blocks longer than 5 lines
- Full method/function implementations
- Complete component implementations (JSX, templates)
- Hardcoded values (hex colors, magic numbers, pixel values)
- Line-number references to existing code (`line 77-81`) — rot immediately; use symbol/function names instead
- Test code (test files, assertions, mock setup)

### Smell Test

If a step contains a full function body → delete the body, keep the signature and intent.  
If an assumption cites `line 77-81` → replace with the function/symbol name.  
If the testing section has mock setup or assertions → replace with scenario descriptions.

---

## Self-Review After Writing

1. **Spec coverage** — does every AC have a step that satisfies it?
2. **Placeholder scan** — any TBD, TODO, vague steps?
3. **File references** — every step has specific paths from exploration?
4. **Type consistency** — consistent naming of modules, functions, paths?
5. **Step ordering** — dependencies respected?
6. **Approach recorded** — traceability header names the chosen approach?

---

## Quality Checklist

- [ ] Phase 0 complete — codebase explored, unknowns resolved
- [ ] Phase 1 done or explicitly skipped with reason
- [ ] Traceability header complete (including chosen approach)
- [ ] Every AC from issue is addressed by a step
- [ ] All steps reference specific files/modules
- [ ] No placeholders (TBD, TODO, "implement later")
- [ ] Each step has a verification
- [ ] Dependencies and risks explicit
- [ ] Scope check done — not spanning independent subsystems
- [ ] No code block exceeds 5 lines
- [ ] No line-number references to existing code (use symbol names)
- [ ] Steps describe intent and interface contract, not implementation
- [ ] Testing section lists scenarios, not test code

---

## Minimal Template

```markdown
# {TRACKER-ID} {Title} - Implementation Plan
**Issue:** `issues/{TRACKER-ID}-{short-kebab-title}.md`
**Completion Summary:** `task-completions/{TRACKER-ID}-{short-kebab-title}-COMPLETION-SUMMARY.md` (TBD)
**Approach:** {chosen approach name or "single approach — pattern established"}
**Owner:** {name}
**Date:** {YYYY-MM-DD}

## Scope & Alignment
{What this plan covers and how it maps to the issue's acceptance criteria}

## Assumptions & Dependencies
- {Explicit assumption or prerequisite}

## Implementation Steps

### Step 1: {Title}
**Files:** `{path/to/file.ts}`
**Action:** {Concrete description of change}
**Verification:** {How to confirm step is complete}

## Testing & Validation
{How to run tests, what to verify end-to-end}

## Risks & Open Questions
- {Known unknown or risk}
```

---

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Writing plan without exploring codebase | Phase 0 is mandatory — explore first |
| File paths guessed or generic | Paths come from Phase 0 exploration — specific always |
| Skipping approach phase for HIGH story | Phase 1 is required for HIGH risk or ambiguous approach |
| Writing plan before approach approved | Hard stop at Phase 1 — wait for user selection |
| TBD or TODO in any step | Research or ask — never defer |
| AC not mapped to steps | Every criterion needs at least one step |
| Plan spans two independent subsystems | Split into two plans |
| Code block > 5 lines in a step | Keep signature + intent only — body goes in PR |
| Line-number reference (`line 77-81`) | Replace with symbol/function name |
| Full component or method in plan | Plan is too detailed — delete body, keep contract |
| Test code or assertions in Testing section | Replace with scenario descriptions in prose |