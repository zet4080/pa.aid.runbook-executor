---
name: pr-review
description: Use when performing a full Bitbucket PR review — fetches Jira context, delegates all Bitbucket/SonarQube queries to qa-specialist, validates against architecture docs, produces and saves structured review report.
---

# PR Review Skill

## Purpose

Full Bitbucket PR review: fetch Jira context, delegate Bitbucket/SonarQube queries to qa-specialist, validate against architecture rules, produce structured review report.

## When to Use

- Performing full code quality review on Bitbucket PR
- Before or as part of merge decision
- When quality gate and AC coverage need validation

---

## Workflow

### Step 1 — Fetch Jira ticket first

Retrieve the linked Jira issue **before looking at any code**.

- `atlassian_jira_get_issue` — full details: summary, description, acceptance criteria
- `atlassian_jira_get_issue_development_info` — confirm PR linkage

### Step 2 — Delegate to qa-specialist

Instruct the `qa-specialist` subagent to fetch all of the following:

- PR metadata (title, description, reviewers, target branch, status)
- **Bitbucket workspace** of the repository (e.g. `destination.repository.full_name` → the segment before `/`, or the workspace field) — needed for repository-rule detection (workspace `proalpha`)
- **Structured diff** — request the structured-diff format described in `qa-specialist.md` § Returning PR Diffs: per-file hunks with explicit `old_line` / `new_line` absolute line numbers for every line. Do **not** ask for the raw unified diff — every finding's `File:Line` reference must be based on the structured form.
- List of changed files with paths
- Pipeline/build status for the PR branch
- SonarQube PR entry matching the Bitbucket PR branch
- Quality gate status, coverage, and duplication metrics
- New bugs, vulnerabilities, and code smells for the PR
- Files with insufficient test coverage
- Full details of any critical or blocker SonarQube issues
- qa-specialist's own model id (to record in report header)

### Step 3 — Load review rule sets

> ⚠️ **Rule file location** — Rule files live in the **OpenCode configuration directory**, not in the project working directory. Resolve every `rules/review/<file>.md` reference below using the **absolute path**:
> - Linux / WSL: `~/.config/opencode/rules/review/<file>.md`
> - macOS: `~/.config/opencode/rules/review/<file>.md`
> - Windows: `%USERPROFILE%\.config\opencode\rules\review\<file>.md`
>
> Use the `read` tool with the **absolute path** — do **not** search via `glob`/`grep` in the project working directory. If a `read` with the absolute path fails, fall back to `$OPENCODE_CONFIG_DIR/rules/review/<file>.md` if that environment variable is set; otherwise record the missing rule file in the report header and continue.

Load these rule files:

1. **Always**: `~/.config/opencode/rules/review/general.md` — language-agnostic baseline rules
2. **ABL extensions** (`.cls`, `.i`, `.p`, `.w` and any additional extensions listed in the file's § Detection): `~/.config/opencode/rules/review/progress.md`
3. **Proalpha repository rules**: `~/.config/opencode/rules/review/proalpha.openedge.md` — load when **any** of these match (see that file's § Detection for the canonical logic):
   - **Bitbucket workspace `proalpha` + ABL file** — the workspace returned by qa-specialist is `proalpha` (case-insensitive) **and** the diff contains at least one Progress/ABL file (OpenEdge standard `.cls`/`.i`/`.p`/`.w` or any Proalpha-specific extension below). Workspace + ABL-file together; if the workspace matches but no ABL file changed, do not load this file. No Bitbucket project key is required — workspace + ABL covers all current and future Proalpha ABL repos.
   - **Repo slug `proalpha.openedge`** (match PR's repo slug case-insensitively) — loads regardless of project key.
   - **Any Proalpha-specific ABL extension in the diff**: `.cdf`, `.clst`, `.cst`, `.df`, `.fir`, `.fld`, `.if`, `.ldf`, `.lfr`, `.lib`, `.pds`, `.pt`, `.sl`, `.sst`, `.tdf`, `.wt` — load even if repo/project detection fails.
   - This file extends ABL detection; treat its § Detection list as additional ABL triggers.

If a detected language has no rule file, note this in the report header and continue without language-specific rules.

Record all loaded rule files in the report header (general + language + repo) so the user can verify the active rule set.

### Step 4 — Validate against architecture documentation

Cross-check against project architecture and API conventions where available (`docs/architecture/`).

### Step 5 — Produce and save review report

Save to `docs/reviews/PR-{number}-review.md`.

**Do NOT include in the report:**
- The raw or summarized PR diff
- A "Changed Files" / "Diff Statistics" listing
- A "Diff Analysis" / "Code Walkthrough" section re-narrating the change

The PR diff is available in Bitbucket itself. Findings reference `File:Line` directly; the diff body must not be duplicated in the markdown report.

Include separate **General Rule Findings**, **\<Language> Rule Findings**, and **\<Repository> Rule Findings** tables when violations exist. The repository-specific table is only present when a repo-level rule file was loaded.

#### Report header table

Every report must open with this identity table:

| Item | Value |
|---|---|
| Review agent | `pr-reviewer` |
| Review agent model | `<your own model id, e.g. litellm/claude-sonnet-latest>` |
| Subagent | `qa-specialist` |
| Subagent model | `<qa-specialist's model id>` |
| Rule files loaded | `<list of loaded rule files>` |
| Pipeline status | `<green / red / unknown>` |
| SonarQube quality gate | `<passed / failed / not found>` |

Rules for identity rows:
- Use the **review agent's own model id** for "Review agent model" — from this agent's frontmatter at runtime, not a hard-coded string.
- Use the **qa-specialist's actual model id** for "Subagent model". Ask qa-specialist to report its model id alongside the diff/data delivery (or inspect the qa-specialist agent definition if direct reporting is not available).
- If qa-specialist was **not** invoked (rare; only when no Bitbucket/SonarQube call was needed), omit the two Subagent rows and add: "Subagent: not invoked".

#### Line-number accuracy

Every `File:Line` reference in a finding **must** be an exact, verifiable line number taken directly from the structured diff:

- Use `new_line` for findings on an added/modified line; `old_line` for findings on a removed line.
- Do **not** prefix line numbers with `~`, `approx`, `~around`, or any uncertainty marker — the structured diff provides exact numbers.
- Do **not** estimate, recompute, or extrapolate from `@@` hunk headers — that is qa-specialist's job.
- If a finding refers to a line **not** present in the structured diff (context line outside diff window, or a file not included):
  1. Request surrounding context from qa-specialist (wider hunk window or relevant file slice) to confirm the exact line number.
  2. If still uncertain, **drop the finding** rather than guessing — confidence below the threshold in `general.md` § Confidence thresholds means no finding.
- Single finding spanning a range: write `File:Start-End` (e.g. `BMCCopyAddDataSvo.cls:2729-2768`) using exact endpoints from the structured diff.
- Finding affecting a whole file (rename, missing test stub): use `File:1` and state "whole file" in the description.

#### Rule reference links

Every finding sourced from a loaded rule set must render its rule name as a **clickable Markdown link** when the rule file provides a `**Reference:**` URL.

- **Source of truth** — Each rule section in `proalpha.openedge.md` (and any rule file following the same convention) starts with `**Reference:** [Rule Name](URL)` directly under the section heading. Use that URL as the link target.
- **Format in findings table** — In the `Rule` column, render as `[Rule Name](URL)`. Do not include the AP number prefix in the link text.
- **Rules without a Reference** — Some rule sections (Code Style sub-sections, `.pdi` deletion review, `Code checked by` override, repository-internal-only rules) have no external Wiki link. For those, write the rule name as plain text.
- **`general.md` / `progress.md` findings** — These files do not carry `**Reference:**` URLs. Use plain-text rule names; do not invent URLs.

### Step 6 — Post findings to PR (only on explicit request)

Default behaviour: **local report only**. If the invoking user/agent explicitly asks to post findings (e.g. "post comments", "comment on PR", "publish findings"), delegate to `qa-specialist`:

- Findings tied to a specific file and line → **inline comments** via `bitbucket_addPullRequestComment` with the `inline` parameter (immediate posting, not pending/batch — see qa-specialist workaround for the Bitbucket batch-publish bug).
- Verdict, summary, and findings without a precise file:line anchor → **general PR comment**.
- Each inline comment must contain: rule category, severity, description, suggested fix.
- **Include the rule reference link** in every comment whose rule has a `**Reference:**` URL. Render as Markdown — Bitbucket renders Markdown in comments. Example: `Reference: [Buffer-Copy Is A Method](https://proalpha.atlassian.net/wiki/pages/viewpage.action?pageId=193437876)`. For rules without a Reference URL, omit the link.
- **Mark every posted comment as AI-generated.** Prepend `🤖 AI-generated review comment` (or `[AI-generated review comment]` when emojis are unwanted) to **both** inline and general PR comments. This is in addition to the rule category / severity / description / suggested fix content.
- If not explicitly requested, do **not** post anything to the PR.

## Acceptance Criteria Validation

For each acceptance criterion from the linked Jira ticket:

| # | Acceptance Criterion | Status | Notes |
|---|----------------------|--------|-------|
| 1 | Description | ✅ Met / ⚠️ Partially Met / ❌ Not Met | Explanation |

If no Jira ticket or acceptance criteria exist, note this explicitly.

## Verdict

> ⚠️ **Gating rule**: A PR **cannot** receive APPROVE if the build pipeline has failed or the SonarQube quality gate is RED.

- **🟢 APPROVE** — Build green, SonarQube passes, implementation meets acceptance criteria, ready for merge
- **🟡 REQUEST_CHANGES** — Code issues must be addressed before merge
- **🔴 BLOCKED** — Build pipeline failed or SonarQube quality gate RED; must not be merged until resolved
- **🔴 NEEDS_DISCUSSION** — Significant architectural or design concerns require team discussion

## Rule-set severity mapping

| Rule file severity | Report severity |
|---|---|
| HIGH | WARNING (escalate to CRITICAL for security / data loss / error-handling / schema / API violations) |
| MEDIUM | WARNING |
| LOW | SUGGESTION |

Confidence threshold from `general.md` § Confidence thresholds still applies — drop findings below the threshold rather than downgrade them.

## Self-check before responding

- [ ] Jira ticket fetched and acceptance criteria read **before** evaluating code?
- [ ] All Bitbucket and SonarQube calls delegated to qa-specialist subagent?
- [ ] Structured diff (not raw diff) requested from qa-specialist, and every `File:Line` reference based on its `new_line` / `old_line` fields?
- [ ] Every finding's `File:Line` reference uses an exact line number — no `~`, no `approx`, no estimates?
- [ ] Every finding sourced from a rule section with a `**Reference:**` URL renders the rule name as a Markdown link `[Rule Name](URL)` in the rule column?
- [ ] `general.md`, any matching language-specific rule file, and any matching repository-specific rule file loaded from absolute paths?
- [ ] Loaded rule files recorded in report header so user can verify active rule set?
- [ ] Pipeline status and SonarQube quality gate checked before issuing verdict?
- [ ] Acceptance criteria validated one by one?
- [ ] Rule-set findings in their own tables (General + language-specific + repository-specific when applicable)?
- [ ] Report saved to `docs/reviews/PR-{number}-review.md`?
- [ ] Report header includes identity rows (Review agent, Review agent model, Subagent, Subagent model)?
