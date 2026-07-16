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
5. **Check for an existing review file.** Look for `pull-requests/{REPO-ABBREVIATION}-PR-{number}.md`. If it exists, this is a follow-up round — go to Section 5 (Multi-Round PR Structure) and follow the append-mode rules in 5.3; do not create a new file or overwrite the existing one. If it does not exist, this is a fresh review — continue with Section 1.

---

## 1. Setup

Create a file at:
```
pull-requests/{REPO-ABBREVIATION}-PR-{number}.md
```

If one already exists, **stop and go to Section 5** — do not create a new file (see Preflight step 5).

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
gh api repos/tazama-lf/{repo-name}/pulls/{number}/comments   # inline review comments
gh api repos/tazama-lf/{repo-name}/issues/{number}/comments  # PR-thread comments (author replies to reviews often land here)
```

Read all comments, review threads, and inline code comments — treat the PR description, commit messages, and comment threads as authoritative context.

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

Only proceed once the repo is synced with `origin/dev` and the PR branch is checked out with SHAs verified. If any step fails (dirty working tree, diverged history, auth failure, branch conflict), stop and report it. Once checked out, the working tree is authoritative — `gh pr diff` is a convenience, and if it disagrees with the working tree (which can happen if the PR was updated between fetches), re-run the fetches and re-checkout rather than mixing the two sources.

If a security-relevant function is touched (auth guards, input handling, URL construction, HTML rendering), read enough surrounding code to understand the full data flow.

### 3.1 How to hunt for issues

Before finalising Issues, run these hunts against the PR's touched files. The bugs they catch are *absence-of-line* defects — code that should have been added at a call site but wasn't, so the diff has no marker.

**Parallel-siblings check.** When a change introduces or modifies a shared value (a helper, filter object, scope variable, config, guard) that is applied at *multiple* call sites, enumerate every call site and confirm the value reaches each one. Grep the shared symbol across the touched function/module, list every consumer, and diff the consumers against each other. Ask "which siblings *don't* use this, and is that intentional?" — not just "does the diff apply this correctly where it changed lines?" Applies to shared filter objects reused across Prisma queries, auth guards applied to some routes but not others, config knobs consumed by some code paths but not others, and any "helper + N call sites" pattern. Typical failure: one query out of a group of siblings silently omits the shared scope object, producing internally inconsistent rows when the filter is exercised.

**Type/prop drift check.** When a shape (interface, prop type, function signature) is loosened, tightened, or renamed at one layer, grep every consumer of that shape for the *old* declaration and confirm each was updated in the same PR. Type changes propagate silently through TypeScript's structural compatibility — a prop typed `{ x: string }` still accepts a caller passing `{ x?: string }`, so the compiler won't flag the drift. Concretely: after seeing a type change in a hook or service, grep the pages/components/services that consume it and diff their local declarations against the new shape. Typical failure: the top-layer type is loosened but one or more consumer prop declarations retain the strict shape, forcing callers to satisfy both.

**Label/boundary drift check.** When a numeric boundary, threshold, or enumeration changes in code, grep for user-facing strings that describe the old boundary (labels, Swagger descriptions, tooltips, tick formatters, log messages, docs). Code and its display strings drift apart because the compiler doesn't check them. Typical failure: a bucket boundary is tightened (e.g. `>15 && <30`) but the display label still names the old range (e.g. `"16-30 days"`), so users read a claim the code no longer honours.

**Guard/scope asymmetry check.** When a PR adds a new endpoint, route, mutation, or query in a group of siblings (a REST controller, a resolver file, a set of Prisma queries in one service), check that authorisation guards, tenant scoping, role-based filters, and rate limits applied to the existing siblings are also applied to the new one. New endpoints and new queries are the two most common places to leak data by accident. Ask for each new sibling: "which existing sibling is the closest analogue, and does the new one carry the same guards, decorators, and scope filters?"

The four checks above are the **specific hunts**. Underlying them is one **mindset — absence-of-line reading**: after reading the `+` lines, read the *unchanged* code immediately adjacent to each hunk with the same care, and ask "given what the diff changed, what other line in this function should have been changed too?"

**When each hunt applies.** Every specific hunt is a few greps and a visual diff of the results. Run all four specific hunts on any PR that touches shared filters, shared types, boundary constants, or endpoint groups. Skip only when the PR is genuinely narrow (a single lockfile bump, a single string change, a single test-only edit).

---

## 4. File Format

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
**HEAD SHA verified:** `{full-sha}`
**Existing approvals:** {see field rule below}
```

