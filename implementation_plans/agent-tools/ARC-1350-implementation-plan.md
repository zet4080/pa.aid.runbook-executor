# ARC-1350 Implement `runbook_check_step` OpenCode tool — Implementation Plan

**Issue:** `issues/agent-tools/ARC-1350-tool-runbook-check-step.md`
**Completion Summary:** `task-completions/ARC-1350-COMPLETION-SUMMARY.md` (TBD)
**Approach:** single approach — pattern established (AST-locate + line-number-targeted raw patch)
**Owner:** agent
**Date:** 2026-07-04

## Scope & Alignment

This plan delivers a single TypeScript file at `~/.config/opencode/tools/runbook_check_step.ts` that exposes one auto-discovered tool: `runbook_check_step`. The tool accepts a story key and a 1-based step number, locates the target sub-step checkbox in the runbook via the AST, patches the raw file using the line number from the AST node, then stages, commits, and pushes via Bun shell.

| AC | Step(s) |
|----|---------|
| Unchecked step checkbox → flipped to `[x]`, committed, pushed | Steps 1–3 |
| Already-checked step → no-op return `"Step {N} of {KEY} already checked"` | Step 1 (`already_checked` guard) |
| Step number not found → error `"Step {N} not found in story {KEY}"` | Step 1 (`step_not_found` guard) |
| Story key not found → error `"Story {KEY} not found in runbook"` | Step 1 (`story_not_found` guard) |
| Tool registered in allowlist for `build` and `senior-coder` agents | Step 7 |

## Assumptions & Dependencies

- `~/.config/opencode/tools/lib/parser.ts`, `./lib/types.ts`, `./lib/astWalker.ts` are live (confirmed ARC-1348/ARC-1349).
- `RunbookSubStep` has `{ label: string; checked: boolean; line: number }` — the `line` field is the 1-based mdast position, used for direct line indexing in the raw text patch.
- `step_number` is 1-based and indexes across ALL sub-steps (including the `🔒 Claimed:` item at index 0 → step_number=1). Callers must know this convention.
- `@opencode-ai/plugin` is available at `~/.config/opencode/node_modules/@opencode-ai/plugin/` (confirmed ARC-1348).
- `Bun.$`, `Bun.file()`, `Bun.write()` are globally available in the OpenCode runtime.
- `~/.config/opencode/tools/` is a symlink to `/home/zimmermann/.config-src/pa.aid.config.md/tools/` — files written there are automatically at both paths.
- No external dependency additions needed — the same `unified`/`remark-parse`/`remark-gfm` packages already installed for ARC-1348/ARC-1349 are reused.

## Implementation Steps

### Step 1: Parse and validate via AST — write `validateStep` helper
- File: `~/.config/opencode/tools/runbook_check_step.ts` (new)
- Function: `validateStep(runbook_path, story_key, step_number, parsed?): { status: 'ok' | 'story_not_found' | 'step_not_found' | 'already_checked', line?: number }`
- Logic:
  1. Call `parseRunbook(runbook_path)` (or use passed `parsed`)
  2. Find story step by matching `step.id` suffix against `story_key` (strip namespace prefix: everything before and including `__`)
  3. If not found: return `{ status: 'story_not_found' }`
  4. Access `step.subSteps[step_number - 1]` (0-based array index from 1-based param)
  5. If undefined: return `{ status: 'step_not_found' }`
  6. If `subStep.checked === true`: return `{ status: 'already_checked', line: subStep.line }`
  7. If `subStep.checked === false`: return `{ status: 'ok', line: subStep.line }`
- Verification: unit tests cover all 4 status codes

### Step 2: Write `patchStepLine` helper — line-number-targeted raw patch
- File: `~/.config/opencode/tools/runbook_check_step.ts`
- Function: `patchStepLine(content: string, line: number): string | null`
- Same pattern as `patchClaimedLine` in `runbook_claim_story.ts`:
  1. Split content by `\n`
  2. Index to `lines[line - 1]` (1-based to 0-based)
  3. If out of bounds: return null
  4. Replace `[ ]` with `[x]` on that line (preserve leading whitespace)
  5. Re-join and return
- Verification: unit tests cover indentation preservation and already-checked guard

