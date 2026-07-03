# ARC-1362 Implement `validate_runbook` OpenCode tool - Implementation Plan

**Issue:** `issues/agent-tools/ARC-1362-tool-validate-runbook.md`
**Completion Summary:** `task-completions/ARC-1362-COMPLETION-SUMMARY.md` (TBD)
**Approach:** Single approach — issue specifies exact structure; pattern established by ARC-1348/ARC-1349
**Owner:** build
**Date:** 2026-07-03

---

## Scope & Alignment

This plan covers creating `~/.config/opencode/tools/validate_runbook.ts` — a read-only OpenCode custom tool that validates a runbook markdown file against both the `pa.aid.conductor.ts` parser structural rules and the `generate-lane-runbooks` skill quality checklist.

AC mapping:
- AC 1 (valid runbook → `{ valid: true, errors: [], warnings: [] }`) → Steps 2–4
- AC 2 (missing `**Lane:**` field → error `metadata.lane`) → Step 2
- AC 3 (H2 not matching `## Wave N` → warning `wave.heading`) → Step 3
- AC 4 (duplicate `ARC-XXXX` key → error `step.duplicate_id`) → Step 3
- AC 5 (unchecked story missing `🔒 Claimed:` → error `story.missing_claimed`) → Step 3
- AC 6 (HIGH story missing INDIVIDUAL PLAN CHECKPOINT → warning `story.missing_checkpoint`) → Step 3
- AC 7 (summary table missing/count mismatch → warning `summary.count_mismatch`) → Step 4
- AC 8 (file not found → `{ valid: false, errors: [{ rule: "file", ... }] }`) → Step 1

---

## Assumptions & Dependencies

- `@opencode-ai/plugin` is available at `~/.config/opencode/node_modules/@opencode-ai/plugin/` (confirmed ARC-1348)
- The tool uses **pure line/regex scanning** — no dependency on `./lib/parser.ts` (issue constraint: "no dependency on the conductor parser itself")
- All regex patterns for parser rules are derived directly from `astWalker.ts` source logic documented in the issue
- `node --experimental-strip-types --test` is available for running tests (confirmed pattern from ARC-1348)
- Allowlist entry for `validate_runbook` must be added to `opencode-sync-config.sh` for `plan`, `build`, and `senior-coder` agents (per technical-context.md §2b)
- The tool is read-only — no file writes, no git operations

---

## Implementation Steps

### Step 1: Create `validate_runbook.ts` tool file

**File:** `~/.config/opencode/tools/validate_runbook.ts`

**Action:**

Create the tool file following the standard pattern (`export default tool({...})`). Define the `RuleViolation` interface: `{ rule: string; message: string; line?: number }`. Define the `ValidationResult` interface: `{ valid: boolean; errors: RuleViolation[]; warnings: RuleViolation[]; summary: string }`.

The tool's `args` schema has one field: `runbook_path: tool.schema.string()`.

The `execute` function:
1. Attempts to read the file with `Bun.file(runbook_path).text()`. If the file does not exist/cannot be read, returns `{ valid: false, errors: [{ rule: "file", message: "File not found: {runbook_path}" }], warnings: [], summary: "❌ File not found." }` immediately.
2. Splits content into lines array (1-indexed by offset).
3. Delegates to `validateRunbook(content, lines)` helper (defined in same file).
4. Returns `JSON.stringify(result, null, 2)`.

**Verification:** File exists at `~/.config/opencode/tools/validate_runbook.ts`; TypeScript syntax valid (no parse errors when loaded).

---

### Step 2: Implement metadata validation rules (errors)

**File:** `~/.config/opencode/tools/validate_runbook.ts`

**Action:**

In helper function `validateMetadata(lines: string[]): RuleViolation[]`, implement these error rules by scanning all lines:

- `metadata.title`: Check that exactly one line matches `/^# .+/` and contains an `ARC-` key. If absent, push error.
- `metadata.epic`: Check that at least one line matches `/\*{0,2}Epic:\*{0,2}\s*\S/i`. If absent, push error.
- `metadata.lane`: Check that at least one line matches `/\*{0,2}Lane:\*{0,2}\s*\S/i`. If absent, push error.
- `metadata.depends_on`: Check that at least one line matches `/\*{0,2}Depends on:\*{0,2}\s*\S/i`. If absent, push error.

