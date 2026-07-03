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

## Design Notes

- **`tool.schema` nested Zod types:** Confirmed available — `tool.schema` is the full `z` Zod namespace. However, the `waves` input is accepted as a JSON string to simplify the tool call interface for agents using the OpenCode tool UI.
- **Checkpoint summary count matching:** The `validate_runbook` summary count check uses `/🔴\s*(?:HIGH|INDIVIDUAL PLAN CHECKPOINT)/gi`. H3 headings use format `### 🔴 ARC-XXXX — title` (not `### 🔴 HIGH — ARC-XXXX`) to prevent H3 headings from contributing to the count, ensuring the table's HIGH count equals exactly `sum(checkpoint_count ?? 1)` for HIGH stories.
- **Wave gate requirement:** Each wave section includes a `### Wave N Gate` subsection with a `🟢 WAVE GATE` sub-item. This satisfies `wave.missing_gate` validation (astWalker classifies `GATE` keyword as a low checkpoint).
- **Parser-First principle (inverse):** Generated markdown is validated by writing to a temp file and calling `parseRunbook()` before writing to the final output path — if parsing fails, the tool returns an error and cleans up.
