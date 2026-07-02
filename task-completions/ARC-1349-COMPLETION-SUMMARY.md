# ARC-1349 Implement `runbook_claim_story` OpenCode tool — Completion Summary

**Issue:** `issues/agent-tools/ARC-1349-tool-runbook-claim-story.md`
**Implementation Plan:** `implementation_plans/agent-tools/ARC-1349-implementation-plan.md`
**Completed:** 2026-07-02
**Duration:** ~1 session
**Commit:** `0444080482a7e17964c503f4766ad15caaa0d030` in `pa.aid.config.md`

---

## Acceptance Criteria Verification

| # | Criterion | Status | Evidence |
|---|-----------|--------|----------|
| AC1 | Given unchecked `🔒 Claimed:` sub-item, when called, then sub-item patched to `[x]` with UTC timestamp, file staged, committed, and pushed | Passed | `validateClaim` test 1 (`unchecked Claimed sub-item → status ok`); `patchClaimedLine` tests 6–8 verify in-place patch with correct indentation and scoping; end-to-end git flow wired in `execute()` |
| AC2 | Given already-claimed sub-item `[x]`, when called, then returns `"Story {KEY} is already claimed"` and makes no changes | Passed | Test 2 (`checked Claimed sub-item → status already_claimed`); guard in `execute()` returns message before any file write |
| AC3 | Given no `🔒 Claimed:` sub-item, when called, then returns `"No Claimed sub-item found for story {KEY}"` and makes no changes | Passed | Test 3 (`no Claimed sub-item → status no_sub_item`); guard in `execute()` returns message before any file write |
| AC4 | Given git push failure, then local file and commit preserved; tool returns retry instructions | Passed | Push wrapped in separate try/catch from commit; on push failure returns `"Push failed for {story_key}. Local commit preserved. Re-run: git -C {repo_path} push"` |
| AC5 | Tool registered in allowlist for `build` and `senior-coder` agents | Passed | `runbook_claim_story` confirmed in `~/.config/opencode/opencode.json` under `build` (line 410) and `senior-coder` (line 483); `opencode-sync-config.sh` at lines 390 and 405 |

---

## Implementation Summary

A single TypeScript file `tools/runbook_claim_story.ts` (276 lines) was added to `pa.aid.config.md`. The tool:

- Uses `parseRunbook()` from `./lib/parser.ts` (delivered by ARC-1348) for structural validation via the AST — verifying story presence and `🔒 Claimed:` sub-item state before touching the file.
- Applies a story-key-anchored raw text patch via `patchClaimedLine()` to rewrite the `🔒 Claimed:` line in-place, preserving leading indentation and fully replacing any trailing placeholder `_(fill in)_`.
- Runs `git add`, `git commit -m "chore({lane}): claim {KEY}"`, and `git push` via `Bun.$` shell, with push isolated in its own try/catch so a push failure does not unwind the local commit.
- Exports `validateClaim` and `patchClaimedLine` as named exports for direct unit test access.
- Registered in the OpenCode tool allowlists for `build` and `senior-coder` agents.

---

## Verification Steps

```bash
# Run unit tests (8/8)
node --experimental-strip-types --test ~/.config/opencode/tools/tests/runbook_claim_story.test.ts

# Type-check (clean — no output expected)
node --experimental-strip-types --check ~/.config/opencode/tools/runbook_claim_story.ts

# Confirm commit in pa.aid.config.md
git -C /home/zimmermann/.config-src/pa.aid.config.md log --oneline | grep ARC-1349
```

---

## Tests Added

- `~/.config/opencode/tools/tests/runbook_claim_story.test.ts` (223 lines) — 8 unit tests:
  1. `validateClaim — unchecked Claimed sub-item → status ok`
  2. `validateClaim — checked Claimed sub-item → status already_claimed`
  3. `validateClaim — no Claimed sub-item → status no_sub_item`
  4. `validateClaim — story key not in runbook → status not_found`
  5. `validateClaim — Claimed sub-item only under different story → status not_found for requested key`
  6. `patchClaimedLine — preserves leading whitespace`
  7. `patchClaimedLine — trailing placeholder _(fill in)_ is fully replaced`
  8. `patchClaimedLine — only patches target story, leaves other stories unchanged`

**Test result:** pass 8 / fail 0
**Type-check result:** clean (no output)
**Local code review result:** 0 blockers, 0 issues, 1 non-blocking suggestion (symlink vs copy — environment state, not a code issue)

---

## Rollback Notes

None. The tool is a new file; removing it reverts the feature. No database migrations or schema changes.

---

## Next Steps

- ARC-1350: `runbook_check_step` — next story in Wave 1; depends on ARC-1349 being complete.
- ARC-1351: `runbook_check_story` — also depends on ARC-1349.