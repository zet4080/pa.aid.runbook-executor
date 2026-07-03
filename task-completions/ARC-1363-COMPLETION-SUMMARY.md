# ARC-1363 Completion Summary — `generate_runbook` OpenCode Tool

**Story:** ARC-1363 — Implement `generate_runbook` OpenCode tool
**Lane:** agent-tools
**Date:** 2026-07-03
**Status:** ✅ Complete

---

## Acceptance Criteria Verification

### AC1: Valid input writes file and returns `{ success: true, path, story_count, wave_count }`

**Evidence:** Test 1 (`generate_runbook.test.ts`) calls `generateRunbook()` with a single-wave, single-story input. Asserts `result.success === true`, `story_count === 1`, `wave_count === 1`, and `existsSync(result.path)`.

**Result:** ✅ PASS

---

### AC2: `priority: "🔴"` story block includes `🔴 INDIVIDUAL PLAN CHECKPOINT` sub-item

**Evidence:** Test 2 calls `buildRunbookMarkdown()` with a HIGH story. Asserts output contains `INDIVIDUAL PLAN CHECKPOINT` and `🔴 INDIVIDUAL PLAN CHECKPOINT`.

**Result:** ✅ PASS

---

### AC3: Dependencies render as `🔒 Blocked by: ARC-XXXX` sub-items

**Evidence:** Test 3 calls `buildRunbookMarkdown()` with `dependencies: ["ARC-0001", "ARC-0002"]`. Asserts output contains both `🔒 Blocked by: ARC-0001` and `🔒 Blocked by: ARC-0002`.

**Result:** ✅ PASS

---

### AC4: Unchecked story includes `🔒 Claimed: —` sub-item

**Evidence:** Test 4 calls `buildRunbookMarkdown()` with `status: "pending"`. Asserts output contains `🔒 Claimed: —`.

**Result:** ✅ PASS

---

### AC5: `checkpoint_count` reflected correctly in checkpoint summary table

**Evidence:** Test 5 verifies `computeCheckpointCounts()` returns `high: 3` for two HIGH stories (`checkpoint_count: 2` and default 1). Also asserts `buildRunbookMarkdown()` output contains `| HIGH | 3 |` in the summary table.

**Result:** ✅ PASS

---

### AC6: Missing required field returns `{ success: false, errors }` without writing file

**Evidence:** Test 6 calls `generateRunbook()` with `epic_key: ""`. Asserts `result.success === false`, `result.errors.length > 0`, and `!existsSync(outputPath)`.

**Result:** ✅ PASS

---

### AC7: Generated file passes `validate_runbook` with `{ valid: true, errors: [] }`

**Evidence:** Test 7 generates a full runbook (2 waves, 3 stories of varying priority) then calls `validateRunbook(result.path)`. Asserts `validation.valid === true` and `validation.errors` is `[]`.

**Result:** ✅ PASS

---

## Artifacts

| Artifact | Path | Status |
|----------|------|--------|
| Tool implementation | `~/.config/opencode/tools/generate_runbook.ts` | ✅ Created |
| Test suite | `~/.config/opencode/tools/tests/generate_runbook.test.ts` | ✅ Created (7/7 tests pass) |
| Skill update | `~/.config/opencode/skills/generate-lane-runbooks/SKILL.md` | ✅ Updated — added `generate_runbook` tool section |
| Allowlist update | `~/.config/opencode/opencode.json` | ✅ Updated — `generate_runbook` added to `build`, `senior-coder`, `plan` agents |
| Implementation plan | `implementation_plans/agent-tools/ARC-1363-implementation-plan.md` | ✅ Complete |
| Completion summary | `task-completions/ARC-1363-COMPLETION-SUMMARY.md` | ✅ This file |

---

## Test Run Evidence

```
node --experimental-strip-types --test ~/.config/opencode/tools/tests/generate_runbook.test.ts

✔ 1. Valid minimal input writes file and returns { success: true } (15.245709ms)
✔ 2. HIGH story includes INDIVIDUAL PLAN CHECKPOINT sub-item (0.139468ms)
✔ 3. Story with dependencies renders Blocked-by sub-items (0.091344ms)
✔ 4. Unchecked story has 🔒 Claimed: — sub-item (0.093682ms)
✔ 5. computeCheckpointCounts returns correct HIGH count for multiple HIGH stories (0.13705ms)
✔ 6. Missing epic_key returns { success: false } without creating file (0.14415ms)
✔ 7. Generated file passes validate_runbook with { valid: true, errors: [] } (10.254015ms)

tests 7 | pass 7 | fail 0
```

Regression check (`validate_runbook.test.ts`): 9/9 pass — no regressions.

