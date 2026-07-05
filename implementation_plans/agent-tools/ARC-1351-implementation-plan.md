# ARC-1351 Implementation Plan — `runbook_check_story` OpenCode Tool

**Story:** ARC-1351 — Implement `runbook_check_story` OpenCode tool
**Issue:** `issues/agent-tools/ARC-1351-tool-runbook-check-story.md`
**Lane:** agent-tools
**Date:** 2026-07-05

---

## Goal

Implement the `runbook_check_story` OpenCode tool that marks a story-level checkbox (`- [ ] **KEY** — Title`) as done (`- [x]`) in a runbook markdown file, then stages, commits, and pushes the change via git.

---

## Phase 0: Exploration Findings

Key findings from exploring the reference implementations:

- **`RunbookStep.line`** — `types.ts` exposes `line: number` (1-based) on `RunbookStep` directly. This is the line of the story-level checkbox (`- [ ] **KEY** — Title`), distinct from `RunbookSubStep.line` used by `runbook_check_step`.
- **`runbook_check_step.ts` pattern** — parses once, passes `ParsedRunbook` to `validateStep()`, then uses `subStep.line` for a direct line replacement in `patchStepLine()`. No regex re-scan. This same pattern applies here using `step.line` instead.
- **`runbook_claim_story.ts` pattern** — identical structure: parse once, validate, locate line via AST, patch, git add → commit → push. Also uses `Bun.$"..."`` for git commands.
- **Test pattern** — `node:test` runner (not bun), imports helpers directly from the tool file (`import { validateStory, patchStoryLine } from '../runbook_check_story.ts'`), uses `writeTmpMarkdown()` helper and `makeRunbook()` fixture factory. Tests live in `~/.config/opencode/tools/tests/`.
- **Tool export pattern** — `export default tool({...})` with `execute` function reference. Single tool per file.
- **Tool file location** — `~/.config/opencode/tools/runbook_check_story.ts` (global tools dir, auto-discovered).
- **`runbook_check_story.ts` does NOT exist yet** — confirmed by directory listing of `~/.config/opencode/tools/`.
- **Namespace stripping** — story keys in `step.id` may be prefixed (`agent-tools__ARC-1351`); strip using `step.id.split('__').pop()`.
- **`step.checked`** — `RunbookStep.checked` reflects the story-level checkbox state. Use this for idempotency check.
- **Allowlist** — tools are managed in `opencode-sync-config.sh` in `pa.aid.wsl-setup.sh`; also update `~/.config/opencode/opencode.json` directly for immediate use on current machine.
- **Tool is committed to `pa.aid.config.md`** — the tools directory is a git repo tracked by `pa.aid.config.md`, not `pa.aid.conductor.ts`.

---

## Phase 1: Implementation

### Step 1: Create worktree for ARC-1351

```bash
git -C /repos/pa.aid.conductor.ts worktree add /repos/ARC-1351 -b ARC-1351 runbook-executor
```

> Note: The tool file itself lives in `~/.config/opencode/tools/` (tracked by `pa.aid.config.md`), not in the worktree. The worktree is created for any companion code changes in `pa.aid.conductor.ts` if needed. Since this tool requires no conductor changes, the worktree creation may be skipped and work done directly in `~/.config/opencode/tools/`.

### Step 2: Write `runbook_check_story.ts`

**File:** `~/.config/opencode/tools/runbook_check_story.ts` (new)

The tool follows the exact same structure as `runbook_check_step.ts` and `runbook_claim_story.ts`:

``
Header docblock → imports → Types section → validateStory() → patchStoryLine() → execute() → export default tool({...})
``

**`validateStory()` helper:
- Takes `runbook_path`, `story_key`, optional pre-parsed `ParsedRunbook`
- Returns `{ status: StoryStatus, line?: number }` where `StoryStatus = 'ok' | 'not_found' | 'already_checked'`
- Iterates `runbook.waves[].steps[]`, strips namespace prefix, matches `bareId === story_key`
- Uses `step.checked` for idempotency check
- Uses `step.line` for the return line number
- Returns `'not_found'` if no match found

**`patchStoryLine()` helper:
- Takes `content: string`, `line: number` (1-based from AST)
- Splits on `\n`, index `line - 1`, replaces `[ ]` with `[x]`
- Returns `null` for out-of-bounds or `line <= 0`
- Returns patched string on success

**`execute()` function:
1. Parse runbook once via `parseRunbook(runbook_path)`
2. Call `validateStory(runbook_path, story_key, parsed)`
3. Handle `'not_found'` → return `"Story ${story_key} not found in runbook"` 
4. Handle `'already_checked'` → return `"Story ${story_key} already checked off"` 
5. Read file via `Bun.file(runbook_path).text()`
6. Call `patchStoryLine(raw, storyLine)`
7. Write via `Bun.write(runbook_path, patched)`
8. `git -C ${repo_path} add ${runbook_path}` (Bun shell)
9. `git -C ${repo_path} commit -m "chore(${lane}): close ${story_key}"` (Bun shell)
10. `git -C ${repo_path} push` (separate try/catch — local commit preserved on failure)
11. Return `"Story ${story_key} checked and committed."`

**Tool export:**
```typescript
export default tool({
  description: 'Mark a story-level checkbox as done ([x]) in a runbook markdown file, then stage, commit, and push the change via git.',
  args: {
    runbook_path: tool.schema.string().describe('Absolute path to the runbook markdown file'),
    story_key: tool.schema.string().describe('Story key, e.g. ARC-1351'),
    lane: tool.schema.string().describe('Lane name used in the commit message, e.g. agent-tools'),
    repo_path: tool.schema.string().describe('Absolute path to the git repository root containing the runbook file'),
  },
  execute,
});
```

### Step 3: Write tests

**File:** `~/.config/opencode/tools/tests/runbook_check_story.test.ts` (new)

8 test scenarios:

| # | Test | What it asserts |
|---|------|-----------------|
| 1 | `validateStory` — story not found | returns `status: 'not_found'` |
| 2 | `validateStory` — story already checked | returns `status: 'already_checked'`, `line` set |
| 3 | `validateStory` — story unchecked | returns `status: 'ok'`, `line` set and positive |
| 4 | `validateStory` — namespace-prefixed story key | strips prefix, matches correctly |
| 5 | `patchStoryLine` — flips `[ ]` to `[x]` on correct line | line contains `[x]`, not `[ ]` |
| 6 | `patchStoryLine` — preserves surrounding lines unchanged | only target line modified |
| 7 | `patchStoryLine` — out-of-bounds returns null | `line=0`, `line=-1`, `line=999` all return null |
| 8 | `patchStoryLine` — cross-story isolation | patching story A line does not affect story B checkbox |

**Fixture factory** (`makeRunbook`): produces a minimal runbook with two stories, first unchecked, second unchecked — same structure as `runbook_check_step.test.ts` fixture.

### Step 4: Update allowlist

Add `runbook_check_story` to the `build` and `senior-coder` agent allowlists:

1. Edit `opencode-sync-config.sh` in `pa.aid.wsl-setup.sh` — add `runbook_check_story` to both agents' tool lists
2. Edit `~/.config/opencode/opencode.json` directly — add `runbook_check_story` to `build` and `senior-coder` agent tool arrays for immediate use
3. Commit `opencode-sync-config.sh` change to `pa.aid.wsl-setup.sh`
4. Commit tool file to `pa.aid.config.md`
5. Restart OpenCode

### Step 5: Check off runbook step

After tool and tests are committed, check off this story's runbook step:

```bash
# Using the runbook_check_step tool (ARC-1350) via OpenCode,
# or via the runbook_check_step tool directly in implementation
```

---

## Phase 2: Tests

**Run command:**
```bash
node --experimental-strip-types --test ~/.config/opencode/tools/tests/runbook_check_story.test.ts
```

**Test scenarios:**

1. `validateStory` — `'not_found'` when story key absent from runbook
2. `validateStory` — `'already_checked'` when story checkbox is `[x]`, line returned
3. `validateStory` — `'ok'` when story checkbox is `[ ]`, line returned and positive
4. `validateStory` — namespace prefix stripped correctly (`agent-tools__ARC-1351` → `ARC-1351`)
5. `patchStoryLine` — `[ ]` → `[x]` on the exact specified line
6. `patchStoryLine` — all other lines unchanged after patch
7. `patchStoryLine` — returns `null` for `line <= 0` and out-of-bounds
8. `patchStoryLine` — only patches target story, not adjacent story checkboxes

---

## Phase 3: Local Review

```bash
# Type-check (Node strips types, catches syntax errors)
node --experimental-strip-types --check ~/.config/opencode/tools/runbook_check_story.ts

