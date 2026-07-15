# PR Review Working Instructions

When given a GitHub PR link, follow these instructions exactly to produce a complete PR review file. Read this file before starting any work.

---

## 0. Preflight

Before creating any files, run these checks:

1. **Verify PR state and readiness.** Fetch the PR (see Section 2) and check the JSON metadata:
   ```bash
   gh pr view {number} --repo tazama-lf/{repo-name} --json state,isDraft,baseRefName,headRefOid,mergeStateStatus,mergeable,statusCheckRollup
   ```
   - If `state` is `MERGED` or `CLOSED`, stop and confirm with the user before producing docs — the PR may no longer need review.
   - If `isDraft` is `true`, confirm the user wants a review of a draft before proceeding.
   - Note `statusCheckRollup` — record failing CI checks in the Overview.
2. **Confirm base branch is correct.** For this repo's workflow, PRs typically target `dev`. If `baseRefName` is not `dev`, flag it as a potential blocking issue in the review (do not assume the author intended it).
3. **Scope guardrail.** If the PR touches **more than 140 files**, stop and confirm scope with the user before writing. Offer: (a) proceed as one large review, (b) split "What Changed (Detailed)" into a grouped/summarised form, (c) narrow scope to specific files. Do not silently produce an unreviewable review file.
4. **Out-of-scope PR check.** If the PR is purely a dependency bump (Renovate/Dependabot), a lockfile-only change, or a generated-file update with no human-authored logic, stop and ask the user whether they want the full-format review or a short-form note. Do not produce the full deliverable for a one-line version bump.

---

## 1. Setup

Create a file at:
```
pull-requests/{REPO-ABBREVIATION}-PR-{number}.md
```

Where `REPO-ABBREVIATION` is the short name used in the project (e.g. `CMS`, `DEMS`, `TCS-LIB`, `TRS`) and `number` is the PR number (e.g. `221`). Match the abbreviation style already used in this directory.

---

## 2. Fetch the PR

Run:
```bash
gh pr view {number} --repo tazama-lf/{repo-name} --comments
```

Also fetch the diff:
```bash
gh pr diff {number} --repo tazama-lf/{repo-name}
```

And fetch review activity (CodeRabbit and human reviewers):
```bash
gh api repos/tazama-lf/{repo-name}/pulls/{number}/reviews
gh api repos/tazama-lf/{repo-name}/pulls/{number}/comments
```

Read all comments, review threads, and inline code comments. The PR description, commit messages, and comment threads are all authoritative context — treat them that way.

---

## 3. Read the Code

Before writing anything, read the actual diff. Do not describe what code *probably* does — trace the exact lines changed. For every meaningful change:

- Read the file it is in (use the clones under `repos/`)
- Confirm the before/after state
- Understand what callers or dependents are affected

Before reading any code, ensure the local repo is synced with the remote `dev` branch, then check out the PR's branch:

- If `repos/{repo-name}/` does not exist, clone it from the `dev` branch:
  ```bash
  git clone -b dev git@github.com:tazama-lf/{repo-name}.git repos/{repo-name}
  ```
- If `repos/{repo-name}/` already exists:
  - If the current branch is not `dev`, switch to it: `git -C repos/{repo-name} checkout dev`
  - Always fetch and fast-forward to the latest remote `dev`:
    ```bash
    git -C repos/{repo-name} fetch origin dev
    git -C repos/{repo-name} pull --ff-only origin dev
    ```
- Then check out the PR's head branch so the working tree reflects the PR under review:
  ```bash
  gh pr checkout {number} --repo tazama-lf/{repo-name}
  ```
  Run this from inside `repos/{repo-name}/`.
- **Verify the checkout matches the PR's latest commit.** `gh pr checkout` can silently land on a stale local branch if one exists from a prior review or if the PR was force-pushed. Compare:
  ```bash
  git -C repos/{repo-name} rev-parse HEAD
  gh pr view {number} --repo tazama-lf/{repo-name} --json headRefOid -q .headRefOid
  ```
  These SHAs must match. If they don't, delete the stale local branch and re-run `gh pr checkout`, or force-reset to the remote head. Do not proceed until they match.

Only after the local repo is synced with `origin/dev` and the PR branch is checked out (with SHAs verified) may you proceed with the review. If any step fails (dirty working tree, diverged history, auth failure, PR branch conflicts), stop and report it — do not read stale code.