---

## Post-Implementation Fix (2026-07-03)
- Fixed wave gate format: promoted `🟢 WAVE GATE` from sub-item to top-level list item so `validate_runbook` correctly detects it

---

## Design Notes

- **`tool.schema` nested Zod types:** Confirmed available — `tool.schema` is the full `z` Zod namespace. However, the `waves` input is accepted as a JSON string to simplify the tool call interface for agents using the OpenCode tool UI.
- **Checkpoint summary count matching:** The `validate_runbook` summary count check uses `/🔴\s*(?:HIGH|INDIVIDUAL PLAN CHECKPOINT)/gi`. H3 headings use format `### 🔴 ARC-XXXX — title` (not `### 🔴 HIGH — ARC-XXXX`) to prevent H3 headings from contributing to the count, ensuring the table's HIGH count equals exactly `sum(checkpoint_count ?? 1)` for HIGH stories.
- **Wave gate requirement:** Each wave section includes a `### Wave N gate` subsection with `🟢 WAVE GATE` as a **top-level list item**. This satisfies `wave.missing_gate` validation (astWalker classifies `GATE` keyword as a low checkpoint; sub-items are not scanned by the gate check).
- **Parser-First principle (inverse):** Generated markdown is validated by writing to a temp file and calling `parseRunbook()` before writing to the final output path — if parsing fails, the tool returns an error and cleans up.

## Post-Implementation Fix 2 (2026-07-03)
- Added standard workflow sub-items to every story block: Lint/tests, Write completion summary, Commit
- HIGH stories additionally get: Write implementation plan, INDIVIDUAL PLAN CHECKPOINT sub-items
- lane_name threaded through buildStoryBlock() for correct path generation

---

## Option 1 Rework — Canonical Batch Model (2026-07-03)

**Trigger:** Testing against the real hand-written runbook `runbook-documentation-cleanup.md` (184 lines) revealed the flat `waves[].stories[]` schema could not express the canonical format. Approved via `decisions/ARC-1363-batch-model-approach-decision.md` (Option 1). See rewritten plan `implementation_plans/agent-tools/ARC-1363-implementation-plan.md`.

### generate_runbook — expanded to canonical structure
- New nested schema: waves contain `high_stories[]` (individual 9-step blocks) and `batches[]` (shared claim + shared Read/Write-plans + one BATCH PLAN CHECKPOINT + per-story Execute items + `☑ all complete`). Top-level `on_hold[]`, plus metadata fields (`feature_goal`, `repos`, `waves_participated`, `depends_on`, `skills`, `out_of_checklist_note`).
- Emits full canonical preamble: Load/Stop/Resume blockquote, `## Checkpoint model` table, `## HIGH story workflow`, `## MEDIUM/LOW batch workflow`, per-wave gate blockquote, on-hold conditional blocks, `### Wave N gate` checklist, On-Hold Register table, and the 5-column Checkpoint Summary.
- Backward-compatible: legacy flat `stories[]` input is auto-adapted (🔴 → HIGH blocks; 🟡/🟢 → batches).

### validate_runbook — batch-aware
- New `groupBlocksByH3()` reconstructs block structure from raw mdast (parser unchanged).
- `story.missing_claimed`: one claim per HIGH/batch/on-hold **block** (not per story). Claim marker now matched as a `Claimed:` label — fixes a false positive when a story title contained the word "claimed".
- `step.duplicate_id`: only flags a key defined as a story in two blocks; batch "Read all issues"/"Write all plans" aggregation lists excluded.
- `wave.missing_gate` recognizes `### Wave N gate` blocks; `summary.count_mismatch` accepts the 5-column canonical table.

### Dual conformance oracle — both clean
- **Original** `runbook-documentation-cleanup.md` → `{ valid: true, errors: [], warnings: [] }` (previously 18 false errors).
- **Regenerated** `runbook-documentation-cleanup-GENERATED.md` (184 lines, 3 batches, 15 stories, on-hold PA-68581, all sections) → `{ valid: true, errors: [], warnings: [] }`, structurally faithful to the original.

### Test evidence
```
node --experimental-strip-types --test tests/generate_runbook.test.ts tests/validate_runbook.test.ts
tests 22 | pass 22 | fail 0
```
- generate_runbook: 10 tests (minimal, HIGH block, deps, batch single-claim, per-wave counts, missing-field, preamble/workflow docs, on-hold block, legacy adapter, conformance oracle).
- validate_runbook: 12 tests (metadata, wave.exists, duplicate, HIGH missing-claimed, feature_goal, wave.heading, file-not-found, batch-level claim satisfies members, Read-list not duplicate, batch missing-claim flagged).
