# ARC-1349 Implement `runbook_claim_story` OpenCode tool — Implementation Plan

**Issue:** `issues/agent-tools/ARC-1349-tool-runbook-claim-story.md`
**Completion Summary:** `task-completions/ARC-1349-COMPLETION-SUMMARY.md` (TBD)
**Approach:** Single standalone TypeScript tool file — pattern established (ARC-1348)
**Owner:** agent
**Date:** 2026-07-02

---

## Scope & Alignment

This plan delivers a single TypeScript file at `~/.config/opencode/tools/runbook_claim_story.ts` that is auto-discovered by OpenCode and exposes one tool named `runbook_claim_story`. The tool patches the `🔒 Claimed:` sub-item in the runbook markdown in-place, then stages, commits, and pushes the change via Bun shell.

AC mapping:

| AC | Covered by |
|----|-----------|
| Unchecked `🔒 Claimed:` sub-item → patched to `[x]` with UTC timestamp, committed, pushed | Steps 3–4: regex patch + `Bun.$` git commands |
| Already-claimed sub-item → return `"Story {KEY} is already claimed"`, no changes | Step 3: pre-flight guard on checked state |
| No `🔒 Claimed:` sub-item found → return `"No Claimed sub-item found for story {KEY}"`, no changes | Step 3: pre-flight guard on presence |
| Push failure → preserve local commit, return error with retry instructions | Step 3: separate try/catch around push, distinct return message |
| Tool registered in allowlist for `build` and `senior-coder` agents | Step 5: already present in `opencode-sync-config.sh`; confirm and sync to `opencode.json` |

---

## Assumptions & Dependencies

- `~/.config/opencode/tools/runbook_find_next_story.ts` (ARC-1348) is live. The new tool follows the identical file/export pattern.
- `@opencode-ai/plugin` is available at `~/.config/opencode/node_modules/@opencode-ai/plugin/` (confirmed ARC-1348).
- The OpenCode Bun runtime provides `Bun.$\`command\`` for shell execution. `child_process` is not used.
- Tools in `~/.config/opencode/tools/` are auto-discovered — no `"plugin"` array change in `opencode.json` is needed (confirmed ARC-1348).
- `opencode-sync-config.sh` already lists `runbook_claim_story` in the `build` and `senior-coder` allowlists. The corresponding entry must be present in the live `~/.config/opencode/opencode.json`.
- The runbook markdown uses a consistent format for the `🔒 Claimed:` sub-item. The unchecked form is `- [ ] 🔒 Claimed:` (or `- [ ] 🔒 Claimed: _(fill in)_`). The tool targets the first sub-item line under the named story's top-level task item that matches `🔒 Claimed:`.
- `repo_path` is the absolute path to the git repository root (e.g. `/repos/pa.aid.runbook-executor`). The runbook file is inside that repository.
- UTC timestamp format is `YYYY-MM-DD HH:MM` produced via `new Date().toISOString().slice(0, 16).replace('T', ' ')`.
- The tool does NOT use the AST parser (`lib/parser.ts`). The patch operates via regex on raw file text — this is intentional to avoid a parse→re-serialize round-trip that would corrupt whitespace and formatting.
- `pa.aid.config.md` tools directory is the source of truth at `/home/zimmermann/.config-src/pa.aid.config.md/tools/`. The `~/.config/opencode/tools/` symlink points there. The tool file is committed to `pa.aid.config.md`.

---

## Implementation Steps

### Step 1: Identify the story's `🔒 Claimed:` sub-item by raw text scan

**Files:** `~/.config/opencode/tools/runbook_claim_story.ts` (new)
**Action:** Write a helper function `findClaimedSubItem(text: string, storyKey: string)` that scans raw runbook file text. The function:
1. Locates the story's top-level task item — the line matching `- [ ] **{storyKey}**` or `- [x] **{storyKey}**` (the story may be checked).
2. Scans subsequent lines with greater indentation until a line matching `/🔒 Claimed:/` is found or a new top-level item begins.
3. Returns an object `{ found: boolean; alreadyClaimed: boolean; lineIndex: number }` where `lineIndex` is the zero-based index into `lines[]` of the `🔒 Claimed:` line.

The function returns `{ found: false }` when no `🔒 Claimed:` line exists under the story, and `{ found: true, alreadyClaimed: true }` when the line starts with `- [x]`.

**Verification:** Unit test cases in Step 5 cover: found+unchecked, found+already-claimed, not-found.

---

### Step 2: Implement the in-place text patch

**Files:** `~/.config/opencode/tools/runbook_claim_story.ts`
**Action:** Write a helper function `patchClaimedLine(line: string, lane: string, timestamp: string): string` that replaces the checkbox `[ ]` with `[x]` and appends `{lane} / {timestamp}` after `🔒 Claimed:`. The function must preserve leading whitespace so indentation is unchanged.