**Source of truth once checked out.** After `gh pr checkout`, the working tree is authoritative for reading code. `gh pr diff` is a convenience for seeing the delta, but if it disagrees with the working tree (which can happen if the PR was updated between fetches), re-run the fetches and re-checkout — do not mix the two sources in a single review.

If a security-relevant function is touched (auth guards, input handling, URL construction, HTML rendering), read enough surrounding code to understand the full data flow.

---

## 4. Determine Review Rounds

A PR may have one or more review rounds if the author pushed additional commits after initial review. For each round, note:
- The commit hash reviewed
- Who reviewed (human and/or CodeRabbit)
- What was requested
- What was resolved or not resolved in the next push

Use this to structure the file as one section per round (Initial Review, Follow-up Review, Final Review, etc.).

**Handling force-pushed / rebased PRs.** If the author force-pushed and prior-round commit hashes no longer exist in the repo (`git cat-file -e {hash}` fails), do not silently produce broken commit citations. Instead:

1. Fetch prior review activity from GitHub's API — reviews and inline comments remain accessible by review ID even after force-push:
   ```bash
   gh api repos/tazama-lf/{repo-name}/pulls/{number}/reviews
   gh api repos/tazama-lf/{repo-name}/pulls/{number}/comments
   ```
2. In the follow-up section, note the force-push explicitly: `**Note:** Prior round reviewed commit `{old-hash}` which no longer exists after force-push on {date}. Prior findings reconstructed from GitHub review API.`
3. Reconcile each prior finding against the new HEAD by content (file + function + concern) rather than by commit range. Cite the current HEAD SHA for the new evidence.
4. If the rewrite is extensive enough that prior findings cannot be mapped, treat this round as a fresh review and note the reason.

---

## 5. File Format

The file is a single Markdown file. All sections must appear in the order below.

---

### Header

```markdown
# PR Review: {REPO-ABBREVIATION} #{number} — {PR title}
```

If the PR has only one review round, place these metadata fields immediately after the title (before the Table of Contents):

```markdown
**Repo:** tazama-lf/{repo-name}
**Branch:** `{source-branch}` → `{target-branch}`
**Author:** {GitHub username} ({display name if known})
**Date Reviewed:** {today's date}
**Label:** {label(s)}
**Size:** +{additions} / -{deletions} lines across {N} files
**Commits:** {N} ({hash1}, {hash2}, ...)
**State:** OPEN / MERGED / CLOSED
**Existing approvals:** {reviewer} — {status} ({date}) [if any]
```

If the PR has multiple review rounds, omit the per-round metadata from the header and instead include it at the top of each review section.

---

### Table of Contents

Always include a Table of Contents immediately after the header block. Every `##` section gets a ToC entry. Every `###` subsection gets a nested ToC entry. Use anchor links.

```markdown
## Table of Contents

- [Overview](#overview)
- [What Changed (Detailed)](#what-changed-detailed)
  - [1. {filename} — {one-line description}](#...)
  - ...
- [Code Quality Analysis](#code-quality-analysis)
  - [Strengths](#strengths)
  - [Issues and Observations](#issues-and-observations)
- [Security Assessment](#security-assessment)
- [Test Coverage](#test-coverage)
- [CodeRabbit Activity](#coderabbit-activity) ← only if CodeRabbit reviewed this PR
- [Summary and Verdict](#summary-and-verdict)
- [GitHub Review Comment](#github-review-comment)
```

For multi-round PRs, the Table of Contents covers all rounds:
```markdown
- [Initial Review (YYYY-MM-DD)](#initial-review-yyyy-mm-dd)
  - [Overview](#overview)
  - [What Changed (Detailed)](#what-changed-detailed)
  - ...
  - [Summary and Verdict](#summary-and-verdict)
- [Follow-up Review (YYYY-MM-DD)](#follow-up-review-yyyy-mm-dd)
  - [Changes Requested — Resolution Status](#changes-requested--resolution-status)
  - [New Issues Found in Updated Commits](#new-issues-found-in-updated-commits)
  - [Updated Verdict](#updated-verdict)
- [Final Review (YYYY-MM-DD)](#final-review-yyyy-mm-dd)
  - [Resolution Status — All Outstanding Items](#resolution-status--all-outstanding-items)
  - [Final Verdict](#final-verdict)
```

---

### Back-to-Top Links

Every `##` section must end with a back-to-top link immediately before the next `---`:

```markdown
[↑ Back to top](#pr-review-{repo-abbreviation-lowercase}-{number}--{title-slug})
```

Use the exact H1 anchor. For example:
```markdown
[↑ Back to top](#pr-review-cms-221--fix-transaction-history-graph-and-visualization)
```