Report the line number of the H1 for `metadata.title`; for paragraph fields, use the first matched line number or 0 if absent.

**Verification:** Unit test with a runbook missing `**Lane:**` returns exactly one error with `rule: "metadata.lane"`.

---

### Step 3: Implement wave, step, and story validation rules (errors + warnings)

**File:** `~/.config/opencode/tools/validate_runbook.ts`

**Action:**

In helper function `validateWavesAndSteps(lines: string[]): { errors: RuleViolation[]; warnings: RuleViolation[] }`, implement:

**Errors:**
- `wave.exists`: Count lines matching `/^## Wave \d+/i`. If count is 0, push error.
- `step.duplicate_id`: Collect all ARC keys from top-level story lines — lines matching `/^- \[[ x\]] \*\*([A-Z]+-\d+)\*\*/`. Track seen keys in a Set; on second occurrence of same key, push error naming the duplicate key and its line number.
- `story.missing_claimed`: For each unchecked story line matching `/^- \[ \] \*\*([A-Z]+-\d+)\*\*/`, scan the immediately following indented sub-items (lines starting with `  - `) until the next top-level item. If no sub-item matches `/claimed/i`, push error.

**Warnings:**
- `wave.heading`: For every line matching `/^## /` where the heading text does NOT match `/^wave\s+\d+/i` AND does NOT match `/^(on.hold|checkpoint|summary)/i`, push warning with the heading text.
- `wave.missing_gate`: For each wave section, check whether any item in the wave matches `/gate/i`. If none, push warning.
- `story.missing_checkpoint`: For each story item where the story's H3 heading (the nearest preceding `### ` line) contains `🔴`, check whether any sub-item matches `/INDIVIDUAL PLAN CHECKPOINT/i`. If absent, push warning.

Note: The H3 heading detection for `story.missing_checkpoint` tracks the most recent `### ` line before the story's `- [ ]` item.

**Verification:** Unit tests for each rule — duplicate ARC key, unchecked story without claimed sub-item, H2 not matching wave pattern, HIGH story without INDIVIDUAL PLAN CHECKPOINT.

---

### Step 4: Implement quality/structure warnings (generate-lane-runbooks checklist)

**File:** `~/.config/opencode/tools/validate_runbook.ts`

**Action:**

In helper function `validateStructure(lines: string[]): RuleViolation[]`, implement:

- `metadata.feature_goal`: Check that at least one line matches `/\*{0,2}Feature goal:\*{0,2}\s*\S/i`. If absent, push warning.
- `metadata.repos`: Check that at least one line matches `/\*{0,2}Repos:\*{0,2}\s*\S/i`. If absent, push warning.
- `metadata.skills`: Check that at least one line matches `/\*{0,2}Skills to load at start:\*{0,2}\s*\S/i`. If absent, push warning.
- `structure.missing_onhold`: Check that at least one line contains `On-Hold` (case-insensitive). If absent, push warning.
- `structure.missing_summary`: Check that the last 60 lines contain a markdown table line with `| HIGH |` or `| Batch |` or text matching `/checkpoint summary/i`. If absent, push warning.
- `summary.count_mismatch`: If the summary table is present, extract its HIGH/batch/gate counts by parsing table cells. Count actual HIGH checkpoints in body (lines matching `INDIVIDUAL PLAN CHECKPOINT`), batch checkpoints (lines matching `BATCH PLAN CHECKPOINT`), and wave gates (lines matching `GATE`). If any count differs, push warning.

**Verification:** Unit test with a runbook that has a summary table whose HIGH count is off by one triggers `summary.count_mismatch` warning.

---

### Step 5: Compose `validateRunbook` and finalize result

**File:** `~/.config/opencode/tools/validate_runbook.ts`

**Action:**

Implement `validateRunbook(content: string): ValidationResult`:
1. Split content into `lines` array.
2. Call `validateMetadata(lines)` → metadata errors array.
3. Call `validateWavesAndSteps(lines)` → `{ errors, warnings }`.
4. Call `validateStructure(lines)` → structure warnings array.
5. Combine: `errors = [...metadataErrors, ...waveErrors]`, `warnings = [...waveWarnings, ...structureWarnings]`.
6. Compute `valid = errors.length === 0`.
7. Compute `summary`: if valid, `"✅ Runbook is valid."`, else `"❌ {errors.length} error(s), {warnings.length} warning(s)."`.
8. Return `{ valid, errors, warnings, summary }`.