**`HEAD SHA verified:`** — record the full HEAD SHA of the commit you actually reviewed. This is the SHA you compared against `gh pr view --json headRefOid` at the end of Section 3. Prevents ambiguity if the PR is force-pushed between review and merge.

**`Existing approvals:`** — one of:
- `none` — no reviewer has approved yet. Note in-flight review state (e.g. `none — CodeRabbit COMMENTED, no human approval yet`).
- `{reviewer} — APPROVED ({YYYY-MM-DD})` per approving reviewer.
- `{reviewer} — CHANGES_REQUESTED ({YYYY-MM-DD})` if a prior review is outstanding.

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

**Anchor-slug rules** (GitHub's algorithm for converting a heading to a fragment):

- Lowercase everything.
- Replace each space with a single `-`.
- Drop punctuation entirely: em-dash (`—`), en-dash (`–`), colon, comma, period, apostrophe, quotes, parentheses, brackets. Do **not** substitute them with `-`.
- An em-dash surrounded by spaces (` — `) collapses to `--` in the slug (the spaces on either side each become `-`, and the em-dash itself drops out — leaving two dashes back-to-back). This is why the example above has `--fix-transaction-history…`.
- Preserve `#` numbers (`#221` → `221`) and letters as-is.

If in doubt, view the rendered file on GitHub, hover the H1's link icon, and copy the fragment.

Place the link after the section content and before the `---` separator.

---

### Overview

One to three paragraphs. State plainly: what the PR does, what files it touches, what problem it solves. If the PR description contains useful context, summarise it here. If the PR bundles unrelated changes (e.g., a feature + test expansion + lockfile churn), identify each distinct concern.

**Always state the base branch explicitly** (e.g. "Targets `dev` — correct for this repo's flow"; if not, flag as a potential blocker). Also note any failing CI checks recorded in Preflight.

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

**How to hunt for security issues on this PR** (in addition to the Section 3.1 hunts):

- **Unvalidated input reaching a data or command layer.** For every new query parameter, request body field, or URL segment introduced or accepted by this PR, trace it from the entry point (controller / route handler) to where it lands (Prisma `where`, filesystem path, shell command, external HTTP call). Ask: is it validated against an enum, schema, or allowlist before use? A raw string reaching Prisma isn't a SQL injection (Prisma parameterises) but is an error-surface concern; a raw string reaching `child_process`, `fs`, or a URL constructor is a real vulnerability.
- **Tenant / role scoping on new or modified queries.** For every new `findMany` / `count` / `groupBy` / `update` / `delete`, confirm the `where` clause includes the tenant scope from the auth token (never from a query param) and the role-based scope helper (e.g. `applyInvestigatorScope`). This overlaps with the Section 3.1 guard/scope asymmetry hunt but is severity-elevated in the Security section.
- **Auth-guard decorators / middleware on new endpoints.** For every new route, confirm it carries the same auth guard, role guard, and rate limit as its siblings in the same controller. Missing decorators are the most common way this codebase leaks endpoints.
- **URL construction / open-redirect / SSRF.** Any new `new URL(...)`, `fetch(...)`, `axios.get(...)`, `window.location.assign(...)`, `res.redirect(...)`, or href set from a variable — check that the source is either a compile-time constant, a validated allowlist, or a trusted server response. Untrusted-input URL construction is an SSRF or open-redirect.
- **HTML rendering / XSS.** Any new `dangerouslySetInnerHTML`, `v-html`, `innerHTML`, `document.write`, or React string concatenation that ends up in the DOM — check the source is trusted or sanitised. In this codebase almost everything renders through React's escaping, so flag any escape from that path.
- **Secrets / PII in logs, errors, and API responses.** Any new `console.log`, `logger.info`, error message, or response body — check it doesn't include tokens, session IDs, hashed credentials, or PII fields that weren't previously exposed. Also: validator errors that echo user-controlled input verbatim — usually fine, but note when present.
- **CI security-scanner status.** Record the state of any security scanner in the PR's `statusCheckRollup` (CodeQL, njsscan, dependency-review) and note if any are failing or newly warning.

If a security concern exists (XSS, injection, auth bypass, request timeout, claim mismatch), state it clearly, note whether it is new or pre-existing, and state whether this PR fixes or worsens it.

If no new vulnerabilities are introduced, say: "No new security vulnerabilities introduced by this PR." Do not omit the section.

---

### Test Coverage

Always include this section. State:
- What is tested
- What is not tested (especially for the core change)
- Whether the PR checklist is completed (and which boxes are checked or unchecked)
- Whether the coverage screenshot or CI evidence is present

**How to hunt for test-coverage gaps on this PR:**

- **Every `Major` in Issues must have a matching test.** If the PR fixes a Major (regression, filter bypass, boundary correction), a spec assertion must lock the fix in place. If not, that's a test-coverage gap of the same severity as the underlying issue — call it out in Blocking.
- **New public surface must be tested.** Every new endpoint, new hook signature, new service method, new exported helper — grep for corresponding spec files. A public export with no direct test is a test-coverage gap.
- **New branches must be tested.** For every new `if`, `try`, `throw`, or ternary in the diff, ask whether both branches are exercised. Validators that throw on bad input frequently have the *happy path* tested implicitly through higher-level tests but the *throw branch* uncovered.
- **Every-mock-call assertions on shared filters.** If the PR modifies a filter helper applied across multiple Prisma calls (per the 3.1 parallel-siblings hunt), the spec must assert the filter on *every* mock call, not just one. A single `toHaveBeenCalledWith` match passes even if three sibling queries missed the filter.
- **Client-side vs. backend parity.** If a client-side implementation mirrors a backend function, both paths need coverage — otherwise the two silently drift.
- **Regression tests for prior bugs.** If the PR fixes a bug caught in a prior review (tooltip desync, filter bypass), the spec must include an assertion that would fail if the fix is removed. Not just "the new code has coverage" — "the old bug cannot come back."

If the core functional change has no test, call it out explicitly — do not soften it.

---

### CodeRabbit Activity (include only if CodeRabbit reviewed this PR)

**Treat CodeRabbit as a safety net, not a co-reviewer.** CodeRabbit is one input, not a substitute for the reviewer's own analysis. Two consequences:

1. **Draft your own Issues and Observations *before* reading CodeRabbit's review.** If you read CodeRabbit first, its framing anchors your analysis and you will drift toward corroborating rather than hunting. Do the 3.1 hunts, write the Issues list, *then* open the CodeRabbit review to reconcile.
2. **A CodeRabbit-caught `Major` that you missed is a signal, not a resolution.** Note the specific hunt from Section 3.1 that would have caught it, and widen that hunt on the next review. If the class isn't listed in 3.1, that's a gap in the instructions — flag it back to the user rather than absorbing it silently.

**Reconcile CodeRabbit findings against your own.** For each CodeRabbit finding:
- If you also flagged it independently, note it as **corroborated** in both sections.
- If CodeRabbit caught something you missed, add it to your Issues list at the appropriate severity — do not leave it only in the CodeRabbit section. Also record which 3.1 hunt should have caught it (or that no hunt covers this class).
- If you flagged something CodeRabbit missed, that's fine — do not remove it.
- If you disagree with a CodeRabbit finding, state your reasoning explicitly. Verify the premise against current code — CodeRabbit sometimes reasons from a stale or hypothetical version of the surrounding code, and the disagreement should cite the current implementation. Do not silently omit the finding.

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
**Verdict: {Approved / Changes Requested}**

{One to two paragraphs summarising the overall quality of the PR and the reasoning behind the verdict.}

### Blocking

1. **{Issue title}** — {one-sentence explanation}

### Non-blocking but recommended

2. **{Issue title}** — {one-sentence explanation}
```

If there are no blocking items, say so explicitly. Do not list non-blocking items as blocking.

Verdict wording conventions:
- `Approved` — no blocking items. Non-blocking items (Minor / Informational) may still be listed as recommendations; they do not gate merge.
- `Changes Requested` — one or more blocking items must be addressed before merge.

The verdict is strictly binary: an item either blocks merge or it does not. If you feel the need for a middle verdict ("approve but with cleanup"), reclassify the items — either they are blocking (verdict is `Changes Requested`) or they are not (verdict is `Approved` with a non-blocking list).

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

**Non-blocking cap:** at most three items by default (see Section 7, item 8, for the five-item stretch on very large PRs).

**Backtick fence rule:** The outer fence wrapping the entire GitHub Review Comment block must use **at least one more backtick** than any fence inside it. If the comment body contains a ` ```diff ` or ` ```typescript ` block (3 backticks), the outer fence must use 4 backticks (` ```` `). If the body contains a 4-backtick fence, use 5 for the outer, and so on. Violating this causes the inner fence to close the outer one, breaking the raw-markdown display.

---

## 5. Multi-Round PR Structure

### 5.0 Identifying and structuring rounds

A PR may have one or more review rounds if the author pushed additional commits after an initial review. Structure the file as one section per round (Initial Review, Follow-up Review, Final Review, etc.), separated by triple `---` dividers:

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

**Handling force-pushed / rebased PRs.** If the author force-pushed and prior-round commit hashes no longer exist in the repo (`git cat-file -e {hash}` fails), do not silently produce broken commit citations. Instead:

1. Fetch prior review activity from GitHub's API — reviews and inline comments remain accessible by review ID even after force-push:
   ```bash
   gh api repos/tazama-lf/{repo-name}/pulls/{number}/reviews
   gh api repos/tazama-lf/{repo-name}/pulls/{number}/comments
   ```
2. In the follow-up section, note the force-push explicitly: `**Note:** Prior round reviewed commit `{old-hash}` which no longer exists after force-push on {date}. Prior findings reconstructed from GitHub review API.`
3. Reconcile each prior finding against the new HEAD by content (file + function + concern) rather than by commit range. Cite the current HEAD SHA for the new evidence.
4. If the rewrite is extensive enough that prior findings cannot be mapped, treat this round as a fresh review and note the reason.

### 5.1 Read ALL comments before writing the follow-up

**Before drafting any follow-up review, re-fetch and re-read every comment on the PR — do not rely on the state captured in the previous round's review file.** Between rounds, authors and reviewers routinely post issue comments, inline review comments, and review-thread replies that resolve items *by explanation* rather than by code change.

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

### 5.2 Follow-up section contents

Each follow-up section contains:
1. Resolution Status (one `### Item N` subsection per prior item) — every item must cite either the commit that fixed it OR the comment (with URL) that answered it OR an explicit `no author response` note. End with a summary table.
2. New Issues Found in Updated Commits (if any new problems were introduced) — treat these as first-class findings with severity + fix, same standards as initial-round issues.
3. Updated Verdict (new verdict + blocking / non-blocking split with one-paragraph reasoning).
4. **GitHub Review Comment (Follow-up)** — a per-round, ready-to-paste GitHub comment. Each follow-up round produces its OWN self-contained comment, headed e.g. `**Changes Requested (follow-up, HEAD `{sha}`)**`, listing only the still-blocking items and any new-issue blockers from this round. Do not rely on the initial round's comment being read alongside it — the follow-up comment must stand on its own so the author sees exactly what is still owed.

The Final Review section (only when the PR is closed out) replaces the Summary and Verdict section from the initial review and includes the definitive verdict with the complete resolution table.

### 5.3 Append mode — do not rewrite prior rounds

**When adding a follow-up round to an existing review file, append; never rewrite.** The prior round's Overview, What-Changed, Issues-and-Observations, Summary-and-Verdict, and GitHub-Review-Comment sections stay intact as the historical record — they capture what was known and posted at that time, and are the anchor for the author's follow-up response. Concretely:

1. **Preserve everything above.** Do not edit prior-round headings, tables, verdict wording, or the initial GitHub Review Comment. If a prior-round claim turned out to be wrong, note the correction in the follow-up section rather than rewriting history.
2. **Extend the Table of Contents.** Add nested entries for the new round (e.g. `- [Follow-up Review (YYYY-MM-DD)](#follow-up-review-yyyy-mm-dd)` with sub-entries for Resolution Status, New Issues, Updated Verdict, GitHub Review Comment). Do not remove the original single-round ToC entries.
3. **Separator.** Insert a triple `---` divider (three `---` lines separated by blank lines) between the previous round's final section and the new `## Follow-up Review (YYYY-MM-DD)` heading.
4. **Header block for the new round.** Start with a metadata block covering: `Reviewed commit`, `Reviewed against` (which prior-round item set), `Delta reviewed` (`git diff {prior-round-HEAD-SHA}..{new-HEAD-SHA} --stat` one-liner — note `prior` is the prior round's HEAD SHA, not the base branch), any force-push / rebase note per Section 5.0, and the `Developer response` quoted verbatim with a link to the issue-comment (`https://github.com/.../pull/{number}#issuecomment-{id}`).
5. **Force-push handling.** If the branch was force-pushed and prior-round SHAs no longer exist on `origin`, follow Section 5.0's reconstruction rules — but still keep the prior round's SHA citations as-written. Note the force-push in the new round's metadata header.
6. **One GitHub Review Comment per round.** Each round's `GitHub Review Comment` sub-section is independent. Do not merge or supersede the prior round's comment in-place. The reader can post the new comment on top of the prior thread; the historical comment stays in the file for reference.
7. **Back-to-top links** still target the single H1 anchor at the top of the file, per Section 6.

---

## 6. Quality Rules

**Never speculate.** If you have not read the code, do not describe it. If the PR touches a file you cannot find in the cloned repos, say so.

**Cite specific code.** Every issue must reference the exact function, file path, and the problematic code block. Do not write vague observations like "the error handling could be improved."

**Show the fix.** For every blocking or non-blocking issue, show what the correct code should look like. The author should be able to apply the fix directly from the review comment.

**Distinguish new from pre-existing.** If a problem exists in the codebase before this PR, say it is pre-existing. Do not ask authors to fix code they did not touch.

**Respect the PR scope.** Do not request refactors or cleanups that go beyond what the PR set out to do. Note out-of-scope concerns as observations, not blockers.

**The GitHub Review Comment is the deliverable.** The full review file is the analysis. The GitHub comment is what the author sees. Make the comment actionable and self-contained.

**Section separators.** Use a single `---` between sections within a review round. Use triple `---` between review rounds (three `---` lines separated by blank lines, as shown in Section 5).

**Back-to-top targets in multi-round files.** All back-to-top links point to the single H1 anchor at the very top of the file, regardless of which round the section belongs to. Do not create per-round back-to-top anchors — one file, one top.

---

## 7. Before You Submit — Self-Check

Run through this list before writing the GitHub Review Comment. Each item takes seconds and catches a specific class of defect that has bitten reviews in this directory.

1. **Verdict ↔ severity coherence.** Every `Major` in Issues and Observations must appear as a Blocking entry in Summary and Verdict, and vice versa. No `Minor`/`Informational` items sneak into Blocking. If the Blocking list is empty, the verdict is `Approved`, not `Changes Requested`.
2. **Prior-round coverage (multi-round only).** Every item from the prior round has an explicit Resolution Status entry (RESOLVED / PARTIALLY RESOLVED / NOT RESOLVED / DECLINED / DEFERRED). No prior item is silently dropped.
3. **Hunt coverage (Section 3.1).** Confirm you ran the four hunts — parallel-siblings, type/prop drift, label/boundary drift, guard/scope asymmetry. If CodeRabbit's review contains a `Major` you didn't independently catch, this check has failed; add the missed finding and record which hunt was skipped or is missing from 3.1.
4. **Backtick-fence depth on the GitHub Review Comment.** Outer fence has strictly more backticks than any inner fence — see the Section 4 GitHub Review Comment rule.
5. **Anchors resolve.** Every ToC entry corresponds to a real heading in the file; every back-to-top link targets the single H1 anchor.
6. **Dates are absolute.** No relative dates ("yesterday", "last week", "Thursday") in the file — all converted to `YYYY-MM-DD` before writing. Get today's date from the environment (system context or `date` command) rather than guessing.
7. **GitHub Review Comment is self-contained.** Cite file paths, line ranges, and short code examples in the comment body — the author must be able to act on the comment without opening the review file.
8. **Non-blocking cap.** The GitHub Review Comment lists at most three non-blocking items by default. Bundle related trivia into one item, or leave them in the review file only. On very large PRs (~1500+ lines or ~30+ files) the cap may stretch to five if each remaining item represents a distinct defect class (type drift, label drift, dead code, test gap, etc.) — pick one representative per class and put the rest in the review file with a one-line pointer in the comment ("further minor observations are recorded in the review file"). The author reads the first three and skims the rest, so the cap protects actionability, not tidiness.