Place the link after the section content and before the `---` separator.

---

### Overview

One to three paragraphs. State plainly: what the PR does, what files it touches, what problem it solves. If the PR description contains useful context, summarise it here. If the PR bundles unrelated changes (e.g., a feature + test expansion + lockfile churn), identify each distinct concern.

**Always state the base branch explicitly** (e.g. "Targets `dev` — correct for this repo's flow" or "Targets `main` — **flag as potential blocker**, this repo's PRs normally target `dev`"). Also note any failing CI checks recorded in Preflight.

Include a file-change summary table:

```markdown
| File | Nature of Change |
|------|-----------------|
| `path/to/file.ts` | Brief description |
```

---

### What Changed (Detailed)

One numbered subsection per meaningfully changed file or logical change group. For each:

- Show the diff in a code block (before/after or diff format)
- Explain what the change does and why
- Note any edge cases, risks, or interactions with other code

For diffs, prefer the before/after format:
```python
# Before
old_code_here()

# After
new_code_here()
```

Or use unified diff format for compact changes:
```diff
- old line
+ new line
```

Do not summarise — show the actual code. Readers must be able to audit the change from this section alone.

---

### Code Quality Analysis (use this section if the PR warrants it)

Two subsections:

**Strengths** — what the PR does well: pattern consistency, clean imports, minimal scope, good test structure, appropriate use of existing abstractions, etc.

**Issues and Observations** — every problem found, categorised by severity:

```
#### Issue {N} — {short title}

**Severity: {Major / Minor / Informational} ({category})**

{Description of the problem. Show the problematic code. Explain why it is wrong or risky. If there is a fix, show it.}
```

Categories: Bug, Security, Test Coverage, Code Quality, Maintainability, Breaking Change Risk, Data Integrity, Performance.

Use `Major` for anything that should block merge. Use `Minor` for things that are recommended but not blocking. Use `Informational` for observations that are pre-existing or out of scope.

---

### Authorization Architecture Analysis (include only if the PR changes auth logic)

Explain how the guard or auth mechanism works, what claims are required, what the effect of the change is, and whether it introduces OR/AND logic nuance. Answer explicitly:
- Who can access the endpoint after the change
- Who is newly blocked
- Whether this is intentional

---

### Security Assessment

Always include this section. Use a table:

```markdown
| Concern | Assessment |
|---------|-----------|
| {concern} | {assessment} |
```

If a security concern exists (XSS, injection, auth bypass, request timeout, claim mismatch), state it clearly, note whether it is new or pre-existing, and state whether this PR fixes or worsens it.

If no new vulnerabilities are introduced, say: "No new security vulnerabilities introduced by this PR." Do not omit the section.

---

### Test Coverage

Always include this section. State:
- What is tested
- What is not tested (especially for the core change)
- Whether the PR checklist is completed (and which boxes are checked or unchecked)
- Whether the coverage screenshot or CI evidence is present

If the core functional change has no test, call it out explicitly — do not soften it.

---

### CodeRabbit Activity (include only if CodeRabbit reviewed this PR)

**Reconcile CodeRabbit findings against your own.** Before writing this section, cross-check every CodeRabbit finding against your "Issues and Observations" list. For each CodeRabbit finding:
- If you also flagged it independently, note it as **corroborated** in both sections.
- If CodeRabbit caught something you missed, add it to your Issues list at the appropriate severity — do not leave it only in the CodeRabbit section.
- If you flagged something CodeRabbit missed, that's fine — do not remove it.
- If you disagree with a CodeRabbit finding, state your reasoning explicitly. Do not silently omit it.

One subsection per CodeRabbit pass. For each pass:

```markdown
### Pass {N} — {brief description}

**Commit reviewed:** `{hash}`
**Findings:** {count} actionable comments

| Finding | Severity | Status |
|---------|----------|--------|
| {description} | {severity} | ✅ Resolved / ⚠️ Partial / ❌ Not resolved |
```

---

### Resolution Status Table (for follow-up and final review sections)

When reviewing a PR after a CHANGES_REQUESTED round, open each prior item with a `### Item {N}` subsection:

```markdown
### Item {N} — {short title}

**Status: RESOLVED / PARTIALLY RESOLVED / NOT RESOLVED**

{Evidence: what changed and why. Show the code if helpful.}
```

End with a summary table:

```markdown
| # | Item | Status |
|---|------|--------|
| 1 | {description} | ✅ Resolved |
| 2 | {description} | ⚠️ Partial — {what remains} |
| 3 | {description} | ❌ Not resolved |
```

