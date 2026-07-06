# Daily Report Construction — Agent Instructions

Instructions for constructing the Beta Testing Issue Tracking daily report from the GitHub project board.

- **Board:** [tazama-lf/projects/20](https://github.com/orgs/tazama-lf/projects/20/views/1)
- **Output location:** `UAT-BOARD/daily-reports/report-YYYYMMDD.md` (today's date)
- **Template reference:** the most recent existing report in this directory (currently [`report-20260706.md`](report-20260706.md)) — mirror its structure exactly.

---

## 1. Determine the activity window

The window is the range `[cutoff, now]`. An issue qualifies for inclusion if **either** its `updated_at` **or** any of its comment `created_at` timestamps falls in the window.

- **If today is Monday:** cutoff = 4 days ago at 00:00 UTC (covers Fri/Sat/Sun/Mon).
- **Any other day:** cutoff = yesterday at 00:00 UTC (covers yesterday + today).

Convert relative dates to absolute (`YYYY-MM-DD`) and state the window explicitly in the report header.

## 2. Pull the board and issue activity

1. `gh project item-list 20 --owner tazama-lf --format json --limit 200` — full board snapshot; extract `(repo, number, status, assignees, title)` for every item where `content.type == "Issue"`.
2. For each issue, call `gh api repos/{repo}/issues/{num}` to get `updated_at`, `state`, `closed_at`, `comments` count.
3. For any issue with `comments > 0`, call `gh api repos/{repo}/issues/{num}/comments` and record the latest comment's `created_at`, `user.login`, and a truncated `body` (~300–400 chars).
4. Filter to issues where `updated_at >= cutoff` OR `latest_comment.created_at >= cutoff`.

## 3. Resolve parent / sub-issue relationships

For every in-window issue, query GraphQL for both directions:

```graphql
{ repository(owner: "…", name: "…") { issue(number: …) {
    parent { number repository { nameWithOwner } title }
    subIssues(first: 30) { nodes { number title state } }
} } }
```

Grouping rules (from the user):

- **If an issue has sub-issues, present those sub-issues as subsections under it.** Do not repeat a sub-issue as its own top-level section later, even if it appears on the board independently.
- **If a parent isn't itself in the window but ≥1 of its sub-issues is:** include the parent as a top-level section anyway, and nest the in-window sub-issues under it. Note in the parent block that the parent's own timestamp is outside the window (see how `PMO #823` is handled in the template).
- **If an in-window issue has a parent that is off-board:** keep the issue as its own top-level section, and mention the off-board parent in the section body (see `CMS #217` / `CMS #219` referencing off-board EPIC `#218`).

## 4. Report structure

Follow the template exactly. Section order and formatting:

1. **Header** — title, board link, report date, window statement.
2. **Table of Contents** — every top-level section and every sub-issue subsection gets a bullet with an anchor link. Indent sub-issue bullets one level under their parent. TOC anchor targets must match the `<a id="…"></a>` tags used later.
3. **Summary section** — anchored `<a id="summary"></a>`, containing:
   - A metrics table (Issues in window, Top-level sections, Sub-issue subsections, Closed in window, Still open).
   - A "Breakdown by repo" table.
   - A **per-user breakdown** table with columns: `Assignee | Total | Open | Closed | Todo | In Progress | Internal Review | Retest | On Hold | Done`. Sort by Total desc. Include a bolded totals row. Right-align numeric columns (`---:`).
   - A "Cross-issue links flagged this window" bullet list — any comment that references another issue as a root cause, blocker, or dependency should be surfaced here.
4. **Per-issue sections** — one section per top-level issue, in the same order as the TOC. Each section contains:
   - `<a id="…"></a>` anchor immediately before the `##` heading.
   - `## <REPO_SHORT> #<num> — <short title>` heading.
   - Metadata bullets: Link, Status (board), Assignee, Updated, Last comment (date + user + trimmed body), and any parent-issue pointer.
   - A 1–3 sentence narrative summarising what changed / what's blocking / what's next. Do not restate the issue title.
   - Sub-issue subsections use `### <a id="…"></a>Sub-issue: <REPO_SHORT> #<num> — <short title>`, followed by a compact metadata line (Link · Status · Closed date · Assignee) and, if the sub-issue had a comment in-window, a Last-comment line + 1-sentence narrative.
   - End every top-level section with `[⬆ Back to top](#table-of-contents)` on its own line, preceded by a blank line and followed by a `---` separator.

Repo short forms used in headings: `CMS` for `tazama-lf/case-management-system`, `BIAR` for `tazama-lf/biar`, `PMO` for `frmscoe/paysys-pmo`, `frms-coe-lib` unchanged.

## 5. Populating the per-user breakdown

Bucket each in-window issue by its `assignee` (use the first assignee if multiple) and count:

- **Total** — every in-window issue for that assignee.
- **Open / Closed** — from `state`.
- **Todo / In Progress / Internal Review / Retest / On Hold / Done** — from the board's `Status` field (exact match).

Include only board statuses that appear this window; if a status has zero for every assignee, still keep the column (matches template) — the goal is a stable column set so week-over-week comparison is straightforward.

## 6. Cross-issue links section

When scanning comments, watch for phrases like "root cause", "blocked on", "depends on", "duplicate of", or explicit `#nnn` / URL references to other issues. Surface each such link as one bullet under "Cross-issue links flagged this window" with the direction (A → B), the source (comment author + date), and a short reason. Example patterns present in the current template:

- `biar#100` flagged as suspected root cause of `CMS#205` (Sandy-at-Tazama, YYYY-MM-DD).
- `biar#94/95/96` held pending `CMS#220 / #214` resolution.

## 7. Anchors and back-to-top

- Every top-level section: `<a id="short-slug"></a>` (e.g., `cms-146`, `pmo-823`, `biar-100`, `lib-431`).
- Every sub-issue subsection: inline anchor inside the `###` heading (e.g., `### <a id="pmo-815"></a>Sub-issue: …`).
- Every top-level section ends with `[⬆ Back to top](#table-of-contents)`.
- Sub-issue subsections do **not** need their own back-to-top link — the parent section's link is sufficient.

## 8. Output

Write the report to `UAT-BOARD/daily-reports/report-YYYYMMDD.md` where `YYYYMMDD` is today's date. Do not modify prior reports. Do not add anything to `MEMORY.md` — daily reports are project state, not memory.

## 9. Verification checklist before finishing

- [ ] Window statement in header matches the Monday-vs-other-day rule.
- [ ] Every TOC entry has a corresponding anchor and vice versa.
- [ ] No sub-issue appears both as a top-level section and as a subsection.
- [ ] Every top-level section ends with a back-to-top link.
- [ ] Per-user table totals row sums match the "Issues in window" count in the metrics table.
- [ ] Cross-issue links list is populated when comments mention other issues.
- [ ] Cross-references between issues in the report use in-document anchor links (e.g., `[BIAR #100](#biar-100)`), not raw GitHub URLs, when the target is also a section in the same report.
