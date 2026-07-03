---
key: ARC-1363
story: Extend generate_runbook data model to reproduce the canonical batch-structured runbook format
status: decided
selected_approach: Option 1
decided_by: Jochen Zimmermann
decided_at: 003.07.2026
---

## Decision Context

**What the story is trying to achieve:** `generate_runbook` must emit runbook markdown that is structurally identical to what the `generate-lane-runbooks` skill defines as canonical — the same format hand-written runbooks (e.g. `runbook-documentation-cleanup.md`) already use.

**The constraint that forced this decision:** Testing against a real runbook (`/repos/pa.aid.ansible-test-conductor/docs/plans/runbook-documentation-cleanup.md`) revealed that the tool's current input schema — a flat `waves[].stories[]` list where every story is an independent block — **cannot express the canonical structure**. The canonical format (defined in `generate-lane-runbooks/SKILL.md` lines 46–160) is organized around **batches**, not individual stories, plus rich preamble and register sections.

**Concretely, the current tool cannot produce any of the following** (all present in every canonical runbook):

1. **Batch grouping** — MEDIUM/LOW stories grouped (max 5 per batch), each batch sharing:
   - ONE `🔒 Claimed:` row (not one per story)
   - ONE shared `Read all issues:` step (with a file sub-list)
   - ONE shared `Write all plans:` step (with a per-story plan-path sub-list)
   - ONE `🟡 BATCH PLAN CHECKPOINT` covering all stories in the batch
   - Per-story `Execute {KEY}` items, each with `Lint / tests pass` / `Write completion summary` / `Commit` sub-items
   - A `☑ all complete` closer
2. **Header preamble** — the `> Load skill / Stop at checkpoints / Resume` blockquote
3. **`## Checkpoint model` table**
4. **`## HIGH story workflow` and `## MEDIUM/LOW batch workflow`** documentation sections
5. **Wave gate blockquote** (`> Gate: …`) at the top of a wave
6. **On-Hold conditional story blocks** (`### On-Hold story — {KEY}` with "If {blockers} complete: … / If not: skip")
7. **Inline story notes** (`> ✅ CLOSED in Jira — skip this story`)
8. **`### Wave N gate`** detailed checklist (All stories checked / All PRs merged / All completion summaries / 🟢 WAVE GATE)
9. Extended metadata fields: `**Feature goal:**`, `**Wave(s):**`, `**Repos:**`, `**Depends on:**`

**Companion issue — `validate_runbook` (ARC-1362) is also wrong for batch runbooks.** Its `story.missing_claimed` rule requires a `🔒 Claimed:` sub-item on *every* unchecked story. In the canonical batch model the claim is at the **batch level**, covering all member stories. This is why the correct, canonical original file reports 18 false errors (15× `story.missing_claimed` + 3× `step.duplicate_id`). Whatever schema we pick for the generator, the validator must be taught the batch-claim model so a canonical runbook validates clean.

**Why a human must choose:** This materially expands ARC-1363 beyond its written acceptance criteria and input schema. The schema shape is a design commitment that affects the skill, the validator, and every future runbook. It should not be chosen unilaterally.

---

## Options

Option 1 (recommended): Full canonical model — batches + individual HIGH stories + preamble + on-hold as first-class schema
  How: Extend the input schema so a wave contains `high_stories[]` (individual 9-step blocks) and `batches[]` (each batch = label + priority + shared read/plan/checkpoint + member stories with per-story execute sub-items + closer). Add top-level `feature_goal`, `repos`, `waves_participated`, `depends_on`, optional `skills`, and an `on_hold[]` register. The generator emits the fixed preamble (blockquote, checkpoint-model table, both workflow docs), per-wave gate blockquote, batch blocks, on-hold conditional blocks, `### Wave N gate` checklist, On-Hold Register table, and checkpoint summary table — matching `generate-lane-runbooks/SKILL.md` byte-for-structure. `validate_runbook` is updated so a batch-level claim satisfies the claim requirement for all member stories, and the duplicate-id rule tolerates the "read list + execute list" repetition within a batch.
  Pro: Faithfully reproduces the canonical format; the tool becomes the single source of truth the issue intended; a regenerated `runbook-documentation-cleanup` matches the original and validates clean.
  Con: Largest change — new nested schema, generator rewrite, validator changes, new tests; meaningfully re-scopes ARC-1363 (should reopen it or open a follow-up).

Option 2: Minimal generator + skill-driven batching
  How: Keep the generator's flat per-story model but add only the missing preamble/register scaffolding. Batching, shared checkpoints, and on-hold blocks stay the responsibility of the `generate-lane-runbooks` skill / calling agent, which assembles those sections by hand or via string composition around the tool output.
  Pro: Smallest code change to the tool; keeps ARC-1363 close to its original scope.
  Con: Does not solve the actual problem — the tool still cannot emit a canonical runbook; the "agents misformat markdown" failure mode the issue exists to eliminate returns for exactly the most complex part (batches). Contradicts the single-source-of-truth goal.

Option 3: Two-layer model — canonical AST types + thin generator
  How: Define a full canonical runbook AST (metadata, waves, high-stories, batches, on-hold, gates) mirroring the parser's `ParsedRunbook` plus the batch/preamble extensions, and make `generate_runbook` a thin serializer over that AST. Callers build the AST. `validate_runbook` and `generate_runbook` share the extended type module so read and write stay symmetric.
  Pro: Cleanest long-term architecture; read (`parseRunbook`) and write (`generate_runbook`) share one type model; maximizes parser-first symmetry.
  Con: Highest up-front design cost; requires extending the shared parser/types (`lib/types.ts`) which is copied from `pa.aid.conductor.ts` — touches the parser-copy contract and needs the conductor types kept in sync.

Recommendation: Option 1 because it directly and faithfully reproduces the canonical format the skill already mandates, fixes both tools together, and delivers exactly what the failed test exposed — without taking on the parser-copy-contract risk of Option 3. Option 3's symmetry is attractive but can be a later refactor once the canonical model has proven stable in Option 1.

## Recommendation

Choosing **Option 1**. It solves the demonstrated gap (batches, preamble, on-hold, wave gates), keeps the change contained to the two agent tools plus the skill, and avoids modifying the parser-copy contract that Option 3 would disturb. ARC-1363 should be reopened (or a follow-up story ARC-13xx created) to cover the expanded scope, and ARC-1362 amended to teach the validator the batch-claim model.

## Notes

- Authoritative format spec: `~/.config/opencode/skills/generate-lane-runbooks/SKILL.md` lines 46–160.
- Concrete test fixture to reproduce: `/repos/pa.aid.ansible-test-conductor/docs/plans/runbook-documentation-cleanup.md` (canonical, hand-written, 184 lines).
- Acceptance oracle after the change: regenerating that runbook from structured input must produce a file that (a) matches the original section-for-section and (b) passes the updated `validate_runbook` with zero errors — and the **original file must also validate clean** once the batch-claim rule is fixed.
- Both tools live at `~/.config/opencode/tools/` (not a git repo); planning artifacts (issue, plan, completion summary, this decision) are tracked in `pa.aid.runbook-executor`.