The timestamp is produced inline in `execute` via `new Date().toISOString().slice(0, 16).replace('T', ' ')` (UTC, `YYYY-MM-DD HH:MM`).

Write a helper `readLines(path: string): string[]` and `writeLines(path: string, lines: string[]): void` that use `Bun.file(path).text()` and `Bun.write(path, content)` respectively.

**Verification:** Unit test for `patchClaimedLine` confirms leading whitespace is preserved and the output matches `  - [x] 🔒 Claimed: {lane} / {timestamp}` exactly.

---

### Step 3: Implement the `execute` function with guards and git operations

**Files:** `~/.config/opencode/tools/runbook_claim_story.ts`
**Action:** The `execute(args)` function must:

1. Read the runbook file with `readLines(runbook_path)`.
2. Call `findClaimedSubItem(text, story_key)`.
3. **Guard — not found:** if `found === false`, return `"No Claimed sub-item found for story {story_key}"` immediately with no file write.
4. **Guard — already claimed:** if `alreadyClaimed === true`, return `"Story {story_key} is already claimed"` immediately with no file write.
5. Produce `timestamp` and call `patchClaimedLine` on the target line. Write the updated lines back with `writeLines`.
6. Run `git add {runbook_path}` in `repo_path` via `Bun.$\`git -C ${repo_path} add ${runbook_path}\``. On non-zero exit, return the stderr as an error string.
7. Run `git commit -m "chore({lane}): claim {story_key}"` via `Bun.$\`git -C ${repo_path} commit -m ...\``. On non-zero exit, return the stderr.
8. Run `git push` via `Bun.$\`git -C ${repo_path} push\``. On non-zero exit: the file and local commit are already written — return `"Push failed for {story_key}. Local commit preserved. Re-run: git -C {repo_path} push"`.
9. On success, return `"Claimed {story_key} as {lane} at {timestamp}. Committed and pushed."`.

Shell commands are wrapped individually in try/catch so a push failure does not roll back the commit.

**Verification:** Correct return strings match the AC wording exactly. Unit tests in Step 5 cover all branches.

---

### Step 4: Wire the tool export and args schema

**Files:** `~/.config/opencode/tools/runbook_claim_story.ts`
**Action:** Export the tool using `export default tool({...})` with the following args schema:

- `runbook_path`: `tool.schema.string().describe("Absolute path to the runbook markdown file")`
- `story_key`: `tool.schema.string().describe("Story key to claim, e.g. ARC-1285")`
- `lane`: `tool.schema.string().describe("Lane name used in the claim timestamp, e.g. core-infrastructure")`
- `repo_path`: `tool.schema.string().describe("Absolute path to the git repository root containing the runbook file")`

Top of file imports: `import { tool } from '@opencode-ai/plugin';` — no other imports except the Bun built-ins (`Bun.file`, `Bun.write`, `Bun.$`) which are globally available in the runtime.

**Verification:** File exports exactly one default `tool({...})` object. Running `node --experimental-strip-types --check ~/.config/opencode/tools/runbook_claim_story.ts` (or equivalent syntax check) exits without errors.

---

### Step 5: Write unit tests

**Files:** `~/.config/opencode/tools/tests/runbook_claim_story.test.ts`
**Action:** Write tests using `node:test` and `node:assert/strict` (same runner as `runbook_find_next_story.test.ts`). Export `findClaimedSubItem` and `patchClaimedLine` from the tool file so they can be imported. Tests must cover:

1. **Happy path — unchecked sub-item found:** fixture text with `- [ ] 🔒 Claimed:` under the target story → `found: true, alreadyClaimed: false`, `lineIndex` points to the correct line.
2. **Already claimed:** fixture with `- [x] 🔒 Claimed: lane / 2026-01-01 12:00` → `found: true, alreadyClaimed: true`.
3. **No sub-item:** fixture with no `🔒 Claimed:` line under the target story → `found: false`.
4. **`patchClaimedLine` output:** input `  - [ ] 🔒 Claimed:` → output `  - [x] 🔒 Claimed: core-infrastructure / 2026-07-02 10:00` with original indentation preserved.
5. **Wrong story key:** fixture has `🔒 Claimed:` only under a different story key → `found: false` for the requested key.
6. **Story not present in runbook:** fixture contains no matching story item → `found: false`.
7. **`patchClaimedLine` with trailing placeholder:** input `  - [ ] 🔒 Claimed: _(fill in)_` → output `  - [x] 🔒 Claimed: {lane} / {timestamp}` (placeholder replaced, not appended).

Run command: `node --experimental-strip-types --test ~/.config/opencode/tools/tests/runbook_claim_story.test.ts`

**Verification:** All 7 tests pass with exit code 0.

---

### Step 6: Commit tool file to `pa.aid.config.md`

