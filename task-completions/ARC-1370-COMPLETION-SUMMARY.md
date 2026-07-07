```markdown
### Completion Summary for Story ARC-1370

| Field | Details |
| --- | --- |
| **Type** | docs |
| **Status** | Completed |
| **Story** | ARC-1370 — Conductor: auto-update issue status and archive story artifacts on completion |
| **Epic** | ARC-1290 — ARC: Parallel Lane Execution |
| **Lane** | parallel-lane-execution |
| **Date completed** | 2026-07-07 |
| **Feature branch** | ARC-1370 (branched from ARC-1369) |
| **Commit hash** | ee9a75b |

#### Goal
This story aimed to implement an automatic process for updating issue statuses and archiving story artifacts upon task completion.

#### Acceptance Criteria Evidence

- **AC1 (Status → Completed, Last Updated → today)**: Implemented in `patchIssueFrontmatter`, tested in test case 5
- **AC2 (issue file moved to done/issues/)**: Implemented, tested in test case 4
- **AC3 (plan moved to done/plans/)**: Implemented, tested in test case 4
- **AC4 (summary moved to done/completions/)**: Implemented, tested in test case 4
- **AC5 (missing summary → skip + warn)**: Implemented, tested in test case 1
- **AC6 (done/README.md updated, committed, pushed)**: Implemented, tested in test cases 7-9
- **AC7 (close-issue skill updated)**: Done — conductor automation note added

#### Files Changed

- `packages/server/src/git/planningArchiver.ts`: New file
- `packages/server/src/runner/laneRunner.ts`: Modified
- `packages/server/src/__tests__/planningArchiver.test.ts`: New file
- `packages/server/src/__tests__/laneRunner.test.ts`: Extended
- `/home/zimmermann/.config/opencode/skills/close-issue/SKILL.md`: Modified

#### Test Evidence

- **New tests pass**: 84/84 new tests pass (68 laneRunner + 16 planningArchiver)
- **Total tests pass**: 239/241 total tests pass; the 2 remaining failures are pre-existing in `checkpointGit.test.ts` unrelated to this story.

```