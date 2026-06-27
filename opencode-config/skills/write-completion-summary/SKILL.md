---
name: write-completion-summary
description: Use when documenting completion of a work item with verifiable evidence mapped to acceptance criteria
---

# Write Completion Summary

## Purpose

Document work item completion with verifiable evidence mapped to each acceptance criterion. Evidence-first principle: every "Passed" claim requires proof.

## When to Use

- After all implementation steps complete and tests pass
- Before finalizing PR or closing issue
- To create auditable completion record

---

## Overview

Document the completion of a work item with evidence mapped to each acceptance criterion.

**Core principle:** Evidence before assertions. Every "Passed" criterion needs proof.

## Output

`task-completions/{EPIC}/{TRACKER-ID}-{kebab-title}-COMPLETION-SUMMARY.md`

## Required Sections

1. **Header** — TRACKER-ID, title, timing, cost
2. **Acceptance Criteria Verification** — each criterion → Passed/Failed/NA with evidence
3. **Implementation Summary** — what was built
4. **Verification Steps** — how to reproduce verification
5. **Tests** — test names and commands
6. **Rollback/Migration Notes** — if applicable
7. **Next Steps** — follow-on issues if any

## Evidence Principle

Each `Passed` criterion must include concise evidence:
- Test name and command
- Command and its output excerpt
- Log line or file path
- Screenshot description if UI

A criterion marked `Passed` with no evidence is not complete.

## After Writing

Update the issue's Completion Tracking section:
- Status: Completed
- Completion: 100%
- Last Updated: today's date

## Minimal Template

```markdown
# {TRACKER-ID} {Title} — Completion Summary

**Issue:** `issues/{TRACKER-ID}-{short-kebab-title}.md`
**Implementation Plan:** `implementation_plans/{TRACKER-ID}-implementation-plan.md`
**Completed:** {YYYY-MM-DD}
**Duration:** {time}
**Cost:** {cost if tracked}

## Acceptance Criteria Verification

| # | Criterion | Status | Evidence |
|---|-----------|--------|----------|
| 1 | Given X, when Y, then Z | Passed | `npm test` → "Z test passes" |

## Implementation Summary

{What was built, key decisions made}

## Verification Steps

```bash
# How to reproduce verification
{command}
```

## Tests Added/Modified

- `{test file}` — {what it tests}

## Rollback Notes

{None | migration steps | rollback procedure}

## Next Steps

- {Follow-on issue if any | None}
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Passed with no evidence | Add test name, command, or output excerpt |
| Summary describes code not outcomes | Map to acceptance criteria language |
| Issue Completion Tracking not updated | Required final step |
| Next Steps left blank | Write "None" explicitly |

## Runbook Check-Off (Mandatory)

After the completion summary is written and committed, find the corresponding story in the active runbook (`docs/plans/runbook-*.md`) and mark its checkbox as done: change `- [ ]` to `- [x]`. Commit this runbook update directly to `main`:

```
git add docs/plans/runbook-*.md
git commit -m "chore: check off <issueid> in runbook"
git push
```

**Definition of Done — final checklist:**
- [ ] Completion summary written and committed
- [ ] Issue `Status: Completed` and `Completion: 100%`
- [ ] Runbook checkbox for this story is `[x]`
- [ ] Runbook update committed to `main`
