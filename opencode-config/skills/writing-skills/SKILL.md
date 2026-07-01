---
name: writing-skills
description: Use when creating new skills, editing existing skills, or verifying skills work before deployment
---

# Writing Skills

## Purpose

Write skills as test-driven process documentation: pressure scenario first, minimal instruction second, verification last.

Core rule: if no agent failure or pressure scenario was observed, you do not know what the skill must teach.

## When to Use

- Creating a new pa.aid-owned skill.
- Editing a pa.aid-owned skill.
- Harmonizing skill style, triggers, catalogues, or discovery text.
- Verifying skill behavior before deployment.
- Not for external/plugin-owned skills unless explicitly adopting them into pa.aid ownership.

## Inputs / Preconditions

- Canonical skills live in `/repos/pa.aid.conductor.ts/opencode-config/pa.aid.config.md/skills/`.
- Runtime copies live in `~/.config/opencode/skills/` and may include external/plugin skills.
- Each skill lives in `skills/<skill-name>/SKILL.md`.
- Load `test-driven-development` before changing executable validation code.

Existing exceptions: `_index` and `ccoe_mod` are grandfathered names. New skills use kebab-case.

## Output Contract

Every pa.aid-owned `SKILL.md` uses this baseline shape unless a section is irrelevant:

```markdown
---
name: skill-name
description: Use when <specific trigger keywords/symptoms>; <optional exclusion>
---

# Skill Name

## Purpose
1-2 sentences. What this skill protects or enables.

## When to Use
- Concrete trigger
- Concrete trigger
- Not for: adjacent case

## Inputs / Preconditions
- Required artifact/context
- Required tool/access

## Workflow
1. Step
2. Step
3. Step

## Output Contract
Expected file/report/answer format.

## Verification
Commands/checks/evidence required.

## Stop / Escalate
When to ask user or hand off.

## Common Mistakes
| Mistake | Fix |
|---|---|

## Related Skills
- `other-skill`
```

Domain reference skills may replace `Workflow` with `Quick Reference`, `Patterns`, or `Examples`.

## Frontmatter Rules

- `name` matches folder.
- New `name` values use lowercase kebab-case only.
- `description` starts with `Use when`.
- Description is trigger-only: user words, filenames, tools, symptoms, error text.
- Do not summarize workflow in description; agents use descriptions to decide whether to load the full skill.
- Keep `name + description` under 1024 chars.

## Workflow

### RED: Pressure Scenario First

- Document observed failure, ambiguity, or routing miss.
- For discipline skills, run a baseline scenario without the changed skill when practical.
- Capture exact rationalization or wrong choice.

### GREEN: Minimal Skill Change

- Add only wording that blocks the observed failure.
- Prefer tables/lists over prose.
- Keep examples exact and copy-pastable.
- Move heavy reference to supporting files.

### REFACTOR: Verify and Harmonize

- Re-run same scenario or perform static verification.
- Update `_index` and `using-skills` when discovery, triggers, names, or categories change.
- Sync runtime copy only after canonical source is correct.
- Leave external/plugin skills alone unless explicitly adopted.

## Verification

- YAML frontmatter parses.
- `name` matches folder, except documented grandfathered names.
- Description starts with `Use when`.
- Discovery catalogues mention new/renamed skills.
- Runtime/source drift is intentional and documented.
- For behavior rules: pressure scenario now passes or static check shows route is unambiguous.

## Writing Rules

- Short, imperative, scannable.
- `MUST` / `NEVER` only for hard constraints.
- No session history or storytelling.
- No duplicate long reference.
- Tables over prose.
- Flowcharts only for non-obvious decisions or loops.
- Frequently loaded skills: target <300 words; others target <500 words unless reference depth is required.
- Code, commands, URLs, identifiers exact.

## Stop / Escalate

- No concrete pressure scenario and change affects behavior: ask for scenario or document static rationale.
- Skill is external/plugin-owned: do not edit; report upstream/owner boundary.
- Change would rename a loaded skill: ask before migration.
- Conflict with user/system/developer instruction: stop and report conflict.

## Common Mistakes

| Mistake | Fix |
|---|---|
| Description explains workflow | Change to trigger-only `Use when...` |
| Editing runtime copy first | Edit canonical source first, then sync |
| External skill edited as pa.aid-owned | Leave it alone or explicitly adopt it |
| Catalogue not updated | Update `_index` and `using-skills` |
| Large prose dump | Use table/list; move reference out |
| Skill change without scenario | Add pressure scenario or static rationale |

## Related Skills

- `test-driven-development` — same RED/GREEN/REFACTOR discipline for code.
- `using-skills` — discovery and loading rules.