---

### Summary and Verdict

Always the second-to-last section (before GitHub Review Comment). Structure:

```markdown
**Verdict: {Approved / Approve with minor cleanup requested / Changes Requested / Request Changes}**

{One to two paragraphs summarising the overall quality of the PR and the reasoning behind the verdict.}

### Blocking

1. **{Issue title}** — {one-sentence explanation}

### Non-blocking but recommended

2. **{Issue title}** — {one-sentence explanation}
```

If there are no blocking items, say so explicitly. Do not list non-blocking items as blocking.

Verdict wording conventions:
- `Approved` — all items resolved, ready to merge
- `Approve with minor cleanup requested` — functionally correct, minor cosmetic/dead-code items remain
- `Changes Requested` — one or more blocking items must be addressed before merge

---

### GitHub Review Comment

Always the last section. Contains ready-to-paste Markdown for posting directly as a GitHub review comment. Format:

````markdown
## GitHub Review Comment

```markdown
**{Verdict}**

{One to two sentences summarising the PR quality and the verdict rationale.}

---

### Blocking

**{N}. {Issue title}**

{Clear description of the problem and what the author needs to do to fix it. Include code examples if helpful.}

---

### Non-blocking (please address in this PR if possible)

**{N}. {Issue title}**

{Description. Code example if useful.}
```
````

The comment must be self-contained — it should make sense to the PR author without needing to read the full review file. Include specific file paths, line references, and code examples in the comment itself.

**Backtick fence rule:** The outer fence wrapping the entire GitHub Review Comment block must use **at least one more backtick** than any fence inside it. If the comment body contains a ` ```diff ` or ` ```typescript ` block (3 backticks), the outer fence must use 4 backticks (` ```` `). If the body contains a 4-backtick fence, use 5 for the outer, and so on. Violating this causes the inner fence to close the outer one, breaking the raw-markdown display.

---

## 6. Multi-Round PR Structure

If the PR has gone through one or more CHANGES_REQUESTED cycles, structure the file as multiple dated sections separated by triple `---` dividers:

```
---
---
---

## Follow-up Review (YYYY-MM-DD)

**Reviewed commit:** `{hash}` — *"{commit message}"* ({date})
**Reviewed against:** CHANGES_REQUESTED on commit `{hash}` by `{reviewer}`
**Developer response:** {what the developer said in their comment, quoted}

...
```

### 6.1 Read ALL comments before writing the follow-up

**Before drafting any follow-up review, re-fetch and re-read every comment on the PR — do not rely on the state captured in the previous round's review file.** Between rounds, authors and reviewers routinely post issue comments, inline review comments, and review-thread replies that resolve items *by explanation* rather than by code change. Missing these is the #1 cause of a follow-up incorrectly marking items NOT RESOLVED.

Concretely, run all four fetches again at the start of every follow-up round — not just the diff:

```bash
gh pr view {number} --repo tazama-lf/{repo-name} --comments
gh pr diff {number} --repo tazama-lf/{repo-name}
gh api repos/tazama-lf/{repo-name}/pulls/{number}/reviews
gh api repos/tazama-lf/{repo-name}/pulls/{number}/comments
gh api repos/tazama-lf/{repo-name}/issues/{number}/comments  # issue-level PR comments (author replies to reviews often land here)
```

For every prior blocking or non-blocking item, check for author responses in *all* of the above before deciding a status. An item can be RESOLVED in four distinct ways:

1. **Code fix** — a new commit changes the code. Cite the commit hash.
2. **Explanation** — the author explains why the concern doesn't apply, or commits to a deployment/ops note. Quote the response and link to the comment (`https://github.com/.../pull/{number}#issuecomment-{id}`).
3. **Declined as intentional** — the author confirms the flagged behaviour is deliberate. Mark as `➖ Declined by author` in the summary table and quote the justification.
4. **Deferred to a follow-up PR / issue** — link the follow-up.

Only mark an item `❌ Not resolved` after confirming there is no author response of any of the four kinds. If in doubt, quote the closest response verbatim and let the reader judge — don't silently assume silence.

### 6.2 Follow-up section contents