### Step 3: Implement `execute` function
- Calls `parseRunbook(runbook_path)` once, shared with `validateStep`
- Guard responses per AC: "Story {KEY} not found in runbook", "Step {N} not found in story {KEY}", "Step {N} of {KEY} already checked" (no-op success)
- If ok: read file, call `patchStepLine`, write file
- git add → git commit ("chore({lane}): {KEY} step {step_number} done") → git push
- Separate try/catch for push so local commit is preserved on push failure
- Returns "Step {step_number} of {story_key} checked and committed." on success

### Step 4: Wire tool export and args schema
- `export default tool({...})` with args: `runbook_path` (string), `story_key` (string), `step_number` (integer/number), `lane` (string), `repo_path` (string)
- Imports: `@opencode-ai/plugin`, `./lib/parser.ts`, `./lib/types.ts`
- Export `validateStep` and `patchStepLine` as named exports for testing

### Step 5: Write unit tests
- File: `~/.config/opencode/tools/tests/runbook_check_step.test.ts`
- Use `node:test` + `node:assert/strict` (same pattern as `runbook_claim_story.test.ts`)
- Run: `node --experimental-strip-types --test ~/.config/opencode/tools/tests/runbook_check_step.test.ts`
- Test scenarios:
  1. `validateStep` — happy path: unchecked sub-step at step_number=2 → `{ status: 'ok' }`
  2. `validateStep` — already checked → `{ status: 'already_checked' }`
  3. `validateStep` — step_number out of range → `{ status: 'step_not_found' }`
  4. `validateStep` — story key not in runbook → `{ status: 'story_not_found' }`
  5. `patchStepLine` — flips `[ ]` to `[x]` on correct line
  6. `patchStepLine` — preserves leading whitespace
  7. `patchStepLine` — out-of-bounds line returns null
  8. `patchStepLine` — only patches target line, leaves other lines unchanged

### Step 6: Commit tool file to `pa.aid.config.md`
- Path: `/home/zimmermann/.config-src/pa.aid.config.md/tools/runbook_check_step.ts` and `tests/runbook_check_step.test.ts` (symlink target of `~/.config/opencode/tools/`)
- Commit: `feat(agent-tools): add runbook_check_step OpenCode tool (ARC-1350)`

### Step 7: Add to allowlist and verify
- Check `opencode-sync-config.sh` at `/repos/pa.aid.wsl-setup.sh/components/opencode/opencode-sync-config.sh` for `runbook_check_step`
- If absent: add to `build` and `senior-coder` allowlists
- Update live `~/.config/opencode/opencode.json` directly with the same pattern
- Restart OpenCode

### Step 8: End-to-end smoke test
- Call `runbook_check_step` with the live agent-tools runbook on a real story/step
- Confirm runbook file patched, git log shows commit, push succeeded

## Testing & Validation

**Unit tests:** `node --experimental-strip-types --test ~/.config/opencode/tools/tests/runbook_check_step.test.ts` — all 8 scenarios pass with exit code 0.

**Type check:** `node --experimental-strip-types --check ~/.config/opencode/tools/runbook_check_step.ts` — no errors.

**Allowlist check:** `runbook_check_step` appears in `~/.config/opencode/opencode.json` under `build` and `senior-coder` agent tool allowlists.

**End-to-end smoke test:** Call the tool on the live agent-tools runbook. Confirm the target sub-step checkbox is patched, `git log` shows `chore({lane}): {KEY} step {N} done`, and push succeeds.

**Guard validation:** Call the tool a second time on the same step. Confirm it returns `"Step {N} of {KEY} already checked"` with no duplicate commit.

## Risks & Open Questions

1. **`step_number` convention:** The plan uses 1-based indexing across ALL sub-steps (including Claimed item). This is the most natural mapping to the AST `subSteps[]` array. Documented explicitly in plan so callers know which index maps to which sub-step.
2. **AST namespace prefix:** Same as ARC-1349 — strip everything before and including `__` when matching `step.id` to `story_key`.
3. **Line number accuracy:** The AST provides `subStep.line` (1-based) from mdast position data. This is the same mechanism verified in ARC-1349 and ARC-1348. No new risk.
4. **Checkpoint sub-steps:** Some sub-steps are checkpoint items (🔴 INDIVIDUAL PLAN CHECKPOINT). The tool checks those off just like any other sub-step — no special handling needed.
