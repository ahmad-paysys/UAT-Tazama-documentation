# Issue Working Instructions

When given a GitHub issue link, follow these instructions exactly to produce a complete issue directory. Read this file before starting any work.

---

## 0. Preflight

Before creating any files, run these checks:

1. **Verify the issue is open.** Fetch the issue (see Section 2) and check `State`. If it is `Closed`, stop and confirm with the user before producing any docs — the issue may already be resolved.
2. **Scan for cross-issue dependencies.** List existing issue directories for this repo:
   ```bash
   ls issues/{repo-name}/ 2>/dev/null
   ```
   Skim the titles/summaries of open issues in the same repo. If any share files with the issue you're about to process, flag it now and plan to document the overlap in the Cross-Issue Dependencies section of `impact-{number}.md`.
3. **Verify the HTML template exists.** Confirm `issues/sample_report.html` is present. If it is missing, stop and ask the user — do not invent a substitute template.

---

## 1. Setup

Create a directory at:
```
issues/{repo-name}/{issue-number}/
```

Where `repo-name` is the repository name (e.g. `case-management-system`, `connection-studio`) and `issue-number` is the issue number (e.g. `220`).

All files you produce go into that directory.

---

## 2. Fetch the Issue

Run:
```bash
gh issue view {number} --repo tazama-lf/{repo-name} --comments
```

Read every comment. The issue body may contain a partial or full design from the author — treat it as authoritative intent. Do not contradict it.

---

## 3. Read the Code

Before writing anything, verify every claim against the actual source code. Do not describe what code *probably* does — read the files and confirm line numbers, function names, and data flows.

**Stay on `dev` for the entire issue.** All citations must come from a single, consistent snapshot. If you need to inspect another branch, tag, or commit to answer a question, note it explicitly in the output (e.g. "as of commit `abc123` on branch `feature/x`") and return to `dev` before continuing.

**If code the issue references has moved or been removed:**
- If the referenced file/function has been renamed, moved, or refactored but the underlying concern still applies, document *both* the location the issue cites and the current location, and proceed against current `dev`.
- If the underlying concern has already been fixed on `dev`, stop and confirm with the user before producing docs — the issue may be stale.
- If you cannot locate the referenced code at all, say so explicitly in `issue-{number}.md` under "How It Works Today" and ask the user how to proceed.

The repositories are cloned under the `repos/` directory. Navigate to them to read files directly.

Before reading any code, ensure the local repo is in sync with the remote `dev` branch:

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

Only after the local repo is synced with `origin/dev` may you proceed with processing the issue. If the sync fails (dirty working tree, diverged history, auth failure), stop and report it — do not read stale code.

For every claim you make in the output documents:
- You must have read the file it refers to
- You must cite the exact file path and line number
- If you cannot find the code, say so explicitly — do not speculate

---

## 4. Produce the Output Files

You produce **four files** for every issue, plus `github-issue-{number}.md` which is always produced — see section 5 for its format, which varies by issue size.

**Scope guardrail.** Before writing `solution-{number}.md`, estimate the size of Track B. If Track B touches more than 20 files, requires a schema migration *plus* a frontend sweep, or otherwise looks like it will produce an unreviewable wall of before/after diffs, **stop and confirm scope with the user first.** Options to offer: (a) proceed as one large document, (b) split solution into multiple phased documents, (c) narrow Track B to a specific sub-scope. Do not silently produce a 3000-line solution file.

### File 1: `issue-{number}.md`

The main issue breakdown. Sections in order:

```
# Issue #{number} — {title}

**Repository:** tazama-lf/{repo-name}
**Issue:** [{title}]({github url})
**Author:** {author}
**State:** Open/Closed
**Report Date:** {today's date}

---

## Executive Summary
One to three paragraphs. State the problem plainly. What is broken, why it matters, what the fix involves at a high level. If the issue has a settled design from the author, summarise it here.

---

## How It Works Today — Confirmed in Code
Show what the broken code actually does. Quote the relevant lines. Cite exact file paths and line numbers. Do not describe what the code should do — only what it currently does.

Use sub-sections if multiple systems are involved.

---

## Root Cause Analysis
One tight section. State the root cause in one sentence. Then explain how every downstream symptom flows from it. Do not repeat code already quoted above.

---

## Blast Radius — Full File Inventory
Two tables: files that must change, files that are indirectly affected. For each file: path, what is affected, relevant line numbers.

---

## Side-Effect Map
A table or diagram showing what breaks if the issue is left unfixed. One row per consequence.

---

## Effort Assessment
Two tracks minimum (Track A = minimal/immediate fix, Track B = full correct fix). For each track:
- What changes, which files, how many lines
- A table of steps with file and effort estimate
- Whether a schema migration is required
- Whether frontend changes are required

End with a recommended sequencing section.

---

## Acceptance Criteria
Checkbox lists, one per track. Each criterion is specific and testable.

---
```

