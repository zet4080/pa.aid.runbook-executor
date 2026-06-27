---
name: pr-triage
description: Use when triaging today's pull requests to decide which ones warrant a deeper code quality review — lightweight first-pass producing a prioritized shortlist
---

# PR Triage

## Purpose

Fast, lightweight assessment of today's pull requests, producing prioritized shortlist of PRs warranting deeper review. Decision driven by code diff as primary signal.

## When to Use

- Triaging today's pull requests to identify review priority
- Before invoking pr-review skill on individual PRs
- When high-volume PR assessment needed

---

## Overview

Do a fast, lightweight assessment of today's pull requests and produce a prioritized shortlist of PRs that deserve a deeper Bitbucket PR review by the `pr-reviewer` agent. Decision driven primarily by the changed code itself. You are NOT doing the deep review — you are deciding what is worth reviewing.

**Core principle:** Code diff is the primary signal. Jira and metadata are supporting context.

## Three Phases (do not skip Phase 2)

### Phase 1 — Fetch and Status-Filter

1. **Fetch Today's Pull Requests** — follow "Fetching Today's PRs Correctly" below. Do NOT just fetch one page.

2. **Read Jira status for every PR** (no diff inspection in this phase):
   - Extract Jira issue key from branch/commit
   - Call `atlassian_jira_get_issue(issue_key=KEY, fields="*all", comment_limit=0)`
   - Record per PR: `id`, `title`, `author`, `target_branch`, `jira_key`, `jira_status`, `jira_summary`, `jira_type`, `jira_priority`, `jira_team` (`customfield_10001.name`)
   - Split into:
      - **`to_inspect`** — Jira status is `Implemented`, `Review`, or `In Review`
     - **`not_checked`** — every other status

3. **Write Phase 1 working list to disk** before continuing:
   - Save to `~/reports/pr-triage-[YYYY-MM-DD]-worklist.md`
   - Section `## To Inspect (N PRs)` — each PR as `- [ ] #ID — Jira-KEY (status, team)`
   - Section `## Not Checked (N PRs)` — rest with status reasons
   - Every `[ ]` in `## To Inspect` must be ticked off in Phase 2
   - Leave a placeholder `**Changed files:** (filled in Phase 2)` line under each `## To Inspect` entry — Phase 2 fills it with the real diffstat. The report's code note is later validated against this list.

### Phase 2 — Per-PR Diff Inspection (mandatory loop)

**Process the `to_inspect` list in batches of at most 10 PRs.** After each batch, re-read the worklist file from disk before continuing, so the loop does not drift or fill notes from memory at high PR volume. Never write a code note for a PR whose diff you have not fetched in the current run.

4. **Iterate `to_inspect` PR by PR.** No shortcuts, no batching of the fetch itself, no guesses. For each PR:
   - **First call `bitbucket_getPullRequestDiffStat`** — this is the ground truth for which files changed. Record the **exact changed file paths** (and add/remove counts) for this PR.
   - **Immediately write the changed-file list into the worklist** under that PR's `**Changed files:**` line (comma-separated paths, or a short list). This persisted list is the anchor for the report note and the pre-save validation.
   - Then call `bitbucket_getPullRequestDiff` for the actual content.
   - Identify file types changed (logic, tests, config, migrations, generated, presentation)
   - Estimate code-change complexity: branches, loops, state transitions, data transformations, query changes, permission checks, error handling, concurrency, integration logic
   - Consider scope: file count, patch size, concentrated vs spread
   - Read commit messages for intent (not substitute for diff)
   - Note target branch, cherry-pick vs original work
   - Use Jira to confirm alignment; flag mismatches
   - Flag unusual patterns: vague commit messages, large commit count, cross-team keys, PR description understating a complex diff
   - Assign exactly one tier: **REVIEW**, **OPTIONAL**, or **SKIP**
   - Update worklist file: `- [ ]` → `- [x]` with assigned tier, and ensure the `**Changed files:**` line is populated from the diffstat (not a guess).

   **Code-complexity note must be diff-anchored.** When you later write the Code Complexity note for a PR, it **must reference at least one real changed file path taken verbatim from that PR's diffstat** (e.g. `BJCPrintSvc.cls`, `bjwpse00.w`). A note that names a file not present in the PR's diffstat is a hard failure — it means the note was guessed, not read. Do not name files from memory, from the Jira text, or from a similar PR.

