# ARC-1363 generate_runbook (Canonical Batch Model) - Implementation Plan
**Issue:** `issues/agent-tools/ARC-1363-tool-generate-runbook.md`
**Completion Summary:** `task-completions/ARC-1363-COMPLETION-SUMMARY.md`
**Approach:** Option 1 — full canonical batch model (see `decisions/ARC-1363-batch-model-approach-decision.md`, answered: Option 1)
**Owner:** debug agent
**Date:** 2026-07-03

## Scope & Alignment

Extend `generate_runbook` to emit the full canonical runbook format defined by the `generate-lane-runbooks` skill (SKILL.md lines 46–160) and exemplified by `runbook-documentation-cleanup.md`. Fix `validate_runbook` so batch-structured runbooks validate clean. Both tools live at `~/.config/opencode/tools/`.

Maps to ARC-1363 AC (expanded under Option 1):
- AC1 (writes file + result struct) — preserved.
- AC2 (HIGH → INDIVIDUAL PLAN CHECKPOINT) — preserved, now inside individual HIGH story blocks.
- AC3 (dependencies → Blocked by) — preserved for HIGH stories; batch stories carry a `depends_note` instead.
- AC4 (unchecked → Claimed) — reinterpreted: claim is per **block** (HIGH story / batch / on-hold), matching canonical format.
- AC5 (checkpoint_count → summary table) — replaced by the canonical 5-column summary (HIGH checkpoints / Batch checkpoints / Wave gates / Total per wave).
- AC6 (missing field → error, no file) — preserved.
- AC7 (output passes validate_runbook) — preserved; PLUS the original hand-written file must also validate clean.

## Assumptions & Dependencies

- Parser (`lib/astWalker.ts`) is authoritative and unchanged: waves = `## Wave N`; H3 sets context only; top-level list items under a wave = steps; nested items = sub-steps; step id from `[A-Z]+-\d+` in item label → H3 → positional; checkpoint iff label has CHECKPOINT/GATE.
- The canonical format (SKILL.md) is the target. Byte-identical reproduction of the original (Jira links, exact commit text) is NOT required; structural + content fidelity is.
- Both tools are auto-discovered; not in a git repo. Planning artifacts tracked in `pa.aid.runbook-executor`.
- Existing callers use the legacy flat `waves[].stories[]` schema (e.g. the generation of `runbook-agent-tools.md`). Backward compatibility is retained via a legacy adapter.

## Implementation Steps

### Step 1: Extend generate_runbook input schema
**Files:** `~/.config/opencode/tools/generate_runbook.ts`
**Action:** Add new interfaces alongside existing ones:
- `GenerateRunbookInput` gains optional: `feature_goal`, `repos`, `waves_participated`, `depends_on`, `skills`, `out_of_checklist_note`, and `on_hold?: OnHoldStoryInput[]`.
- `WaveInput` gains optional: `gate_note`, `no_high_note`, `dependent_gate_checks?: string[]`, `high_stories?: HighStoryInput[]`, `batches?: BatchInput[]`. `stories?` becomes optional (legacy).
- `HighStoryInput`: `key`, `title`, `dependencies?: string[]`, `depends_note?`, `notes?`.
- `BatchInput`: `label`, `priority: '🟡' | '🟢'`, `stories: BatchStoryInput[]`, `claimed?: {by, done}`.
- `BatchStoryInput`: `key`, `title`, `issue_file?`, `plan_path?`, `commit_message?`, `closed_note?`, `depends_note?`.
- `OnHoldStoryInput`: `key`, `title`, `blocked_by: string[]`, `issue_file?`, `plan_path?`, `commit_message?`, `action_if_unblocked`.
**Verification:** `node --experimental-strip-types --check generate_runbook.ts` passes.

### Step 2: Legacy adapter
**Files:** `generate_runbook.ts`
**Action:** Add `normalizeWave(wave)` — if a wave has `high_stories`/`batches`, pass through; else derive from legacy `stories[]`: 🔴 → `high_stories`, remaining 🟡/🟢 grouped into a single default batch labelled by dominant priority. Keeps existing tests and callers working.
**Verification:** Legacy-input unit test produces one HIGH block + one batch block.