### File 2: `impact-{number}.md`

A focused impact study. Sections in order:

```
# Impact Study — Issue #{number}: {title}

**Issue:** [#{number}]({github url})
**Study Date:** {today's date}
**Author:** Ahmad Khalid
**Repository:** tazama-lf/{repo-name}

---

## Summary
Two to four sentences. What the issue is, what the fix does, what the two tracks are.

---

## Confirmed Root Cause
Bullet points confirming each claim in the issue against the code. Cite file paths and line numbers. If something in the issue is wrong or has changed, note it here.

---

## Track A — {short name}

### What Changes
What files, what kind of change, how many lines.

### Impact
A table: Files changed / Schema migration required / Frontend changes required / Downtime required / Risk of regression / Reversibility.

Then two bullet lists: what this fixes immediately, what this does not fix.

End with: "Track A is safe to ship in isolation."

---

## Track B — {short name}

### What Changes
Overview sentence.

### Schema Impact
Table of schema changes if any. Migration notes and risks.

### Backend Code Impact
Tables of core rewrites, new files, and enum/reference sweep files.

### Frontend Code Impact
Table of key UI files. Note on test fixture scope.

---

## Side Effects and Risks

### Risks of Track A Alone
Table: Risk / Likelihood / Mitigation

### Risks of Track B
Table: Risk / Likelihood / Mitigation

### Cross-Issue Dependencies (if any)
If this issue shares files with another open issue, describe the merge conflict risk and recommended sequencing.

---

## Effort Estimate
Table: Track / Files / Effort. Totals at the bottom.

---

## Acceptance Criteria (Verification Checklist)
Checkbox lists per track. These should match the acceptance criteria in issue-{number}.md.

---

## Recommended Sequencing
Numbered list. Which track ships first, any dependency on other issues, any deferred steps.
```

### File 3: `solution-{number}.md`

Exact code changes. This is the implementation reference. Sections in order:

```
## Complete Fix — Exact Code Changes

Brief intro: Track A can merge independently, Track B must land as a single coordinated release (or note if it can be split).

---

### Track A — {short name} (PR 1 of N)

For each file changed:
- File path as a header
- The exact change: show BEFORE and AFTER code blocks
- Explain what the change does and why, in one or two sentences after the code block
- Note the impact of Track A alone at the end of this section

---

### Track B — {short name} (PR 2 of N)

Numbered steps (B1, B2, ...). For each step:
- Step name as a header (e.g. "Step B1 — Schema Migration")
- File path
- BEFORE/AFTER code blocks showing the exact change
- Any migration commands to run
- Any caveats or ordering dependencies

Include a data migration section if the schema changes. Show the exact SQL.

Include a test cases section at the end of solution-{number}.md covering:
- Unit tests (new tests to add, with example test code)
- Integration / E2E test scenarios (written as numbered scenario steps)
- Data migration validation SQL
- Manual / UAT checks table: # / Scenario / Steps / Expected Result

End with an "Overall Impact of the Fix" table: Area / Before / After.

End with a "Fix Summary" section: two to four paragraphs in plain prose describing what was broken, what Track A does, and what Track B does. Written so a non-technical reader can understand the scope.
```

### File 4: `impact-report.html`

A self-contained HTML file rendering the content of `impact-{number}.md` as a readable report. Requirements:

- Single file, no external dependencies except the Mermaid CDN for diagrams
- Fixed top bar with: Expand All / Collapse All buttons, Light / Dark / Solar theme toggle, Back to Top button
- Each major section is a `<details>` element (open by default), with a `<summary>` containing an `<h2>`
- Sub-sections use nested `<details>` with `<summary>` containing `<h3>`
- Tables use `<thead>` with dark header row, alternating `<tbody>` row colours
- Inline badges for risk levels: green = low risk, amber = medium, red = high, blue = informational/expected
- Any flowchart or dependency diagram rendered as a Mermaid diagram inside a `.diagram-wrap` div (horizontally scrollable, drag-to-pan)
- Theme persisted to `localStorage`
- Smooth scroll with scroll-padding-top to account for the fixed bar
- CSS uses CSS variables for theming (light, dark, solar variants)
- The top bar and `<details>` expand/collapse/theme logic implemented in a single inline `<script>` block at the bottom of `<body>`

