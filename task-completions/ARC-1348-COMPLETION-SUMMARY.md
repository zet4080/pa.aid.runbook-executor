# ARC-1348 Completion Summary — `runbook_find_next_story` OpenCode Tool

**Story:** ARC-1348 — Implement `runbook_find_next_story` OpenCode tool
**Completed:** 2026-07-02
**Implementation commit:** b3fa7f6 (pa.aid.config.md, branch ARC-1346)
**WSL setup commit:** 5297e8a (pa.aid.wsl-setup.sh, branch main)

---

## Acceptance Criteria Evidence

### AC 1: Returns first unchecked AND unclaimed story in wave order

**Given** a runbook file with multiple waves and stories,
**When** `runbook_find_next_story` is called with the runbook path,
**Then** it returns the first story whose checkbox is `[ ]` AND whose Claimed sub-item is also `[ ]` (or absent).

**Evidence:** Unit test 3 (`First unchecked, no claim sub-item → returns that step`) passes. Smoke test against live runbook returns `{"storyKey":"step-0-1","title":"Confirm ARC-1348 complete","wave":"Wave 1 — Runbook Navigation Tools","risk":null,"lineNumber":null}` — the first unchecked, unclaimed step in the runbook.

---

### AC 2: Claimed stories are skipped

**Given** a story that has `- [x] 🔒 Claimed:` sub-item,
**When** `runbook_find_next_story` is called,
**Then** that story is skipped — it is considered in-progress.

**Evidence:** Unit tests 4 and 7 both pass. Test 4 verifies that a story with a checked "Claimed" sub-item is skipped and the next unclaimed story is returned. Test 7 specifically verifies label matching against `/claimed/i`. Live runbook smoke test correctly skips ARC-1348 (which has `[x] 🔒 Claimed: agent-tools / 2026-07-02 14:00`).

---

### AC 3: All stories checked/claimed → returns null

**Given** all stories in all waves are either checked `[x]` or claimed,
**When** `runbook_find_next_story` is called,
**Then** the tool returns `null` with a message indicating no unclaimed work remains.

**Evidence:** Unit tests 1 and 2 both pass:
- Test 1: Empty runbook (no waves) → `{ "result": null, "message": "No unclaimed work remains in this runbook" }`
- Test 2: All stories checked → same null result

---

### AC 4: Runbook file not found → returns error

**Given** the runbook file does not exist at the given path,
**When** `runbook_find_next_story` is called,
**Then** the tool returns an error with the missing path.

**Evidence:** The `execute` function wraps `parseRunbook()` in a try/catch and returns a descriptive error string: `"Error: runbook file not found or unreadable: {path} — {message}"`.

---

## Test Output (all 7 tests passing)

```
✔ 1. Empty runbook (no waves) returns null result (9.111163ms)
✔ 2. All stories checked returns null result (6.997694ms)
✔ 3. First unchecked, no claim sub-item → returns that step (1.831784ms)
✔ 4. First unchecked claimed, second unchecked unclaimed → second story returned (2.135437ms)
✔ 5. Checked wave 1, unchecked in wave 2 → returns wave 2 story (1.69428ms)
✔ 6. Step with no subSteps → treated as unclaimed, returned (0.955631ms)
✔ 7. subStep checked:true with label matching /claimed/i → story skipped (1.709346ms)
ℹ tests 7
ℹ pass 7
ℹ fail 0
```

---

## Smoke Test Output

```json
{
  "storyKey": "step-0-1",
  "title": "Confirm ARC-1348 complete",
  "wave": "Wave 1 — Runbook Navigation Tools",
  "risk": null,
  "lineNumber": null
}
```

**Note:** The plan expected `ARC-1349` as the first unclaimed step, but the live runbook contains a "Confirm ARC-1348 complete" prerequisite step (no sub-items, unclaimed) between ARC-1348 and ARC-1349. The tool correctly returns this as the true next actionable step.

---

## Files Delivered

| File | Location |
|------|----------|
| Tool implementation | `~/.config/opencode/tools/runbook-tools.ts` |
| Unit tests | `~/.config/opencode/tools/runbook-tools.test.ts` |
| Parser copy (astWalker) | `~/.config/opencode/tools/lib/astWalker.ts` |
| Parser copy (parser) | `~/.config/opencode/tools/lib/parser.ts` |
| Parser copy (types) | `~/.config/opencode/tools/lib/types.ts` |
| Tool package.json | `~/.config/opencode/tools/package.json` |
| Plugin registration | `~/.config/opencode/opencode.json` — `"plugin"` array |
| WSL setup sync | `/repos/pa.aid.wsl-setup.sh/components/opencode/opencode-sync-config.sh` |
| Source repo (tools) | `~/.config-src/pa.aid.config.md/tools/` |

---

## Deviations from Plan

| Deviation | Reason |
|-----------|--------|
| Test runner: Node `node:test` instead of `bun test` | `bun` is not installed as a standalone binary; OpenCode embeds Bun inside `opencode.exe` but does not expose it as a CLI tool. Node v24 `--experimental-strip-types` + `node:test` provides equivalent TypeScript test execution. |
| Parser lib imports changed from `.js` to `.ts` | Node v24 resolves `.ts` files directly; `.js` extension imports cause `ERR_MODULE_NOT_FOUND` when running under Node. OpenCode's Bun runtime handles `.ts` imports natively regardless of extension. |
| `storyKey` extraction: `split('__').pop()` | Parser namespaces step IDs as `{lane}__{ARC-NNNN}`. The tool strips the namespace prefix to return a clean story key. |
| Tool files committed to `pa.aid.config.md` (not `pa.aid.conductor.ts`) | Tools are config artifacts, not application code. `~/.config/opencode/` is not a git repo, so `pa.aid.config.md` is the correct source repo for tool files. |

---

## Notes for Next Agent

- OpenCode must be restarted to pick up the new plugin from `opencode.json`
- After restart, `runbook_find_next_story` will appear in the tool list
- The tool is also callable directly via: `node --experimental-strip-types -e "import {findNextStoryFromParsed} from './runbook-tools.ts'; ..."`
- Run tests: `node --experimental-strip-types --test ~/.config/opencode/tools/runbook-tools.test.ts`