### Step 3: Preamble + workflow docs builders
**Files:** `generate_runbook.ts`
**Action:** Add `buildHeader()` (H1 `# Runbook: {feature} — {epic}`, the Load/Stop/Resume blockquote), `buildMetadataBlock()` (Feature goal, Epic, Lane, Wave(s), Repos, Skills to load at start, Depends on; optional out-of-checklist note), `buildCheckpointModel()` (3-row table), `buildWorkflowDocs()` (HIGH workflow + MEDIUM/LOW batch workflow numbered lists), then `---`.
**Verification:** Generated output contains `## Checkpoint model`, `## HIGH story workflow`, `## MEDIUM/LOW batch workflow`.

### Step 4: HIGH story block builder
**Files:** `generate_runbook.ts`
**Action:** `buildHighStoryBlock(story, lane)` emits `### 🔴 {KEY} — {title}` then a top-level `- [ ] **{KEY}** — {title}` with sub-items in canonical order: `🔒 Claimed: —`, optional `🔒 Blocked by:` per dependency, `Read issues/...`, `Write implementation_plans/...`, `🔴 INDIVIDUAL PLAN CHECKPOINT`, `Execute plan`, `Run local-code-review`, `Lint / tests pass`, `Write task-completions/...`, `Commit`.
**Verification:** Unit test: HIGH block contains one Claimed and one INDIVIDUAL PLAN CHECKPOINT sub-item.

### Step 5: Batch block builder
**Files:** `generate_runbook.ts`
**Action:** `buildBatchBlock(batch, lane)` emits `### {priority} Batch — {label}` then top-level items: ONE `🔒 Claimed: —`; `Read all issues: {keys}` with a `Files:` sub-item listing `issue_file`s; `Write all plans:` with per-story `plan_path` sub-items; `{priority} BATCH PLAN CHECKPOINT`; per story an `Execute {KEY} — {title}{depends_note}` item with sub-items `Lint / tests pass`, `Write task-completions/{KEY}-COMPLETION-SUMMARY.md`, `Commit: {commit_message}`, optional trailing `> {closed_note}` blockquote; then `☑ all complete`.
**Verification:** Unit test: batch block has exactly one Claimed top-level item and one BATCH PLAN CHECKPOINT; each story has an Execute item.

### Step 6: On-hold + wave gate + registers
**Files:** `generate_runbook.ts`
**Action:**
- `buildOnHoldBlock(story, lane)` emits `### On-Hold story — {KEY}`, the "If {blockers} complete:" line, a checklist (Claimed, Read, Write plan, BATCH PLAN CHECKPOINT, Execute plan, Lint, Write completion, Commit), then "If not: log in On-Hold Register below and skip."
- `buildWaveGate(wave)` emits `### Wave N gate` with `All active stories… / All PRs merged / All completion summaries written / 🟢 WAVE GATE — …`.
- `buildOnHoldRegister(onHold)` emits `## On-Hold Register` table (Story | Blocked by | Last checked | Action if unblocked); "No items on hold." when empty.
- `buildCheckpointSummary(waves)` emits `## Checkpoint Summary` 5-column table (Wave | HIGH checkpoints | Batch checkpoints | Wave gates | Total).
**Verification:** Generated file contains all four sections with correct row counts.

### Step 7: Rewrite computeCheckpointCounts + buildWaveSection + buildRunbookMarkdown
**Files:** `generate_runbook.ts`
**Action:** `computeCheckpointCounts` returns per-wave `{high, batch, gates, total}` where high = number of HIGH story blocks, batch = number of batch blocks, gates = 1 per wave. `buildWaveSection(wave, lane)` emits `## Wave N — title`, gate_note blockquote, no_high_note, dependent gate checks, HIGH blocks, batch blocks, on-hold blocks scoped to the wave, then the wave gate. `buildRunbookMarkdown` composes header → metadata → checkpoint model → workflow docs → `---` → waves → On-Hold Register → Checkpoint Summary.
**Verification:** Full generated documentation-cleanup file parses via `parseRunbook()` without throwing.

