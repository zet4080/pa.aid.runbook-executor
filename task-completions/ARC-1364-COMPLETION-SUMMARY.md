### Frontmatter
---
issue: ARC-1364
lane: agent-tools
completion: true
---

### Implementation Summary

A new OpenCode custom tool `generate_issue_file.ts` was created to generate fully conformant issue markdown files from structured input. This tool is part of the `agent-tools` lane and was implemented to streamline the creation of issue files.

### Acceptance Criteria Verification

| Acceptance Criteria | Evidence/Test | 
|---|---|---
| Valid input → file written, returns `{ success: true, path }` | Test 7 | 
| Missing required fields → returns `{ success: false, errors }` without writing | Tests 8, 9 | 
| `{ given, when, then }` objects rendered as **Given**/**When**/**Then** blocks | Tests 2, 4 | 
| Missing output dir → created recursively, file written | Test 12 | 
| Existing file + no overwrite → returns error with 'File already exists:...' message | Test 10 | 

### File List

- `/home/zimmermann/.config/opencode/tools/generate_issue_file.ts`
- `/home/zimmermann/.config/opencode/tools/tests/generate_issue_file.test.ts`

### Test Results Summary

All 15 tests pass.