# Run tests
node --experimental-strip-types --test ~/.config/opencode/tools/tests/runbook_check_story.test.ts

# Verify tool appears in opencode.json allowlists for build and senior-coder agents
grep -A 20 '"build"' ~/.config/opencode/opencode.json | grep runbook_check_story
grep -A 20 '"senior-coder"' ~/.config/opencode/opencode.json | grep runbook_check_story
```

---

## Acceptance Criteria Mapping

| Acceptance Criterion | Implementing Step |
|----------------------|------------------|
| Story checkbox `[ ]` → `[x]` when found and unchecked | `patchStoryLine()` in Step 2; test #5 |
| Committed with message `chore({lane}): close {KEY}` | `execute()` git commit in Step 2 |
| Pushed via `git push` | `execute()` git push in Step 2 |
| No-op success when already `[x]`: `"Story {KEY} already checked off"` | `validateStory()` `already_checked` branch; test #2 |
| Error when key not found: `"Story {KEY} not found in runbook"` | `validateStory()` `not_found` branch; test #1 |
| Tool at `~/.config/opencode/tools/runbook_check_story.ts` | Step 2 file path |
| Inputs: `runbook_path`, `story_key`, `lane`, `repo_path` | Tool `args` in `export default tool({...})` |
| Uses `@opencode-ai/plugin` `tool()` helper | Import in Step 2 |
| Parser-first: uses `parseRunbook()`, no raw regex scan | `execute()` parse step; `validateStory()` via AST |

---

## Commit Messages

- **Tool implementation commit** (`pa.aid.config.md`): `feat(agent-tools): implement runbook_check_story OpenCode tool (ARC-1351)`
- **Allowlist update commit** (`pa.aid.wsl-setup.sh`): `feat(agent-tools): add runbook_check_story to agent allowlists (ARC-1351)`
- **Completion summary commit** (`pa.aid.runbook-executor`): `docs(agent-tools): ARC-1351 completion summary`
- **Runbook checkoff commit** (`pa.aid.runbook-executor`): `chore(agent-tools): close ARC-1351`
