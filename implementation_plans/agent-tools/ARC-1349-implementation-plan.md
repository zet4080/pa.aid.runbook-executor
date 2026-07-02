# ARC-1349 Implement `runbook_claim_story` OpenCode tool — Implementation Plan

**Issue:** `issues/agent-tools/ARC-1349-tool-runbook-claim-story.md`
**Completion Summary:** `task-completions/ARC-1349-COMPLETION-SUMMARY.md` (to be created)
**Approach:** AST-locate + raw line patch (hybrid)
**Owner:** agent
**Date:** 2026-07-02

---

## Scope & Alignment

This plan delivers a single TypeScript file at `~/.config/opencode/tools/runbook_claim_story.ts` that is auto-discovered by OpenCode and exposes one tool named `runbook_claim_story`. The tool uses `parseRunbook()` from `./lib/parser.ts` to structurally validate story presence and sub-item state, then patches the `🔒 Claimed:` sub-item in-place using a story-key-anchored raw text replacement. It then stages, commits, and pushes the change via Bun shell.

AC mapping:

| AC | Covered by |
|----|-----------|
| Unchecked `🔒 Claimed:` sub-item → patched to `[x]` with UTC timestamp, committed, pushed | Steps 3–4: AST guards + story-anchored raw patch + `Bun.$` git commands |
| Already-claimed sub-item → return `"Story {KEY} is already claimed"`, no changes | Step 3: AST pre-flight guard on `subStep.checked` state |
| No `🔒 Claimed:` sub-item found → return `"No Claimed sub-item found for story {KEY}"` | Step 3: AST pre-flight guard on sub-item presence |
| Push failure → preserve local commit, return error with retry instructions | Step 3: separate try/catch around push, distinct return message |
| Tool registered in allowlist for `build` and `senior-coder` agents | Step 5: confirm presence in `opencode-sync-config.sh` and sync to `opencode.json` |

---

## Assumptions & Dependencies

- `~/.config/opencode/tools/lib/parser.ts`, `./lib/types.ts`, `./lib/astWalker.ts` are live (confirmed ARC-1348). `parseRunbook()` is imported from `./lib/parser.ts`.
- `ParsedRunbook` (from `./lib/types.ts`) exposes `waves[] → steps[] → subSteps[]`. `RunbookSubStep` fields are `{ label: string; checked: boolean }` — **no line number or character offset**. The AST is used for structural validation only; the actual file patch is performed on the raw text.
- The raw patch is anchored to the story key: the regex targets the `🔒 Claimed:` line that appears as a sub-item inside the story's list block. The regex pattern matches the story's checked/unchecked top-level item line followed (within the same block) by the `🔒 Claimed:` sub-item, using a two-capture approach described in Step 2.
- `@opencode-ai/plugin` is available at `~/.config/opencode/node_modules/@opencode-ai/plugin/` (confirmed ARC-1348).
- The OpenCode Bun runtime provides `Bun.$\`command\`` for shell execution. `child_process` is not used.
- Tools in `~/.config/opencode/tools/` are auto-discovered — no `"plugin"` array change in `opencode.json` is needed (confirmed ARC-1348).
- `opencode-sync-config.sh` already lists `runbook_claim_story` in the `build` and `senior-coder` allowlists (to be confirmed in Step 5).
- UTC timestamp format: `new Date().toISOString().slice(0, 16).replace('T', ' ')` → `YYYY-MM-DD HH:MM`.
- `pa.aid.config.md` tools directory is the source of truth at `/home/zimmermann/.config-src/pa.aid.config.md/tools/`. `~/.config/opencode/tools/` is a symlink pointing there.

---

## Implementation Steps

### Step 1: Parse and validate via AST

**Files:** `~/.config/opencode/tools/runbook_claim_story.ts` (new)
**Action:** Write a helper function `validateClaim(runbook_path: string, story_key: string): { status: 'ok' | 'not_found' | 'no_sub_item' | 'already_claimed' }` that:

