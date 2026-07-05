# ARC-1351 Completion Summary

## Story/Issue/Date
Story: ARC-1351 — Implement `runbook_check_story` OpenCode tool
Issue: `issues/agent-tools/ARC-1351-tool-runbook-check-story.md`
Lane: agent-tools
Date: 2026-07-05

## Summary
The `runbook_check_story` tool was implemented to check off a story in a runbook and commit the change. The tool was added to the allowlists for `build` and `senior-coder` agents.

## Acceptance Criteria

1. **Given** a runbook with a story whose top-level checkbox is `[ ]`, **When** `runbook_check_story({ runbook_path, story_key, lane, repo_path })` is called, **Then** the story checkbox is flipped to `[x]`, the file is committed and pushed with message `chore({lane}): close {KEY}`.
    - ✅ Implemented in `execute()` via `patchStoryLine()` + git add/commit/push

2. **Given** the story checkbox is already `[x]`, **When** `runbook_check_story` is called, **Then** the tool returns a no-op success: "Story {KEY} already checked off".
    - ✅ Implemented in `validateStory()` `already_checked` branch; test #2 passes

3. **Given** the story key is not found in the runbook, **When** `runbook_check_story` is called, **Then** the tool returns an error: "Story {KEY} not found in runbook".
    - ✅ Implemented in `validateStory()` `not_found` branch; test #3 passes

## Implementation Notes
- `validateStory()` uses `step.line` (1-based) and `step.checked` from `RunbookStep` AST fields — no raw regex
- `patchStoryLine()` directly indexes into the line array by line number — no regex re-scan
- Namespace prefix stripping: `step.id.split('__').pop()` matches "agent-tools__ARC-1351" → "ARC-1351"
- Push failure returns informative string with re-run command, preserving local commit
- Commit message: `chore({lane}): close {story_key}`

## Test Evidence
```
✔ 1. validateStory — unchecked story → status ok, line set and positive (10.594273ms)
✔ 2. validateStory — already-checked story → status already_checked, line set (4.359271ms)
✔ 3. validateStory — key not present → status not_found (4.759882ms)
✔ 4. validateStory — namespace-prefixed story key stripped correctly (1.622875ms)
✔ 5. patchStoryLine — flips [ ] to [x] on correct line (0.134352ms)
✔ 6. patchStoryLine — preserves surrounding lines unchanged (0.126289ms)
✔ 7. patchStoryLine — out-of-bounds line returns null (0.078664ms)
✔ 8. patchStoryLine — two stories in fixture: only target line patched (0.092747ms)
ℹ tests 8, pass 8, fail 0
```

## Files Changed
- `~/.config/opencode/tools/runbook_check_story.ts`
- `~/.config/opencode/tools/tests/runbook_check_story.test.ts`

## Commit Reference
Commit hash: 19370d5
Branch: `ARC-1351`