Use `issues/sample_report.html` as the exact template for CSS variables, top bar markup, and script logic. Do not invent a different structure.

---

## 5. `github-issue-{number}.md` — Always Produced

This file is always produced for every issue. Its content varies based on how much work the issue requires.

---

**Classification trigger (mechanical — do not eyeball).** An issue is **significant** if *any* of the following are true:
- Track B touches more than 3 files, OR
- Track B requires a schema migration, OR
- Track B requires frontend changes, OR
- Track B spans more than one service/repo.

Otherwise the issue is **small**. Apply this test before choosing between the two formats below.

---

### Small issues (single fix, no significant Track B, or work small enough to leave as a comment on the linked issue)

Produce a **single user story** — one concise block ready to post as a GitHub comment or link to a ticket. Format:

```
# GitHub Issue Comment — Issue #{number}

## User Story

**Title:** {short imperative title}

**Body:**
{What the problem is and what the fix does — two to four sentences.}

**Scope:**
- Files: {list the files}
- Change: {one line describing the nature of the change}
- Migration required: Yes / No
- Frontend changes: Yes / No

**Acceptance Criteria:**
- [ ] {specific, testable criterion}
- [ ] {specific, testable criterion}

**Testing:**
- {How to verify this works}
```

---

### Significant issues (Track B involves multiple services, a schema migration, a frontend sweep, or more than roughly a day of effort)

Produce a `github-issue-{number}.md` **and** a separate `user-stories-{number}.md`.

`github-issue-{number}.md` contains the GitHub issue body and PR descriptions formatted as ready-to-post Markdown:

```
# GitHub Issue & PR Templates — Issue #{number}

---

## GitHub Issue Body

A rewrite or update of the original issue body suitable for posting or updating on GitHub. Sections:

### Problem
What is broken and why it matters. No code yet.

### Root Cause
One paragraph, plainly stated.

### Impact
Bullet list of downstream effects.

### Proposed Fix
Two tracks. For each: what it does, what files it touches, whether it needs a migration, estimated effort.

### Acceptance Criteria
Checkbox list.

---

## PR Descriptions

One PR description per track (Track A PR, Track B PR). Each contains:

### PR Title
Short (under 70 characters).

### PR Body
- **Summary**: 3-5 bullet points describing what changed
- **Test Plan**: bulleted checklist of manual verification steps
- **Related Issue**: link to the GitHub issue
- **Migration**: (Track B only) migration steps for the reviewer
- **Sequencing Note**: (Track B only) what must be merged before this PR
```

`user-stories-{number}.md` contains one user story per discrete piece of work. Follow the format in `issues/case-management-system/220/user-stories-220.md` as the template. Each user story has:
- A title in the format `US-{number}-{seq} — {short title}`
- **Body**: what the problem is and what the fix does
- **Scope**: which track, which files, what kind of change
- **Acceptance Criteria**: checkbox list, specific and testable
- **Testing**: bullet points describing how to verify

---

## 6. Quality Rules

**Never speculate.** If you have not read the file, do not reference it. If the code does not match the issue description, note the discrepancy — do not silently adopt the wrong version.

**Cite everything.** Every file path in a table or prose must be a path you have confirmed exists. Every line number must be one you have read.

**Be precise about what Track A does not fix.** The "What this does not fix" list in impact-{number}.md is as important as what it does fix. It tells the reader what debt remains.

**Match the format exactly.** Future readers will navigate these documents expecting the same structure across issues. Do not invent new sections or rename existing ones.

**Dates.** Use today's actual date for Report Date and Study Date fields.

**Cross-issue dependencies.** If Track A or Track B touches a file that another open issue also modifies, call it out explicitly in the impact file and recommended sequencing. The Preflight step (Section 0) already covers the initial scan — revisit it once you know the full file list for both tracks and update the Cross-Issue Dependencies section accordingly.

## 7. Committing your work

`commits.md` at the repository root is the source of truth for how commits are made in this repo. After processing an issue — before you `git add` anything — read `commits.md` and follow it. Do this every time, even if you have processed issues in this repo before. The rules there override any general commit habits.

In particular: one file per commit, never commit anything under `repos/`, `solutions/`, or `UAT-BOARD/`, never commit secrets, and never push to a remote.