1. Calls `parseRunbook(runbook_path)` (imported from `./lib/parser.ts`).
2. Iterates `parsed.waves → steps` to find the step whose `id` ends with the `story_key` (accounting for the namespace prefix applied by `parseRunbook`).
3. If no matching step is found, returns `{ status: 'not_found' }`.
4. Searches `step.subSteps` for a sub-item whose `label` matches `/claimed/i` and contains `🔒`.
5. If no such sub-item exists, returns `{ status: 'no_sub_item' }`.
6. If the sub-item exists and `subStep.checked === true`, returns `{ status: 'already_claimed' }`.
7. Otherwise returns `{ status: 'ok' }`.

This function uses the AST's hierarchical structure to guarantee that story lookup is key-scoped (not a flat text search). The sub-item state check is structurally authoritative.

**Verification:** Unit tests in Step 5 cover all four status codes.

---

### Step 2: Patch the raw file using a story-key-anchored replacement

**Files:** `~/.config/opencode/tools/runbook_claim_story.ts`
**Action:** Write a helper function `patchClaimedLine(raw: string, story_key: string, lane: string, timestamp: string): string` that performs the in-place text replacement.

The function uses a two-step approach:

1. Locate the story's list block in the raw text using a pattern that matches the story key in the top-level item line: the regex `/^([ \t]*- \[[ x]\].*\*{0,2}` + escaped `story_key` + `/\*{0,2}.*\n(?:[ \t]+.*\n)*?[ \t]*- \[ \] (🔒 Claimed:.*)/m`. This anchors the `🔒 Claimed:` sub-item search to the story's own block, preventing false matches from other stories that also have `🔒 Claimed:` sub-items.
2. Replace only the `🔒 Claimed:` sub-item line's checkbox and trailing content: transform `- [ ] 🔒 Claimed:…` into `- [x] 🔒 Claimed: {lane} / {timestamp}`, preserving the leading indentation.

The replacement is performed with `String.prototype.replace` using a callback that reconstructs the full matched segment, patching only the `🔒 Claimed:` sub-item portion.

If the pattern does not match (safety check after the AST validated `ok`), the function returns `null` to signal a patch failure without writing.

**Verification:** Unit tests in Step 5 cover: correct indentation preserved, placeholder replacement, already-claimed guard (not reached since AST catches it first).

---

### Step 3: Implement the `execute` function with guards and git operations

**Files:** `~/.config/opencode/tools/runbook_claim_story.ts`
**Action:** The `execute(args)` function:

1. Calls `validateClaim(runbook_path, story_key)`.
2. **Guard — not found:** if `status === 'not_found'`, return `"No story found with key {story_key} in runbook"` immediately.
3. **Guard — no sub-item:** if `status === 'no_sub_item'`, return `"No Claimed sub-item found for story {story_key}"` immediately.
4. **Guard — already claimed:** if `status === 'already_claimed'`, return `"Story {story_key} is already claimed"` immediately.
5. Reads the runbook file with `await Bun.file(runbook_path).text()`.
6. Produces `timestamp` via `new Date().toISOString().slice(0, 16).replace('T', ' ')`.
7. Calls `patchClaimedLine(raw, story_key, lane, timestamp)`. If `null` is returned, returns `"Patch failed: could not locate Claimed sub-item in raw text for {story_key}"` without writing.
8. Writes the patched text with `await Bun.write(runbook_path, patched)`.
9. Runs `git add {runbook_path}` via `Bun.$\`git -C ${repo_path} add ${runbook_path}\``. On non-zero exit, returns stderr as error string.
10. Runs `git commit -m "chore({lane}): claim {story_key}"` via `Bun.$`. On non-zero exit, returns stderr.
11. Runs `git push` via `Bun.$\`git -C ${repo_path} push\``. On non-zero exit: local file and commit are already saved — returns `"Push failed for {story_key}. Local commit preserved. Re-run: git -C {repo_path} push"`.
12. On success, returns `"Claimed {story_key} as {lane} at {timestamp}. Committed and pushed."`.

Each git command is wrapped in its own try/catch so a push failure does not unwind the commit.

**Verification:** Guard return strings match AC wording exactly. Unit tests in Step 5 cover all branches.