**Verification:** Call with a fully valid runbook fixture → `{ valid: true, errors: [], warnings: [] }`.

---

### Step 6: Write unit tests

**File:** `~/.config/opencode/tools/tests/validate_runbook.test.ts`

**Action:**

Create test file following the same pattern as `tests/runbook_find_next_story.test.ts`:
- Import `node:test` and `node:assert/strict`
- Import `validateRunbook` (named export) from `../validate_runbook.ts`
- Write one test per acceptance criterion (8 tests total):
  1. Valid runbook → `valid: true`, `errors: []`, `warnings: []`
  2. Missing `**Lane:**` → one error with `rule: "metadata.lane"`
  3. H2 not matching `## Wave N` → one warning with `rule: "wave.heading"`
  4. Duplicate ARC key → one error with `rule: "step.duplicate_id"` naming the key
  5. Unchecked story without `🔒 Claimed:` → one error with `rule: "story.missing_claimed"`
  6. HIGH story without INDIVIDUAL PLAN CHECKPOINT → one warning with `rule: "story.missing_checkpoint"`
  7. Summary table count mismatch → one warning with `rule: "summary.count_mismatch"`
  8. File not found → `{ valid: false, errors: [{ rule: "file" }] }`

Run command: `node --experimental-strip-types --test ~/.config/opencode/tools/tests/validate_runbook.test.ts`

**Verification:** All 8 tests pass.

---

### Step 7: Add `validate_runbook` to agent allowlists

**File:** `/repos/pa.aid.wsl-setup.sh/components/opencode/opencode-sync-config.sh`

**Action:**

In the embedded Python `"agent"` dict, add `"validate_runbook"` to the tool lists for:
- `build` agent
- `senior-coder` agent
- `plan` agent (per technical-context.md §2b)

Also update `~/.config/opencode/opencode.json` directly with the same addition for immediate use (without waiting for the wsl-setup sync run).

**Verification:** `grep -r "validate_runbook" /repos/pa.aid.wsl-setup.sh/components/opencode/opencode-sync-config.sh` shows the new entry; `grep "validate_runbook" ~/.config/opencode/opencode.json` shows it in all three agent allowlists.

---

## Testing & Validation

Run the full test suite:
```
node --experimental-strip-types --test ~/.config/opencode/tools/tests/validate_runbook.test.ts
```

Additionally, run the smoke test against a real runbook file:
```
# Via OpenCode tool invocation in a session
validate_runbook({ runbook_path: "/repos/pa.aid.runbook-executor/docs/plans/runbook-core-infrastructure.md" })
```
Expected: returns JSON with `valid`, `errors`, `warnings`, `summary` fields; no exceptions thrown.

Run existing tool tests to verify no regression:
```
node --experimental-strip-types --test ~/.config/opencode/tools/tests/runbook_find_next_story.test.ts
node --experimental-strip-types --test ~/.config/opencode/tools/tests/runbook_claim_story.test.ts
```

---

## Risks & Open Questions

- **Summary table format variability**: Runbooks may format the checkpoint summary table differently. The `summary.count_mismatch` check must be lenient — if the table is present but its format cannot be parsed, emit a warning about unparseable format rather than throwing.
- **`story.missing_checkpoint` H3 detection**: The "nearest preceding `### ` line" heuristic assumes stories are organized under H3 headings. Stories not preceded by an H3 with 🔴 will never trigger this warning — which is the correct behavior (no false positives).
- **On-Hold pattern**: `structure.missing_onhold` checks for "On-Hold" text anywhere in the file. This is intentionally broad — a dedicated section heading, a table row, or a comment all count.
- **`wave.heading` expected exclusions**: The warning suppresses headings matching `/^(on.hold|checkpoint|summary)/i` — this list may need expansion if runbooks use other structural H2s. If so, update the exclusion regex without changing the rule logic.

---

After writing the file, commit and push to `pa.aid.runbook-executor` main:

```bash
git -C /repos/pa.aid.runbook-executor add implementation_plans/agent-tools/ARC-1362-implementation-plan.md
git -C /repos/pa.aid.runbook-executor commit -m "docs(agent-tools): add ARC-1362 implementation plan"
git -C /repos/pa.aid.runbook-executor push
```

Report back the commit hash and confirm push succeeded.