Each follow-up section contains:
1. Resolution Status (one `### Item N` subsection per prior item) — every item must cite either the commit that fixed it OR the comment (with URL) that answered it OR an explicit `no author response` note. End with a summary table.
2. New Issues Found in Updated Commits (if any new problems were introduced) — treat these as first-class findings with severity + fix, same standards as initial-round issues.
3. Updated Verdict (new verdict + blocking / non-blocking split with one-paragraph reasoning).
4. **GitHub Review Comment (Follow-up)** — a per-round, ready-to-paste GitHub comment. Each follow-up round produces its OWN self-contained comment, headed e.g. `**Changes Requested (follow-up, HEAD `{sha}`)**`, listing only the still-blocking items and any new-issue blockers from this round. Do not rely on the initial round's comment being read alongside it — the follow-up comment must stand on its own so the author sees exactly what is still owed.

The Final Review section (only when the PR is closed out) replaces the Summary and Verdict section from the initial review and includes the definitive verdict with the complete resolution table.

### 6.3 Append mode — do not rewrite prior rounds

**When adding a follow-up round to an existing review file, append; never rewrite.** The prior round's Overview, What-Changed, Issues-and-Observations, Summary-and-Verdict, and GitHub-Review-Comment sections stay intact as the historical record — they capture what was known and posted at that time, and are the anchor for the author's follow-up response. Concretely:

1. **Preserve everything above.** Do not edit prior-round headings, tables, verdict wording, or the initial GitHub Review Comment. If a prior-round claim turned out to be wrong, note the correction in the follow-up section rather than rewriting history.
2. **Extend the Table of Contents.** Add nested entries for the new round (e.g. `- [Follow-up Review (YYYY-MM-DD)](#follow-up-review-yyyy-mm-dd)` with sub-entries for Resolution Status, New Issues, Updated Verdict, GitHub Review Comment). Do not remove the original single-round ToC entries.
3. **Separator.** Insert a triple `---` divider (three `---` lines on their own with blank lines between, or run together — either is fine) between the previous round's final section and the new `## Follow-up Review (YYYY-MM-DD)` heading.
4. **Header block for the new round.** Start with a metadata block covering: `Reviewed commit`, `Reviewed against` (which prior-round item set), `Delta reviewed` (`git diff prior..new --stat` one-liner), any force-push / rebase note per Section 4, and the `Developer response` quoted verbatim with a link to the issue-comment (`https://github.com/.../pull/{number}#issuecomment-{id}`).
5. **Force-push handling.** If the branch was force-pushed and prior-round SHAs no longer exist on `origin`, follow Section 4's reconstruction rules — but still keep the prior round's SHA citations as-written. Note the force-push in the new round's metadata header.
6. **One GitHub Review Comment per round.** Each round's `GitHub Review Comment` sub-section is independent. Do not merge or supersede the prior round's comment in-place. The reader can post the new comment on top of the prior thread; the historical comment stays in the file for reference.
7. **Back-to-top links** still target the single H1 anchor at the top of the file, per Section 7.

Result: a single file whose sections read top-to-bottom in chronological order, each round self-contained, with the newest round's GitHub Review Comment as the tail block.

---

## 7. Quality Rules

**Never speculate.** If you have not read the code, do not describe it. If the PR touches a file you cannot find in the cloned repos, say so.

**Cite specific code.** Every issue must reference the exact function, file path, and the problematic code block. Do not write vague observations like "the error handling could be improved."

**Show the fix.** For every blocking or non-blocking issue, show what the correct code should look like. The author should be able to apply the fix directly from the review comment.

**Distinguish new from pre-existing.** If a problem exists in the codebase before this PR, say it is pre-existing. Do not ask authors to fix code they did not touch.

**Severity is binary at the verdict level.** Either an item blocks merge or it does not. Do not leave ambiguity. Items with `Major` severity are blocking. Items with `Minor` or `Informational` severity are non-blocking recommendations.

**Respect the PR scope.** Do not request refactors or cleanups that go beyond what the PR set out to do. Note out-of-scope concerns as observations, not blockers.

**The GitHub Review Comment is the deliverable.** The full review file is the analysis. The GitHub comment is what the author sees. Make the comment actionable and self-contained.

**Back-to-top links are required.** Every `##` section must have one. Use them consistently.

**Section separators.** Use a single `---` between sections within a review round. Use triple `---` between review rounds (three `---` lines separated by blank lines, as shown in Section 6).

**Back-to-top targets in multi-round files.** All back-to-top links point to the single H1 anchor at the very top of the file, regardless of which round the section belongs to. Do not create per-round back-to-top anchors — one file, one top.

**Dates.** Use today's actual date for all review date fields (format: YYYY-MM-DD). Convert relative dates in PR comments to absolute dates before writing them.