---

### Step 4: Wire the tool export and args schema

**Files:** `~/.config/opencode/tools/runbook_claim_story.ts`
**Action:** Export using `export default tool({...})` with the following args schema:

- `runbook_path`: `tool.schema.string().describe("Absolute path to the runbook markdown file")`
- `story_key`: `tool.schema.string().describe("Story key to claim, e.g. ARC-1285")`
- `lane`: `tool.schema.string().describe("Lane name used in the claim timestamp, e.g. core-infrastructure")`
- `repo_path`: `tool.schema.string().describe("Absolute path to the git repository root containing the runbook file")`

Top-of-file imports:
```typescript
import { tool } from '@opencode-ai/plugin';
import { parseRunbook } from './lib/parser.ts';
import type { ParsedRunbook } from './lib/types.ts';
```

No other external imports. `Bun.file`, `Bun.write`, `Bun.$` are globally available in the runtime.

Export `validateClaim` and `patchClaimedLine` as named exports so the test file can import them directly.

**Verification:** File exports exactly one default `tool({...})` object. Running `node --experimental-strip-types --check ~/.config/opencode/tools/runbook_claim_story.ts` exits without errors.

---

### Step 5: Write unit tests

**Files:** `~/.config/opencode/tools/tests/runbook_claim_story.test.ts`
**Action:** Tests use `node:test` and `node:assert/strict` (same runner as `runbook_find_next_story.test.ts`). Import `validateClaim` and `patchClaimedLine` from `../runbook_claim_story.ts`. For `validateClaim`, write fixture runbooks to temp files (using `writeTmpMarkdown` helper pattern from `runbook_find_next_story.test.ts`). Tests must cover:

1. **validateClaim — happy path:** fixture with `- [ ] 🔒 Claimed:` under target story → `{ status: 'ok' }`.
2. **validateClaim — already claimed:** fixture with `- [x] 🔒 Claimed: lane / 2026-01-01 12:00` → `{ status: 'already_claimed' }`.
3. **validateClaim — no sub-item:** fixture with no `🔒 Claimed:` line under target story → `{ status: 'no_sub_item' }`.
4. **validateClaim — story not present:** fixture contains no matching story key → `{ status: 'not_found' }`.
5. **validateClaim — wrong story key:** fixture has `🔒 Claimed:` only under a different story → `{ status: 'not_found' }` for the requested key.
6. **patchClaimedLine — preserves indentation:** input raw text with `  - [ ] 🔒 Claimed:` indented two spaces under the story → output line is `  - [x] 🔒 Claimed: core-infrastructure / 2026-07-02 10:00` with identical leading whitespace.
7. **patchClaimedLine — replaces trailing placeholder:** input `  - [ ] 🔒 Claimed: _(fill in)_` → output `  - [x] 🔒 Claimed: {lane} / {timestamp}` (placeholder fully replaced, not appended).
8. **patchClaimedLine — story-scoped:** raw text with two stories both having `🔒 Claimed:` sub-items; patch targets only the correct story → second story's `🔒 Claimed:` line is unchanged.

Run command: `node --experimental-strip-types --test ~/.config/opencode/tools/tests/runbook_claim_story.test.ts`

**Verification:** All 8 tests pass with exit code 0.

---

### Step 6: Commit tool file to `pa.aid.config.md`

**Files:** `/home/zimmermann/.config-src/pa.aid.config.md/tools/runbook_claim_story.ts`, `/home/zimmermann/.config-src/pa.aid.config.md/tools/tests/runbook_claim_story.test.ts`
**Action:** The `~/.config/opencode/tools/` directory is a symlink to `/home/zimmermann/.config-src/pa.aid.config.md/tools/`. The files written in Steps 1–4 already live in the correct source location. Commit:

```bash
git -C /home/zimmermann/.config-src/pa.aid.config.md add tools/runbook_claim_story.ts tools/tests/runbook_claim_story.test.ts
git -C /home/zimmermann/.config-src/pa.aid.config.md commit -m "feat(agent-tools): add runbook_claim_story OpenCode tool (ARC-1349)"
```