**Files:** `/home/zimmermann/.config-src/pa.aid.config.md/tools/runbook_claim_story.ts`
**Action:** The `~/.config/opencode/tools/` directory is a symlink to `/home/zimmermann/.config-src/pa.aid.config.md/tools/`. The file written in Steps 1–4 therefore already lives in the correct source location. Commit it:

```bash
git -C /home/zimmermann/.config-src/pa.aid.config.md add tools/runbook_claim_story.ts tools/tests/runbook_claim_story.test.ts
git -C /home/zimmermann/.config-src/pa.aid.config.md commit -m "feat(agent-tools): add runbook_claim_story OpenCode tool (ARC-1349)"
```

**Verification:** `git -C /home/zimmermann/.config-src/pa.aid.config.md log --oneline -1` shows the commit.

---

### Step 7: Confirm allowlist in `opencode.json` and restart OpenCode

**Files:** `~/.config/opencode/opencode.json`, `/repos/pa.aid.wsl-setup.sh/components/opencode/opencode-sync-config.sh`
**Action:** `opencode-sync-config.sh` already lists `runbook_claim_story: True` for `build` and `senior-coder` agents (confirmed via grep). Verify that the live `~/.config/opencode/opencode.json` reflects this — the tool should appear in the `tools` allowlist under both agents. If it is absent (because `opencode.json` was not regenerated since the script was updated), patch it directly using the same Python-dict structure already present in the file for `runbook_find_next_story`.

Restart OpenCode to load the new tool.

**Verification:** After restart, invoke `runbook_claim_story` via OpenCode tool call against a test fixture (or use the smoke test in Step 8). Confirm it does not return "tool not found".

---

### Step 8: End-to-end smoke test

**Files:** Live runbook at `/repos/pa.aid.runbook-executor/docs/plans/runbook-agent-tools.md`
**Action:** Identify the first unclaimed story in the agent-tools runbook (expected: ARC-1349 or the next unclaimed entry). Call `runbook_claim_story` with:
- `runbook_path`: absolute path to `runbook-agent-tools.md`
- `story_key`: the identified key
- `lane`: `agent-tools`
- `repo_path`: `/repos/pa.aid.runbook-executor`

Confirm the runbook file is updated in-place, `git log` shows the new commit, and `git push` succeeds.

**Verification:** `git -C /repos/pa.aid.runbook-executor log --oneline -1` shows `chore(agent-tools): claim {KEY}`.

---

## Testing & Validation

**Unit tests:**
Run `node --experimental-strip-types --test ~/.config/opencode/tools/tests/runbook_claim_story.test.ts`. All 7 scenarios must pass with exit code 0.

**Type check:**
Run `node --experimental-strip-types --check ~/.config/opencode/tools/runbook_claim_story.ts`. No errors.

**Allowlist check:**
Confirm `runbook_claim_story` appears in `~/.config/opencode/opencode.json` under `build` and `senior-coder` agent tool allowlists.

**End-to-end smoke test:**
Call the tool against the live agent-tools runbook via OpenCode session. Confirm the runbook file is patched, the commit appears in git log, and the push succeeds.

**Guard validation:**
Manually invoke the tool a second time on the same story after it is claimed. Confirm the return value is `"Story {KEY} is already claimed"` and no duplicate commit is made.

---

## Risks & Open Questions

1. **Regex fragility for story key matching:** The `findClaimedSubItem` function must locate the story's section boundary reliably. Runbook markdown uses `- [ ] **{KEY}**` as the top-level task item. If stories use inconsistent bold-wrap formatting, the scan may miss the section. Mitigation: test against the actual runbook files during Step 8.

2. **Multi-story runbooks with sub-items of varying indentation:** The sub-item scan advances until a line at equal or lesser indentation to the story item is found. If indentation is inconsistent (tabs vs spaces), the boundary detection may be off. Mitigation: normalize indentation check to strip-then-compare leading chars.

3. **Concurrent claims from two agents:** If two agents call `runbook_claim_story` for the same story simultaneously, the second call will either: (a) see the already-claimed guard if it reads after the first write, or (b) produce a duplicate commit. This is an inherent TOCTOU race. Mitigation: the `already-claimed` guard is checked before any write — acceptable for the current single-agent workflow.

4. **Push failure leaving repo in diverged state:** After a successful local commit, a push failure leaves the local branch ahead of remote. The tool returns instructions to re-run `git push`. The agent must not retry the full claim (which would double-commit). The push-only retry path is safe because the file and commit already exist.

5. **`patchClaimedLine` trailing content replacement:** The existing sub-item may have trailing text like `_(fill in)_`. The replacement must overwrite the entire line content after `🔒 Claimed:`, not append. Ensure the regex replaces from `🔒 Claimed:` to end-of-line, not just the checkbox prefix.
