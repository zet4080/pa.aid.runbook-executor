---
name: write-workorder
description: Use when a set of rewritten issues exists in an issues/ directory and a workorder document needs to be produced — converts epics and stories into independent-lane execution plan with explicit cross-lane prerequisites, effort estimates, and risk classification.
---

# Skill: write-workorder

## Purpose

Produce workorder doc from issue files: feature-first execution plan with one lane per epic, maximum parallelism, explicit cross-lane prerequisites, and risk-classified stories.

## When to Use

- Converting issue files (after restructure-epics or create-epics) into workorder
- Before generate-lane-runbooks
- When execution plan with inter-epic dependencies needed

---

## Prerequisites

Before invoking this skill:
- Rewritten issue files exist in `issues/` directory (one file per story/task)
- `docs/plans/workorder.md` does not yet exist
- Epic list is known (from Jira or issue files)

## Step 1 — One lane per epic

Assign each epic a lane. Default: one lane per epic. Lane name = feature name (not Lane A/B/C).

Merge two epics into one lane ONLY when:
- Epic B cannot start until Epic A is ≥80% complete (hard data dependency)
- Document every merge with explicit rationale

## Step 2 — Build inter-epic dependency graph

For each epic identify:
- Which other epics must be complete (or reach a named story) before this epic can start
- Which stories within this epic have intra-epic ordering constraints
- Which epics are fully independent (can run in parallel immediately)

Foundation epic (if present): list the specific stories that block other epics — not the whole epic.

## Step 3 — Identify start conditions per lane

For each lane, determine its start condition:

- **Immediately** — lane has no cross-lane prerequisites. Can start on day 1, fully parallel with all other immediately-starting lanes.
- **After [story X] merged** — lane has one named cross-lane prerequisite. The runbook for this lane begins with a pre-check: "verify [story X] is merged before starting".

Rules:
1. Default is "immediately". Only assign a dependency if a specific story in another lane is a hard technical prerequisite.
2. If a story within a lane depends on another story in the SAME lane — that is intra-lane ordering, not a start condition.
3. Do NOT create a global sync gate. "All other lanes must finish before this lane starts" is almost never correct.
4. If two lanes have a cross-lane dependency, record it in the Cross-Lane Dependencies table and in the dependent lane's start condition. Nothing else.
5. Independent lanes get fully independent runbooks. No wave barrier.

Examples of start conditions:
- Lane with 4 internal sequential phases → start condition: "immediately" (phases are intra-lane)
- Lane whose first real story needs an SDK framework from another lane → start condition: "after [SDK story] merged"
- Lane with a single standalone story → start condition: "immediately"

## Step 4 — Estimate effort

Per story:
- 🟢 LOW (concept/doc/analysis only, no code changes): 1–2h
- 🟡 MEDIUM (single artifact change, non-trivial, moderate scope): 2–4h
- 🔴 HIGH (new artifact, cross-repo, security-critical, wide blast radius, architectural decision): 3–6h

Per lane: sum of story estimates + 20% coordination overhead.
Wall-clock total = longest independently-running lane (all immediate lanes run in parallel).
Supervision budget: ~15 min per story checkpoint (no per-wave gate overhead).

## Risk Classification

Classify every actionable story as 🔴 HIGH / 🟡 MEDIUM / 🟢 LOW. Drives checkpoint model in runbooks.

| Level | Criteria |
|-------|----------|
| 🔴 HIGH | Cross-repo change · security enforcement · wide blast radius (5+ artifacts) · architectural decision blocking downstream · new artifact that is a dependency for other new artifacts |
| 🟡 MEDIUM | New single artifact · moderate-scope single-artifact change · config structural addition · artifact migration using a defined pattern |
| 🟢 LOW | Concept/analysis document · documentation · on-hold evaluation · single-param addition to existing section |

Record per lane in workorder under each lane section:

```markdown
**🔴 HIGH stories (individual checkpoints):**
{KEY1} (reason)
```

## Workorder Document Structure

Write `docs/plans/workorder.md` using this template:

```markdown
# Workorder

## Execution Model
[roles: coding agent, review agent, human supervisor]
[workflow per story — load generate-lane-runbooks skill to produce runbooks]

## Feature Lane Overview
[table: Lane | Feature | Epic Key | Stories | Est. Total | Start condition]
[ASCII parallel lane diagram — one column per lane]
[critical path lane, wall-clock estimate (longest lane total)]

Each lane is independently executable. Start conditions are story-level prerequisites, not global wave gates.

## Lane: {feature-name}
**Epic:** {KEY} | **Repo:** {repo} | **Start condition:** immediately / after {STORY-KEY} merged

[table: Story | Summary | Est. | Intra-lane depends on | Risk]

**🔴 HIGH stories (individual checkpoints):**
{KEY1} (reason)

## Effort & Schedule Summary
[table: Lane | Stories | Wall-clock | Agent-hours total | Supervision]

## Supervision Budget
[per-story checkpoints, grand total]

## Cross-Lane Dependencies
[table: Story | Blocks | Type | Notes]

## On-Hold Items
[table: Story | Lane | Blocked by | Revisit at]

## Known Risks
```

## Quality Checks

Before committing the workorder, verify:
- [ ] Every epic/feature has exactly one lane (merges documented with rationale)
- [ ] Every story appears in exactly one lane
- [ ] No lane has 'after X' start condition unless X is in a different lane
- [ ] Cross-Lane Dependencies table complete — no implicit dependencies
- [ ] Wall-clock = longest lane total (all immediate lanes run fully in parallel)
- [ ] On-Hold items listed with named blockers and revisit condition

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| One lane per technical layer instead of per feature | Regroup: lane = feature/epic |
| Start conditions chain more than 1 hop | Flatten: only one hop of cross-lane dep is almost always sufficient |
| Cross-lane dependency not in dependency table | Every cross-lane assumption must be explicit |
| Wall-clock calculated as sum of all lanes | Wall-clock = longest lane (they run in parallel) |
| Merging lanes for convenience | Only merge on hard dependency — document rationale |
| Global sync gate added "all Wave N must complete first" | Remove: only name specific story prerequisites, never whole-wave gates |

## Commit

After writing `docs/plans/workorder.md`:

```
git add docs/plans/workorder.md
git commit -m "docs: add workorder with parallel lanes and start conditions"
```

## Next Step

Load the `generate-lane-runbooks` skill to convert the workorder into executable per-lane agent runbooks with checkpointed workflows.