**Verification:** `git -C /home/zimmermann/.config-src/pa.aid.config.md log --oneline -1` shows the commit.

---

### Step 7: Confirm allowlist in `opencode.json` and restart OpenCode

**Files:** `~/.config/opencode/opencode.json`, `/repos/pa.aid.wsl-setup.sh/components/opencode/opencode-sync-config.sh`
**Action:** Grep `opencode-sync-config.sh` for `runbook_claim_story` to confirm it is listed for `build` and `senior-coder` agents. Verify the live `~/.config/opencode/opencode.json` reflects this. If absent (because `opencode.json` was not regenerated since the script was updated), patch it directly using the same Python-dict structure already used for `runbook_find_next_story`.

Restart OpenCode to load the new tool.

**Verification:** After restart, a tool call to `runbook_claim_story` with invalid args returns a schema error, not "tool not found".

---

### Step 8: End-to-end smoke test

**Files:** Live runbook at `/repos/pa.aid.runbook-executor/docs/plans/runbook-agent-tools.md`
**Action:** Identify the first unclaimed story in the agent-tools runbook. Call `runbook_claim_story` with:
- `runbook_path`: absolute path to `runbook-agent-tools.md`
- `story_key`: the identified key
- `lane`: `agent-tools`
- `repo_path`: `/repos/pa.aid.runbook-executor`

Confirm the runbook file is updated in-place, `git log` shows the new commit, and `git push` succeeds.

**Verification:** `git -C /repos/pa.aid.runbook-executor log --oneline -1` shows `chore(agent-tools): claim {KEY}`. The `🔒 Claimed:` line in the runbook is `- [x] 🔒 Claimed: agent-tools / {timestamp}`.

---

## Testing & Validation

**Unit tests:**
Run `node --experimental-strip-types --test ~/.config/opencode/tools/tests/runbook_claim_story.test.ts`. All 8 scenarios must pass with exit code 0.

**Type check:**
Run `node --experimental-strip-types --check ~/.config/opencode/tools/runbook_claim_story.ts`. No errors.

**Allowlist check:**
Confirm `runbook_claim_story` appears in `~/.config/opencode/opencode.json` under `build` and `senior-coder` agent tool allowlists.

**End-to-end smoke test:**
Call the tool against the live agent-tools runbook via OpenCode session. Confirm the runbook file is patched, the commit appears in git log, and the push succeeds.

**Guard validation:**
Invoke the tool a second time on the same story after it is claimed. Confirm the return value is `"Story {KEY} is already claimed"` and no duplicate commit is made.

---

## Risks & Open Questions

1. **AST namespace prefix on step IDs:** `parseRunbook()` prepends the lane name as a namespace (e.g. `agent-tools__ARC-1349`). The `validateClaim` function must match the story key using a suffix comparison (`step.id` ends with `story_key` or equals `story_key` after stripping the namespace prefix). If the lane is null, the filename is used as the namespace. Mitigation: strip everything before and including `__` before comparing to `story_key`.

2. **Story-anchored regex boundary:** The regex in `patchClaimedLine` must stop at the next top-level list item to avoid spanning across stories. If two consecutive stories share indentation and neither has a blank-line separator, the regex may over-reach. Mitigation: test against the actual runbook files during Step 8; add a blank-line anchor to the pattern if needed.

3. **Multi-story runbooks with non-unique `🔒 Claimed:` text:** The story-anchored regex (test 8 in Step 5) verifies the patch is scoped correctly. If the runbook format deviates significantly, the patch returns `null` and the tool reports a patch failure without writing the file — no silent corruption.

4. **Concurrent claims from two agents:** If two agents call `runbook_claim_story` for the same story simultaneously, the second call reads after the first write and hits the `already_claimed` guard (AST sees `checked: true`). This is inherently a TOCTOU race; acceptable for the current single-agent workflow.

5. **Push failure leaving repo in diverged state:** After a successful local commit, a push failure leaves the local branch ahead of remote. The tool returns retry instructions. The agent must not retry the full claim (double commit risk). The push-only retry path is safe because file and commit already exist.