5. **Verify Phase 2 completeness before Phase 3**:
   - Re-read worklist file
   - If any `## To Inspect` entry still has `- [ ]`, return to Phase 2
   - If any `## To Inspect` entry has an empty `**Changed files:**` line, its diff was not inspected — return to Phase 2 for that PR
   - Hard gate: cannot enter Phase 3 with unchecked items or empty changed-file lists

### Phase 3 — Categorize and Report

6. **Tier definitions** (absolute, no exceptions):
   - **REVIEW**: non-trivial logic changes, meaningful control-flow, risky data handling, business-impacting fixes, sensitive domain code, implementation complexity
   - **OPTIONAL**: real but moderate changes; review useful but not urgent
   - **SKIP**: diff **was inspected** and clearly low-risk — routine facelift, text-only edits, mechanical renames, generated-file churn, cherry-picks of already-reviewed commits, merge-only commits
   - **NOT CHECKED**: Jira status not `Implemented`, `Review`, or `In Review` — diff must not be inspected; no other factor overrides

7. **Generate Triage Report**:
   - For REVIEW/OPTIONAL/SKIP: short code-complexity note from diff
   - For NOT CHECKED: only Jira status reason — no diff data
   - Include direct PR and Jira links
   - Verify report has same number of PRs as worklist; tier counts must match Phase 2
   - **Mandatory pre-save checklist** (all items must pass):
     1. All four sections present in exact order
     2. Summary line contains all four counters: REVIEW, OPTIONAL, SKIP, NOT CHECKED
     3. REVIEW + OPTIONAL + SKIP + NOT CHECKED == Total PRs today
      4. Every PR in NOT CHECKED has Jira status that is NOT `Implemented`, `Review`, or `In Review`
      5. Every PR in SKIP/OPTIONAL/REVIEW has Jira status that IS `Implemented`, `Review`, or `In Review`
      6. No SKIP entry uses "Status In Progress" / "Status Closed" / "DRAFT" as reason
     7. For every REVIEW/OPTIONAL block: Code Complexity and Reason are different sentences
     8. Encoding sanity: no mojibake markers; no ASCII transliteration of umlauts/special chars
	 9. Author field contains the Bitbucket PR author (the person who opened the PR), not the author of source-code commits or a file header — flag if the listed author does not match the PR submitter
	 10. **Diff-anchor validation (hard gate against hallucination):** For every REVIEW/OPTIONAL/SKIP block, every file path named in the Code Complexity note must appear in that PR's `**Changed files:**` list in the worklist. Re-read the worklist and cross-check each named file. If a note names a file that is not in the PR's diffstat, the note was hallucinated — discard it and re-derive the note from the worklist's changed-file list (re-fetch the diffstat if needed). Do not save until every named file matches.
   - If any check fails, fix before saving

8. **Write report to disk**: `~/reports/pr-triage-[YYYY-MM-DD].md`. Confirm file path.

9. **Clean up**: Delete both temporary files:
   - PR list file from `bitbucket_getPullRequests` under `/root/.local/share/opencode/tool-output/`
   - Worklist file `~/reports/pr-triage-[YYYY-MM-DD]-worklist.md`
   - Only the final report is kept

## Reading the Jira Issue

Extract issue key:
- Branch names: `<version>.<ISSUE-KEY>` → e.g. `9.PA-67105` → `PA-67105`
- Commit messages start with key: `PA-67105: Facelift | ...`
- Cherry-picks: use key of original commit

Call: `atlassian_jira_get_issue(issue_key="PA-12345", fields="*all", comment_limit=0)`

Read: issue type, summary/description, priority, status, team (`customfield_10001.name`), labels/components.

**Hard rule — check status first**: if Jira status NOT `Implemented`, `Review`, or `In Review` → assign NOT CHECKED, stop. Do not inspect diff. No exceptions.

Skip Jira lookup for: pure merge commits, cherry-picks where original PR was clearly already reviewed.

## Triage Criteria

**Lean REVIEW when:**
- Diff introduces or changes control flow, branching, state transitions, calculations, queries, permission checks, integration logic, or error handling
- Change appears non-trivial even if PR title sounds small
- Touches sensitive domains: compliance, billing, pricing, import/export, permissions, external integrations
- Spans many files, crosses subsystem boundaries, dense changes in core file
- Tests changed substantially, or behavior changed without corresponding tests
- Unusual number of fixup commits
- Introduces new behavior
- Jira describes bug with meaningful business impact
- Jira and code do not match