### Step 8: Update tool arg description + execute
**Files:** `generate_runbook.ts`
**Action:** Extend the `waves` arg description and add optional top-level string args (`feature_goal`, `repos`, `waves_participated`, `depends_on`, `skills`, `on_hold` as JSON string). Parse and thread through. Keep required args (`lane_name`, `epic_key`, `output_path`, `waves`).
**Verification:** Tool invocation with canonical input returns `{ success: true, ... }`.

### Step 9: Make validate_runbook batch-aware
**Files:** `~/.config/opencode/tools/validate_runbook.ts`
**Action:** Add a raw-mdast block grouper `groupBlocksByH3(root)` producing blocks `{ h3Text, kind, topLevelItems[], storyKeys[] }` where kind ∈ high|batch|onhold|gate|other (classified from H3 text: `/batch/i` → batch, `/on.hold/i` → onhold, `/wave.*gate/i` or GATE → gate, `**KEY**` heading item → high). Rewrite:
- `story.missing_claimed`: require exactly one Claimed among top-level items of each high/batch/onhold block; do not require per-Execute-item claims.
- `step.duplicate_id`: collect story-definition keys from HIGH blocks (`**KEY**` item) and batch `Execute {KEY}` items; flag only a key defined as a story twice. Exclude "Read all issues" / "Write all plans" aggregation items from key collection.
**Verification:** Unit tests below; original documentation-cleanup file → 0 errors.

### Step 10: Adapt summary + gate rules in validate_runbook
**Files:** `validate_runbook.ts`
**Action:** Update `wave.missing_gate` to detect a `### Wave N gate` block (kind gate) per wave. Update `summary.count_mismatch` to accept the 5-column canonical summary table and remain warning-only (tolerant if columns differ). Keep metadata, wave.exists, wave.heading, structure rules.
**Verification:** Canonical generated file → 0 warnings for gate/summary.

### Step 11: Update tests for both tools
**Files:** `~/.config/opencode/tools/tests/generate_runbook.test.ts`, `~/.config/opencode/tools/tests/validate_runbook.test.ts`
**Action:** Update existing tests to the new schema where needed; add generate tests (batch block, HIGH block, on-hold block, preamble present, 5-col summary, legacy adapter) and validate tests (batch-level claim satisfies member stories, Read-list + Execute not flagged duplicate, on-hold claim required, canonical file valid). Add a fixture-based dual-oracle test.
**Verification:** `node --experimental-strip-types --test` on both files — all pass.

### Step 12: Regenerate + dual conformance check
**Files:** none (verification only)
**Action:** Regenerate `runbook-documentation-cleanup-GENERATED.md` from structured input mirroring the original (3 batches A/B/C, on-hold PA-68581, wave 3). Validate both the original and the regenerated file.
**Verification:** Original → `{ valid: true, errors: [] }`; Generated → `{ valid: true, errors: [] }`; generated is structurally equivalent (3 batches, 15 stories, on-hold block, all sections) to the original.

## Testing & Validation

- Unit (generate): legacy adapter; HIGH block claim+checkpoint; batch block single claim + per-story Execute; on-hold block; preamble/workflow-doc presence; 5-column summary counts; missing-field error path.
- Unit (validate): batch-level claim satisfies members; duplicate tolerance for Read/Write-plans aggregation; on-hold claim required; HIGH block requires own claim; canonical file valid.
- Integration (dual oracle): regenerate documentation-cleanup and validate; validate the original hand-written file — both clean.
- Regression: all pre-existing passing tests still pass after schema extension.

## Risks & Open Questions

- **Parser flattening:** the validator must reconstruct block grouping from raw mdast because `ParsedRunbook.waves` loses H3 grouping. Mitigation: `groupBlocksByH3` walks raw mdast directly (parser unchanged).
- **Backward compatibility:** existing runbooks generated with the legacy flat schema must still generate. Mitigation: legacy adapter (Step 2); keep legacy tests.
- **Fidelity vs canonical:** the regenerated file will use canonical phrasing/gate wording, not byte-identical to the original's ad-hoc text. Accepted per decision doc (structural + content fidelity, not byte-identity).
- **Duplicate-id semantics:** batch "Read all issues" lists intentionally repeat keys; the validator must exclude aggregation items to avoid false positives. Covered by Step 9 + tests.