**Lean SKIP when:**
- Limited to formatting, comments, copy text, translations, layout-only, cosmetic-only
- Tiny and mechanically obvious, no logic change
- Cherry-pick of already-reviewed commit
- Only merge commits
- Issue and metadata align with cosmetic or isolated low-risk change

## Output Format

Four mandatory sections, always in this exact order (empty sections still present with `*(none)*`):

```
# PR Triage Report – [Date]
# Repository: [repo]

## Summary
Total PRs today: [X]  |  REVIEW: [X]  |  OPTIONAL: [X]  |  SKIP: [X]  |  NOT CHECKED: [X]

---

## Shortlist — Recommended for Review

### PR #XXXXX – [Title] @[Author] ([team]) → [target branch]
**Tier**: REVIEW
**PR**: [#XXXXX](https://...)
**Jira**: [ISSUE-KEY](https://...) – [summary] ([type], status: [status], priority: [priority], team: [team])
**Code Complexity**: [1 sentence — WHAT changed: MUST name at least one real changed file path from the PR's diffstat (e.g. `BJCPrintSvc.cls`), plus file count, classes/methods touched, new branches, etc. Factual, diff-anchored.]
**Reason**: [1-2 sentences — WHY this tier: risk, business impact, regression potential. Must NOT repeat Code Complexity.]

---

## Optional PRs

### PR #XXXXX – ...
[same block structure]

---

## Skipped PRs
PRs inspected but assessed as low-risk.
[compact list: [PR #ID](url) | [ISSUE-KEY](url) | team | complexity note | reason]

---

## Not Checked — Jira Status Gate
PRs skipped without diff inspection because Jira status is neither `Implemented`, `Review`, nor `In Review`.
[compact list: [PR #ID](url) | [ISSUE-KEY](url) | team | Jira status | reason]
```

## Fetching Today's PRs Correctly

⚠️ The Bitbucket PR list API sorts by `updated_on`. Always follow this procedure.

1. Call `bitbucket_getPullRequests` with `all: true`, `pagelen: 50`, `state: OPEN`. Full result saved to file.

2. Run Python filter script on the output file:

```bash
python3 - <<'EOF' /root/.local/share/opencode/tool-output/tool_<REPLACE_WITH_ACTUAL_FILENAME>
import json, sys
from datetime import date

today = str(date.today())
data = json.load(open(sys.argv[1]))
prs = data if isinstance(data, list) else data.get("values", data.get("pullrequests", []))
today_prs = [pr for pr in prs if pr.get("updated_on", "").startswith(today)]

print(f"Total PRs in file : {len(prs)}")
print(f"PRs updated today ({today}): {len(today_prs)}")
print()
for pr in sorted(today_prs, key=lambda x: x["updated_on"]):
    draft = " [DRAFT]" if pr.get("draft") else ""
    dest  = pr["destination"]["branch"]["name"]
    print(f"  #{pr['id']:>6}  {pr['updated_on'][11:16]}  {pr['author']['display_name']:<30}  dest={dest:<20}  {pr['title'][:80]}{draft}")
EOF
```

3. Use filtered IDs as working list.

Manual paging is unreliable — `updated_on` sort means old PRs with new comments jump to the top.

## Encoding Rules (UTF-8 mandatory)

Two failure modes — both forbidden:
1. **Mojibake** — broken byte sequences (`Fa├ƒbender` instead of `Faßbender`)
2. **ASCII transliteration** — replacing `ä→ae`, `ö→oe`, `ü→ue`, `ß→ss`. This is a hard failure.

When shelling out to Python:
- Always pass `encoding="utf-8"` to `open()`, `json.load()`, `json.dump()`, file IO calls
- Set `PYTHONIOENCODING=utf-8` or use `python -X utf8`
- Never use `.encode("ascii", errors="ignore")` or transliteration libraries

**Verification before saving — both mandatory**:
1. **Mojibake scan**: search for `├`, `┬`, `┼`, `Ã`, `ƒ`, `├╝`, `├ñ`, `├ƒ`. Any found = corrupted, regenerate.
2. **Transliteration scan**: verify non-ASCII characters present where expected. Any German name appearing as ASCII-only = transliteration failure, regenerate.
