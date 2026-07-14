# PR Review: CMS #242 — fix: Case Dashboard Count

**Repo:** tazama-lf/case-management-system
**Branch:** `paysys/dashboard-fixes` → `dev`
**Author:** sobia-rizwan1567 (Sobia Rizwan)
**Date Reviewed:** 2026-07-14
**Label:** bug
**Size:** +1903 / -1403 lines across 46 files
**Commits:** 17 (172900ef, 1ea1e08b, 6776f61d, 1b29bf88, 5d9930dd, a2e10892, 8344ed9f, 23250974, 7ad65ff4, 2be890c6, 47be30c7, 2540ca27, 48c1d456, b60467d1, b7a58e7e, f03cfab6, da2108bb)
**State:** OPEN (mergeStateStatus: BLOCKED — DCO / branch-protection, not a check failure; all 15 CI checks SUCCESS)
**Head SHA verified:** `da2108bb1ba530b08b7da304dd9758d3fe3e9d69` matches remote
**Existing approvals:** none (author has left 4 review-comments-only entries)

---

## Table of Contents

- [Overview](#overview)
- [Per-Issue Verdict Table](#per-issue-verdict-table)
- [Per-Issue Detailed Verdicts](#per-issue-detailed-verdicts)
  - [Issue #142 — user names show as user IDs](#issue-142--user-names-show-as-user-ids)
  - [Issue #152 — dashboard links to "cases assigned to you"](#issue-152--dashboard-links-to-cases-assigned-to-you)
  - [Issue #153 — dashboard links to "high priority alerts"](#issue-153--dashboard-links-to-high-priority-alerts)
  - [Issue #158 — transaction history cumulative value graph empty](#issue-158--transaction-history-cumulative-value-graph-empty)
  - [Issue #201 — ageing report exports show User ID in Investigator column](#issue-201--ageing-report-exports-show-user-id-in-investigator-column)
  - [Issue #209 — CMS Dashboard review and fixes](#issue-209--cms-dashboard-review-and-fixes)
  - [Issue #210 — visualizations tab on case details](#issue-210--visualizations-tab-on-case-details)
  - [Issue #217 — Case Ageing Report analysis](#issue-217--case-ageing-report-analysis)
  - [Issue #219 — Case Status Report analysis](#issue-219--case-status-report-analysis)
- [What Changed (Grouped)](#what-changed-grouped)
- [Code Quality Analysis](#code-quality-analysis)
  - [Strengths](#strengths)
  - [Issues and Observations](#issues-and-observations)
- [Security Assessment](#security-assessment)
- [Test Coverage](#test-coverage)
- [CodeRabbit Activity](#coderabbit-activity)
- [Summary and Verdict](#summary-and-verdict)
- [GitHub Review Comment](#github-review-comment)

---

## Overview

PR #242 reconciles the CMS dashboard's stat-cards, "Recent/Active Cases" panels, and Case Status Report with the actual investigator-scoping and status semantics used by the Cases list, and simultaneously repairs the notebook powering the transaction-history visualization tab. It touches 46 files across three surfaces: (a) the backend report and case-query services, (b) the frontend dashboard and report pages/tests, and (c) `notebooks/transaction-viz.ipynb` (a real logic rewrite — not just an output strip).

**Targets `dev` — correct for this repo's flow.** All 15 CI checks green (CodeQL, DCO, dependency-review, encoding, hadolint, Node.js CI build/style/tests, gpg-verify, njsscan, CodeRabbit). `mergeStateStatus: BLOCKED` reflects missing human approval, not a check failure.

The PR is bundled: nine closing issues (#142, #152, #153, #158, #201, #209, #210, #217, #219) span dashboard behavior, visualizations, exports, and multiple report analyses. Coverage of those issues is uneven — most of the dashboard reshape from #209 and the notebook fix for #158 land, while #201 and #217 are effectively not touched. See the per-issue table below for the verdicts you asked for.

| File | Nature of Change |
|------|-----------------|
| `backend/src/modules/case/services/case-query.service.ts` | Add `STATUS_99_ABANDONED` to `closedOnly` set; broaden investigator DRAFT visibility |
| `backend/src/modules/report/helpers/getDateRange.ts` | Add `'all'` case → epoch start |
| `backend/src/modules/report/report.controller.ts` | Wire `dateRange=all` and `tenantId/requestingUserId` into `getFilters` |
| `backend/src/modules/report/report.service.ts` | Major rewrite: new stat fields (`availableCases`, `openAssignedCases`, `overdueCases`, `resolvedThisMonth`, `openStatusCounts`, `openPriorityCounts`), `applyInvestigatorScope` helper, `STATUS_DISTRIBUTION_MAP` per-status, `computeOutcomes` refuted split, add `STATUS_99_ABANDONED` to `CLOSED_STATUSES`, format-status rewrite |
| `backend/src/modules/report/types/report.types.ts` | Add `'all'` to dateRange enum |
| `backend/test/report.service.spec.ts` | +142/-14 covering new fields |
| `frontend/src/features/cases/components/CaseFilters.tsx` | Add `STATUS_99_ABANDONED` to closed-status buttons |
| `frontend/src/features/cases/components/TasksDetailsModal.tsx` | Conditional Visualizations tab (issue #210) |
| `frontend/src/features/cases/components/ViewCaseModal.tsx` | Drop unused `onSwitchToCaseDetails` prop |
| `frontend/src/features/cases/hooks/useCaseDashboard.ts` | Read `status`/`priority` from URL, sync filter → URL |
| `frontend/src/features/dashboard/components/*` | Alert/Case summary items, `StatsCard[s]`, `DashboardSection` reshape (link support, +N-more, max-based scaling) |
| `frontend/src/features/dashboard/pages/Dashboard.tsx` | New "Open Cases by Priority" and "Open Cases by Status" panels replacing "Recent/Active"; anchor links wired |
| `frontend/src/features/dashboard/services/dashboardService.ts` | Send `dateRange=all`; consume new fields |
| `frontend/src/features/dashboard/types/dashboard.types.ts` | New stat + panel field types |
| `frontend/src/features/reports/components/*` | Chart tweaks (BarChart horizontal support, PieChart labels, MultiBarChart, WorkloadBarChart, FiltersPanel) |
| `frontend/src/features/reports/pages/CaseStatusReport.tsx` | Reorder sections; dynamic status labels; "Live Workload" / "Closed Throughput" headings |
| `frontend/src/features/reports/services/reportsService.ts` | New CaseStatusStats fields (missing on fallback path — see Issues) |
| `frontend/src/features/reports/types/reports.types.ts` | Add types for new fields |
| `frontend/src/shared/constants/options.ts` | Add `STATUS_99_ABANDONED` to closed group |
| `frontend/src/shared/utils/exportUtils.ts` | Header rename ("Avg Age of Current Cases") |
| `notebooks/transaction-viz.ipynb` | Real rewrite: granularity-aware bucketing across timeline/cumulative/volume with pd.date_range reindex; explicit x-axis per granularity |

[↑ Back to top](#pr-review-cms-242--fix-case-dashboard-count)

---

## Per-Issue Verdict Table

| Issue | Title | Coverage | One-line status |
|---|---|---|---|
| #142 | User names showing as user IDs | Partial (~20%) | Only the report **Filters** dropdown (`getFilters`) joins `cms_usernames` — case timeline, ageing report body, workload report, and trend dashboard still show raw IDs |
| #152 | Dashboard link → cases assigned to you | Substantial (~80%) | Dashboard tiles now anchor to `/cases?status=…`; `useCaseDashboard` reads URL params and syncs on filter change |
| #153 | Dashboard link → high-priority alerts | Substantial (~85%) | "Open Cases by Priority" tiles link to `/cases?priority=…`; old ambiguous "High Priority" card split into **Available** and **Overdue** (real `SlaState.BREACHED`) |
| #158 | Transaction cumulative value graph empty | Full (~90%) | Notebook rewritten: shared bucketing + reindex + `cumsum()` from timeline; day/month/year switch now retargets all three subplots |
| #201 | Ageing exports show User ID in Investigator col | **Not addressed** | No changes to `getCaseAgeing`, `CaseAgeingReport.tsx`, or the ageing export path (Sandy's 2026-06-30 comment confirms "not fixed") |
| #209 | CMS Dashboard review and fixes (Justus) | Substantial (~75%) | Total/High Priority/Open/Resolved/Recent/Active/CLOSED_STATUSES/`isInvestigator` all touched — but the way `STATUS_99` is folded into `CLOSED_STATUSES` contradicts #219 |
| #210 | Add Visualizations tab to case details | Substantial (~90%) | Task modal gains a conditional Viz tab; existing case-details Viz tab preserved — matches Sandy's 2026-07-10 clarification (should be on **both**) |
| #217 | Case Ageing Report requirements | **Not addressed (~5%)** | `getCaseAgeing` and its three copy-pasted OR clauses untouched; no container / STATUS_99 exclusion, no shared helper extraction, no widget-level fixes |
| #219 | Case Status Report analysis | Partial (~45%) | Status Distribution reshape, refuted-split, avg-resolution-null-on-empty, case-type-open-only landed — but `STATUS_99` added to `CLOSED_STATUSES` is the wrong direction; `STATUS_03_RETURNED` silently dropped |

[↑ Back to top](#pr-review-cms-242--fix-case-dashboard-count)

---

## Per-Issue Detailed Verdicts

### Issue #142 — user names show as user IDs

**What the issue asks.** Across case-ageing details, investigator-workload report, timeline, and dashboards, investigator IDs (UUIDs) are shown where names should be; also asks whether non-tenant users are leaking in.

**What the PR does.** The **Filters** endpoint (`getFilters` in [report.service.ts](repos/case-management-system/backend/src/modules/report/report.service.ts) and its controller wiring) now joins `cms_usernames` scoped by `tenantId` and returns the investigator's actual `name`. Previously the dropdown labelled investigators `User <id.slice(0,8)>`. Falls back to the raw UUID when no `cms_usernames` row exists.

**What's left.**
- Case Ageing details table (Investigator column) — untouched (same symptom as #201).
- Investigator Workload report body — untouched.
- Case timeline component — untouched.
- Case Management trend dashboard — untouched.
- Tenant-scoping concern (only Tazama-tenant users acting) — not addressed.

**Concerns.** If `cms_usernames` is missing a row for a user present in cases, the dropdown still displays the raw UUID — this is the exact reported symptom. Reasonable fallback but not a full guard.

---

### Issue #152 — dashboard links to "cases assigned to you"

**What the issue asks.** Clicking the count on the dashboard should land on a filtered Cases list.

**What the PR does.** `frontend/src/features/dashboard/pages/Dashboard.tsx` wraps each `CaseSummaryItem` with `<a href={`/cases?status=…`}>`. `useCaseDashboard` reads `status` and `priority` from `location.search` on mount and pushes changes back via `navigate(..., { replace: true })`. So `/cases?status=STATUS_10_ASSIGNED` lands on that filter, and since the Cases page's backend query is investigator-scoped for investigators, an investigator lands on their own cases.

**What's left.** There is no explicit `assignedTo=me` — intent is delivered indirectly via investigator scoping (which is fine for now). Only `status`/`priority` round-trip through the URL; other useful filters do not.

**Concerns.** None.

---

### Issue #153 — dashboard links to "high-priority alerts"

**What the issue asks.** Dashboard priority tiles should link to a filtered view; clarify what "high priority" means.

**What the PR does.** The ambiguous **High Priority Cases** stat card is deleted. In its place: **Available Cases** (`STATUS_02_READY_FOR_ASSIGNMENT`), **Overdue Cases** (computed from real `SlaState.BREACHED` via `computeCaseSlaState`), plus an **Open Cases by Priority** panel whose tiles link to `/cases?priority=HIGH|MEDIUM|LOW`. Priority now unambiguously means `Priority.HIGH`; SLA breach is its own card.

**What's left.** The panel shows cases, not alerts, but the issue was really about cases-under-alert.

**Concerns.** None.

---

### Issue #158 — transaction history cumulative value graph empty

**What the issue asks.** Cumulative value graph is empty on task 360 / case 323; follow-up (2026-07-10) adds that granularity switching (month → year) does not refresh the chart.

**What the PR does.** `notebooks/transaction-viz.ipynb` is materially rewritten (+507/-885, mostly output strip + real logic changes). All three subplots (Timeline, Cumulative Value, Volume Distribution) share one bucketing pass driven by `_filter in {"day","month","year"}`, then reindex against a full `pd.date_range` so empty buckets zero-fill. Cumulative series is derived from the bucketed timeline (`df_volume_derived['totalAmount'].cumsum()`) rather than a separate API payload, and explicit `range=[bucket_x_start, bucket_x_end]` on the x-axis retargets all subplots when granularity changes.

**What's left.** Manual verification against Tazama tenant task 360 across day/month/year modes.

**Concerns.** None significant. Notebook is a real logic change, not an output strip.

---

### Issue #201 — ageing report exports show User ID in Investigator column

**What the issue asks.** In CSV/XLSX/PDF exports of the Case Ageing Report the `Investigator` column shows the UUID instead of the name.

**What the PR does.** Nothing on this surface. No changes to `getCaseAgeing` in [report.service.ts](repos/case-management-system/backend/src/modules/report/report.service.ts), no changes to [CaseAgeingReport.tsx](repos/case-management-system/frontend/src/features/reports/pages/CaseAgeingReport.tsx), no changes to the ageing export rows in [exportUtils.ts](repos/case-management-system/frontend/src/shared/utils/exportUtils.ts) (the only change there is a header rename `"Avg Time in Status"` → `"Avg Age of Current Cases"` on the Case Status report). Sandy's 2026-06-30 comment already flagged this as "not fixed" and deferred to #217's rewrite.

**What's left.** Everything the issue asks for: `cms_usernames` join in the ageing rows, feed `Investigator` column with `name`, and align exports with the ageing analysis.

**Concerns.** None (nothing was changed here).

---

### Issue #209 — CMS Dashboard review and fixes

**What the issue asks.** A structured critique of every dashboard widget by Justus, including (1) Total Cases narrative and timeframe bug, (2) High Priority vs SLA conflation, (3) Open Cases bug for investigators, (4) Resolved-This-Month uses rolling 30d instead of MTD, (5)/(6) Recent/Active vocabulary and 30d drop, (7) `CLOSED_STATUSES` missing `STATUS_99_ABANDONED`, (8) `isInvestigator` OR clause over-scopes.

**What the PR does.** Nearly every named point lands:

- **Total Cases** — subtitle rewritten to "Total count of cases in system"; the dashboard now sends `dateRange=all` (`dashboardService.ts`), and [getDateRange.ts](repos/case-management-system/backend/src/modules/report/helpers/getDateRange.ts) maps `'all'` to `startDate = new Date(0)`. Point 1 addressed.
- **High Priority Cases** — card deleted; replaced with **Available** and **Overdue** cards (real `SlaState.BREACHED`). Point 2 addressed.
- **Open Cases** — new backend field `openAssignedCases` excludes both closed AND `STATUS_00_DRAFT` (see report.service.ts around line 604). Point 3 addressed.
- **Resolved This Month** — new field `resolvedThisMonth` derived from `updated_at >= monthStart` and `status ∈ CLOSED_STATUSES`, independent of the incoming `dateRange` (MTD by close date). Point 4 addressed.
- **Recent / Active panels** — removed; replaced with "Open Cases by Priority" and "Open Cases by Status". Points 5 and 6 addressed.
- **`CLOSED_STATUSES`** — `STATUS_99_ABANDONED` added at [report.service.ts line 171](repos/case-management-system/backend/src/modules/report/report.service.ts) and to `case-query.service.ts`'s `closedOnly` set and `getAllCases`'s excluded-closed set; also to frontend `CaseFilters.tsx` and `options.ts`. Point 7 addressed **but see Concerns**.
- **`isInvestigator` OR** — `applyInvestigatorScope` rewritten in `report.service.ts`: `owned` OR `task-assigned` OR `status ∈ {DRAFT, READY_FOR_ASSIGNMENT}` OR `PENDING_CASE_CREATION_APPROVAL AND case_owner_user_id IS NULL`. The old creator-based branches (ABANDONED, READY_FOR_ASSIGNMENT via `case_creator_user_id`) are gone. Point 8 addressed.

**What's left.**
- Justus's semantic critique of "Active" vocabulary is honored by renaming panels, but the top-level card is still labelled "Open Cases" (subtitle "Currently being investigated"). Small.
- `openAssignedCases` excludes DRAFT for the *count*, but `priorityScopedCases` also silently drops drafts, while the "Open Cases by Status" panel *does* include DRAFT — small internal inconsistency.

**Concerns.**
- **Adding `STATUS_99_ABANDONED` to `CLOSED_STATUSES` in `report.service.ts`** is the biggest data-shape decision in this PR. It also means:
  - `resolvedThisMonth` counts abandonments as resolutions.
  - `computeOutcomes` maps `refuted/confirmed/inconclusive/pending` but not abandoned — so `closedCases > sum(outcomes)`, breaking outcome-reconciliation.
  - This **contradicts #219's explicit recommendation** to treat abandoned as if the case never existed.
- **Investigator now sees every DRAFT and every READY_FOR_ASSIGNMENT in the tenant, without ownership guard.** Previously drafts required `case_owner_user_id ∈ (null, me)`. Sobia's inline comment on line 470 says "STATUS_02 comes under Open Cases in Cases Dashboard. So it has to be that way." — suggesting intentional, but worth confirming with product because it *expands* investigator visibility.
- **CodeRabbit major (unresolved):** `checkUserCaseAccess` in `case-query.service.ts` L873 was NOT updated to include `STATUS_00_DRAFT` while `getAllCases` at L567/L585 was updated to admit DRAFT. Investigators can see a DRAFT in the list but be denied access when routing via `checkUserCaseAccess`. This is the same OR-clause divergence #209 warned about, reintroduced.

---

### Issue #210 — visualizations tab on case details

**What the issue asks.** Original: Viz tab lives only on task; add it to case details. Follow-up from Sandy on 2026-07-10: viz should be on **both** the case and the task view.

**What the PR does.** [ViewCaseModal.tsx](repos/case-management-system/frontend/src/features/cases/components/ViewCaseModal.tsx) already carries a `VisualizationsTab` (pre-existing; the PR only removes the unused `onSwitchToCaseDetails` prop). [TasksDetailsModal.tsx](repos/case-management-system/frontend/src/features/cases/components/TasksDetailsModal.tsx) now conditionally renders a Visualizations tab when `row.transaction.FIToFIPmtSts` is present; a matching test in [TasksDetailsModal.test.tsx](repos/case-management-system/frontend/src/features/cases/components/__tests__/TasksDetailsModal.test.tsx) verifies the tab hides when `transaction: undefined`.

**What's left.** Manual verification in the Tazama tenant across statuses (before task creation, during, after close).

**Concerns.** The transaction-ID extraction chain (`FIToFIPmtSts.TxInfAndSts.OrgnlEndToEndId || EndToEndId || transaction_id || transactionId`) is brittle but defensively guarded.

---

### Issue #217 — Case Ageing Report analysis

**What the issue asks.** A per-widget rewrite spec for Case Ageing: exclude `FRAUD_AND_AML` container from the dataset, exclude `STATUS_99_ABANDONED` from the ageing scope entirely (do NOT treat as closed), drop the "claimable pool" (unassigned + READY_FOR_ASSIGNMENT) from the investigator OR clause, extract the shared OR clause into one helper (currently copy-pasted at report.service.ts L966, L1091, L1167), and multiple widget-level semantic fixes (`avgCaseAge` open-only, return `null` on empty population, etc.).

**What the PR does.** Effectively nothing on this surface. `getCaseAgeing` and its three sub-queries are untouched. `applyInvestigatorScope` is extracted for the *Case Status* service but is NOT used by `getCaseAgeing`. The one tangential change — `NON_CONTAINER_CASE_FILTER` — is *widened* to admit FRAUD_AND_AML containers in DRAFT/PENDING_APPROVAL, which slightly moves in the opposite direction to #217's ask.

**What's left.** The entire ageing rewrite.

**Concerns.** Widening `NON_CONTAINER_CASE_FILTER` to admit in-flight containers means Total Cases and other Case-Status widgets that use this filter will count in-flight containers as real cases. Small blast radius today, but points to the wrong direction ahead of the #217 rewrite.

---

### Issue #219 — Case Status Report analysis

**What the issue asks.** Per-widget spec: exclude `STATUS_84_COMPLETED` from `CLOSED_STATUSES` (or retire it), exclude `FRAUD_AND_AML` container, exclude `STATUS_99_ABANDONED` from the dataset entirely, plus per-widget fixes.

**What the PR does.**
- `STATUS_84_COMPLETED` stays excluded from `CLOSED_STATUSES` — aligned with #219.
- `STATUS_99_ABANDONED` is **added to `CLOSED_STATUSES`** — opposite direction from #219.
- `Case Types` bar now open-cases-only (`typeCounts` uses `openCasesWhere`) — aligned.
- `STATUS_DISTRIBUTION_MAP` rewritten to one bucket per `CaseStatus` — aligned.
- `computeCaseTypes` filters null `case_type` entries — aligned.
- `computeOutcomes` now splits `refuted` from `resolved` (its own bucket) — aligned.
- `avgResolutionDays` returns `null` when there are no closed cases (see [report.service.ts line 178](repos/case-management-system/backend/src/modules/report/report.service.ts)) — aligned with #219's empty-data recommendation.
- Frontend `CaseStatusReport.tsx` reorders sections into "Live Workload" / "Closed Throughput" groups, moves status details above the closed charts, derives status labels dynamically — all aligned with the issue's per-widget layout critique.

**What's left.**
- FRAUD_AND_AML container exclusion at the query level — not done (only the `case_type` filter is touched).
- `STATUS_99_ABANDONED` treated correctly (i.e., excluded, not closed).
- `STATUS_84` retirement — explicitly out of scope.
- Outcome-reconciliation with closed-count (broken because STATUS_99 is now closed but has no outcome slice).
- Investigator column in the details table still shows raw IDs.

**Concerns.**
- `computeStatusDetails` derives rows from `Object.values(CaseStatus).filter(...)` and **excludes `STATUS_03_RETURNED`** with no justifying comment. Supervisors lose visibility of returned cases in the details table.
- `formatStatusName` rewritten from `"10 ASSIGNED"` (upper snake stripped) to `"10 Assigned"` (Title Case). This changes every rendered status label on the Case Status Details table and its exports — downstream matching/paste-back will break silently.
- `currentTrendPeriod` hardcoded to `'0'` (was `'+0%'`) — placeholder.

[↑ Back to top](#pr-review-cms-242--fix-case-dashboard-count)

---

## What Changed (Grouped)

The PR is too broad to enumerate line-by-line; the per-issue section above cites the specific hotspots. Grouped summary:

**Backend — Report service**
- `CLOSED_STATUSES` gains `STATUS_99_ABANDONED` (report.service.ts L171).
- `STATUS_DISTRIBUTION_MAP` becomes 1:1 (one bucket per `CaseStatus`) — API shape change consumed by `caseStatus[]` output.
- New `Prisma.CaseWhereInput` slices in `computeCaseStatus`: `openCasesWhere`, `openAssignedCasesWhere`, `closedWindowFilters`, `availableCasesWhere`, `overdueCasesWhere` (SLA `BREACHED`), `resolvedThisMonthWhere` (`updated_at >= monthStart`), `openStatusCountsWhere`, `openPriorityCountsWhere`.
- `applyInvestigatorScope` extracted (report.service.ts L132-176) — but `getCaseAgeing` still inlines its own OR clause (three copies) unchanged.
- `computeOutcomes` splits `refuted` from `resolved`.
- `avgResolutionDays` returns `null` on empty population.
- `formatStatusName` returns Title Case ("10 Assigned").
- `getFilters` joins `cms_usernames` scoped by `tenantId` for the investigator dropdown.

**Backend — Case Query service**
- `closedOnly` and excluded-closed lists gain `STATUS_99_ABANDONED`.
- Investigator DRAFT branch: `status: STATUS_02_READY_FOR_ASSIGNMENT` → `status: { in: [STATUS_00_DRAFT, STATUS_02_READY_FOR_ASSIGNMENT] }` (case-query.service.ts L564-566, L582-584).
- **`checkUserCaseAccess` NOT updated** — divergence with `getAllCases` (CodeRabbit major, open).

**Backend — helpers/types**
- `getDateRange` supports `'all'` (epoch start).
- `ReportsQueryDto` / controller enum add `'all'`; Swagger `@ApiQuery` enum was not updated (CodeRabbit minor).

**Frontend — dashboard**
- New Available / Overdue / Open / Resolved-This-Month cards; `StatsCard[s]` supports color-key + trend + subtitle.
- New "Open Cases by Priority" and "Open Cases by Status" panels with max-based bar scaling, expand/collapse "+N more".
- `AlertSummaryItem` / `CaseSummaryItem` gain link support.
- `useCaseDashboard`: URL round-trip for status/priority.
- `dashboardService` sends `dateRange=all`.

**Frontend — reports**
- Section reorder (Live Workload / Closed Throughput headings).
- Dynamic status labels; horizontal bar option; MultiBarChart integer-only ticks; WorkloadBarChart tweaks; FiltersPanel investigator dropdown consumes new name-bearing options.

**Frontend — task modal (issue #210)**
- Conditional VisualizationsTab in `TasksDetailsModal` when `FIToFIPmtSts` present.

**Notebook**
- Bucketing rewrite; cumulative from `cumsum(bucketed)`; explicit x-axis per granularity.

[↑ Back to top](#pr-review-cms-242--fix-case-dashboard-count)

---

## Code Quality Analysis

### Strengths

- **Real fix to the SLA/priority conflation.** Splitting the old "High Priority" card into `availableCases` and `overdueCases` (with the latter computed from real `SlaState.BREACHED` via `computeCaseSlaState`) directly answers Justus's central complaint in #209.
- **`resolvedThisMonth` is MTD, not rolling 30 days.** `updated_at >= monthStart` with `status ∈ CLOSED_STATUSES`, independent of the request `dateRange`, matches #209's ask.
- **`STATUS_DISTRIBUTION_MAP` is now 1:1.** Statuses are no longer silently collapsed into `pendingApproval` / `closed`, which was one of #219's per-widget fixes.
- **URL round-trip in `useCaseDashboard`.** Filter → URL sync uses `replace: true` so history is not polluted; landing directly on `/cases?status=…` works.
- **Notebook rewrite is a real logic change, not an output strip.** Bucketing + reindex + explicit ranges are the right primitives.
- **`applyInvestigatorScope` extracted** for the Case Status service — reduces one instance of the shared OR clause. (Regrettably, `getCaseAgeing` still inlines three copies.)
- **`avgResolutionTime` returns `null` on empty populations** — the correct type contract for "no data" vs "zero-day mean".

### Issues and Observations

#### Issue 1 — `STATUS_99_ABANDONED` added to `CLOSED_STATUSES` contradicts #219 and breaks outcome-reconciliation

**Severity: Major (Data Integrity)**

Backend [report.service.ts line 171](repos/case-management-system/backend/src/modules/report/report.service.ts) adds `CaseStatus.STATUS_99_ABANDONED` to the class-level `CLOSED_STATUSES`. Every widget on the Case Status Report consumes this constant:

```typescript
// report.service.ts (line 171)
private static readonly CLOSED_STATUSES: CaseStatus[] = [
  CaseStatus.STATUS_71_AUTOCLOSED_CONFIRMED,
  CaseStatus.STATUS_72_AUTOCLOSED_REFUTED,
  CaseStatus.STATUS_81_CLOSED_REFUTED,
  CaseStatus.STATUS_82_CLOSED_CONFIRMED,
  CaseStatus.STATUS_83_CLOSED_INCONCLUSIVE,
  CaseStatus.STATUS_99_ABANDONED, // ← added
];
```

Consequences:

1. `resolvedThisMonth` counts abandoned cases as resolutions this month (`resolvedThisMonthWhere` matches on `status ∈ CLOSED_STATUSES` AND `updated_at >= monthStart`). Abandonment is not resolution.
2. `computeOutcomes` maps four outcomes (`refuted / confirmed / inconclusive / pending`) but does not have a bucket for abandoned, so `closedCases > sum(outcomes)`. The Outcomes pie will no longer reconcile against the Closed Cases count on the same page.
3. Issue #219 explicitly recommends treating `STATUS_99` as if the case never existed ("exclude from the dataset entirely"). Folding it into `CLOSED_STATUSES` moves in the opposite direction.

**Suggested fix.** Do NOT add `STATUS_99` to `CLOSED_STATUSES`. Instead:
- Where the dashboard needs "not-active" filtering (e.g. `openAssignedCases`), keep the current combined list as an ad-hoc `notActiveStatuses = [...CLOSED_STATUSES, STATUS_99_ABANDONED]` local variable.
- For anywhere that means "resolved" — Case Status Report `Closed Cases`, `resolvedThisMonth`, outcome pie, avg-resolution — stick with the current five closed statuses only.
- If issue #219's stronger recommendation (exclude entirely) is acceptable to product, add an early `NOT { status: 'STATUS_99_ABANDONED' }` at the Case-Status query builder.

#### Issue 2 — `checkUserCaseAccess` DRAFT divergence (CodeRabbit major, unresolved)

**Severity: Major (Bug / Authorization)**

`case-query.service.ts` L567 and L585 admit `STATUS_00_DRAFT` into the investigator OR clause for `getAllCases`:

```typescript
{ status: { in: [CaseStatus.STATUS_00_DRAFT, CaseStatus.STATUS_02_READY_FOR_ASSIGNMENT] } }
```

But `checkUserCaseAccess` (around L838-887, unchanged in this PR) still guards with `STATUS_02_READY_FOR_ASSIGNMENT` only. So an investigator can see a DRAFT in the Cases list and follow an alert-driven deep link, then be denied by the access check. CodeRabbit flagged this on 2026-07-13 and it was not resolved in later commits.

**Suggested fix.** Update the `checkUserCaseAccess` `whereCondition` to include `STATUS_00_DRAFT` in the same set used by `getAllCases`.

#### Issue 3 — `openAssignedCases` missing from `reportsService.ts` error fallback and test fixtures (CodeRabbit critical)

**Severity: Major (Bug / Test Coverage)**

`frontend/src/features/reports/services/reportsService.ts` L61-67 adds `openAssignedCases: number` as a required field on `CaseStatusStats` but does not include it in the error-fallback stats object, nor in two test fixtures. This will fail the TS compile and produce runtime `undefined` on error paths. Author replied "fixed" on other CodeRabbit items but did not explicitly respond to this one.

**Suggested fix.** Add `openAssignedCases: 0` (or an appropriate `safeFallback`) to the error path and both fixtures.

#### Issue 4 — `getCaseAgeingData` `safeFallback(..., 0)` erases the nullable-null contract (CodeRabbit major)

**Severity: Major (Data Integrity)**

Backend now returns `avgCaseAge: null` / `avgResolutionTime: null` for empty populations (an intentional #219 fix). Frontend `reportsService.ts` L235-238 coerces those to `0` via `safeFallback(x, 0)`. The card cannot then render "N/A" for "no data" — the whole point of the backend change is lost.

**Suggested fix.** Change the frontend `safeFallback` for these two fields to preserve `null`, and let `StatsCard` render `null` as `N/A` (or `–`).

#### Issue 5 — Investigator visibility expanded silently

**Severity: Minor (Data Integrity / Product Alignment)**

`applyInvestigatorScope` now unions all DRAFT and all READY_FOR_ASSIGNMENT (any owner) into the investigator's view (report.service.ts L163-170). Previously drafts required `case_owner_user_id ∈ (null, me)`. This is a real expansion of investigator visibility. Sobia's comment on line 470 asserts this is intentional ("STATUS_02 comes under Open Cases in Cases Dashboard. So it has to be that way"), but no product sign-off is captured in the PR.

**Suggested fix.** Confirm with product and add a JSDoc comment on `applyInvestigatorScope` explaining the intent.

#### Issue 6 — `computeStatusDetails` silently hides `STATUS_03_RETURNED`

**Severity: Minor (Data Integrity)**

```typescript
// report.service.ts (around line 353)
.filter((status) => !ReportsService.CLOSED_STATUSES.includes(status)
                    && status !== CaseStatus.STATUS_03_RETURNED)
```

Returned cases now do not appear in the Case Status Details table. Neither #219 nor #217 asks for this exclusion. Supervisors lose visibility of returned cases in that report.

**Suggested fix.** Either drop the `STATUS_03_RETURNED` exclusion, or document the reason (with an inline comment linking a decision).

#### Issue 7 — `formatStatusName` output changes silently

**Severity: Minor (Breaking Change Risk)**

`formatStatusName` (report.service.ts around L1290) is rewritten from producing `"10 ASSIGNED"` (upper-snake stripped) to `"10 Assigned"` (Title Case). This changes every status label on the Case Status Details table and its CSV/XLSX/PDF exports. Downstream consumers relying on paste-back or string-matching will break silently.

**Suggested fix.** If the visual change is intentional, note it in the PR description; ensure export headers / downstream integration docs are updated.

#### Issue 8 — `NON_CONTAINER_CASE_FILTER` widened to admit in-flight FRAUD_AND_AML containers

**Severity: Minor (Data Integrity)**

Diff L74-88 relaxes the "non-container" filter to admit `FRAUD_AND_AML` cases whose status is `STATUS_00_DRAFT` or `STATUS_01_PENDING_CASE_CREATION_APPROVAL` (rationale: container has not "split" into siblings yet). This moves in the opposite direction to #217/#219's explicit ask to remove the container from the dataset. The blast radius is limited today (few in-flight containers), but the direction is wrong.

**Suggested fix.** Prefer to leave `NON_CONTAINER_CASE_FILTER` strict and, if a specific widget needs the in-flight container, admit it locally rather than widening the shared filter.

#### Issue 9 — Swagger `@ApiQuery` enum does not include `'all'` (CodeRabbit minor)

**Severity: Minor (API Contract)**

Backend `getDateRange` now understands `'all'` and the DTO enum was updated, but the Swagger `@ApiQuery` on the report controllers still lists only the older values. External consumers reading Swagger will not know `'all'` is valid.

**Suggested fix.** Add `'all'` to the `@ApiQuery` enum.

#### Issue 10 — `isInvestigator` computation duplicated across controller methods (CodeRabbit trivial)

**Severity: Informational**

The `isInvestigator = claims include CMS_INVESTIGATOR && !CMS_SUPERVISOR && !CMS_ADMIN` computation is inlined in three controller methods. Trivial refactor.

[↑ Back to top](#pr-review-cms-242--fix-case-dashboard-count)

---

## Security Assessment

| Concern | Assessment |
|---------|-----------|
| Authorization bypass | `checkUserCaseAccess` still guards on `STATUS_02_READY_FOR_ASSIGNMENT` only, while `getAllCases` now admits DRAFT — this is *tighter than* the list query, so the risk is a false-negative (denying a legitimate deep link), not a bypass. Still needs fixing (Issue 2). No open-mode expansion. |
| Tenant scoping | New `getFilters` join is `WHERE tenant_id = ?` — correct. `applyInvestigatorScope` uses `case_owner_user_id = <me>` from JWT claims — correct. |
| SQL injection | Prisma-parameterised. No raw SQL added. |
| XSS | Dashboard tiles use `<a href>` with `encodeURIComponent(status/priority)` — safe. No `dangerouslySetInnerHTML` added. |
| CSRF / open redirect | URL sync uses `navigate(..., { replace: true })` to same-origin routes — safe. |
| Data leakage | `applyInvestigatorScope` widens investigator visibility to all DRAFTs / READY_FOR_ASSIGNMENT in the tenant regardless of ownership (Issue 5). This is likely intentional per product but should be signed off — it does expand what one investigator can see about tenant-wide work. |
| Secrets / logging | None touched. |

**No new security vulnerabilities introduced by this PR.** The `checkUserCaseAccess` divergence is a deny-side bug, not a bypass; the investigator-scope widening is a product decision, not a leak.

[↑ Back to top](#pr-review-cms-242--fix-case-dashboard-count)

---

## Test Coverage

- Backend `backend/test/report.service.spec.ts` — +142/-14. Covers `openStatusCounts` excluding STATUS_99, `openPriorityCounts`, `resolvedThisMonth` MTD boundaries, `availableCases`, `overdueCases` (SLA breached), and the new investigator scope.
- Frontend `frontend/src/features/cases/hooks/__tests__/useCaseDashboard.test.ts` — +93 (new). Covers URL parameter round-trip and filter → URL sync.
- Frontend `frontend/src/features/cases/components/__tests__/TasksDetailsModal.test.tsx` — updated to verify conditional Viz tab presence/absence.
- Frontend dashboard test updates: `StatsCard[s]`, `CaseSummaryItem`, `Dashboard.test.tsx` (+28/-4).
- Reports test updates: `useReports.test.tsx`, `reports.types.test.ts`, `dashboardService.test.ts`, `ReportStatsCards`, `ReportsTable`, `EvidenceFindingsStatsCards`.

**Gaps.**
- CodeRabbit noted `getFilters` unit tests do NOT pass the new `{ tenantId, requestingUserId }` params, so the investigator scoping on the filter dropdown is not exercised.
- No tests for the frontend error-fallback path in `reportsService.ts` (`openAssignedCases` missing there) — CodeRabbit critical.
- No test asserting `getCaseAgeingData` preserves `null` for empty populations (currently coerced to `0` by `safeFallback` — Issue 4).
- No test around `checkUserCaseAccess` DRAFT parity with `getAllCases` (Issue 2).
- Notebook has no automated verification; needs manual QA in Tazama tenant.

**PR checklist.** Author marked "Locally", "Development Environment", "Husky successfully run", and "Unit tests passing and Documentation done"; left "Not needed, changes very basic" unchecked. Coverage screenshot not attached.

[↑ Back to top](#pr-review-cms-242--fix-case-dashboard-count)

---

## CodeRabbit Activity

CodeRabbit performed two review passes.

### Pass 1 — 2026-07-13 11:01Z

**Commit reviewed:** `47be30c7` (approx.)
**Findings:** 4 actionable + 2 outside-diff + 6 nitpicks

| Finding | Severity | Status |
|---------|----------|--------|
| `checkUserCaseAccess` DRAFT divergence with `getAllCases` | Major | ❌ Not resolved (Issue 2 in this review) |
| `dashboardService.test.ts` `getDescription('Low')` string mismatch | Critical | ✅ Resolved — author replied "fixed in latest commit" |
| Duplicated closed-status list in `CaseFilters.tsx` | Trivial | ⚠️ Partial — new abandoned entry added, no dedup |
| Duplicated color helpers in `AlertSummaryItem.tsx` | Trivial | ⚠️ Partial |
| Missing show-more/less test in `Dashboard.test.tsx` | Trivial | ❌ Not addressed |
| 5-column grid responsive breakpoints | Trivial | ❌ Not addressed |
| O(n²) reduce with spread in `rawStatusCounts` / `casesByStatus` | Trivial | ❌ Not addressed |
| Missing tests for `availableCases`, `openAssignedCases`, `resolvedThisMonth` | Nitpick | ✅ Resolved — added in `report.service.spec.ts` |

### Pass 2 — 2026-07-13 14:55Z

**Commit reviewed:** `b60467d1` (approx.)
**Findings:** 4 outside-diff + 3 nitpicks

| Finding | Severity | Status |
|---------|----------|--------|
| `reportsService.ts` L61-67: `openAssignedCases` missing from error fallback + 2 fixtures | Critical | ❌ Not resolved (Issue 3) |
| `reportsService.ts` L235-238: `safeFallback(x, 0)` erases nullable contract | Major | ❌ Not resolved (Issue 4) |
| Swagger `@ApiQuery` enum missing `'all'` | Minor | ❌ Not resolved (Issue 9) |
| `getFilters` tests missing `{ tenantId, requestingUserId }` | Minor | ❌ Not resolved |
| `isInvestigator` duplicated across controller methods | Trivial | ❌ Not addressed (Issue 10) |
| `dateRange=all` performance risk on large tenants | Trivial | ❌ Not addressed |
| `MultiBarChart` YAxis missing integer `tickFormatter` | Trivial | ❌ Not addressed |

Reconciliation: The two CodeRabbit criticals and both majors that remain (`openAssignedCases` fallback, `getCaseAgeingData` safeFallback, `checkUserCaseAccess` DRAFT) are corroborated in my Issues 2/3/4 above.

[↑ Back to top](#pr-review-cms-242--fix-case-dashboard-count)

---

## Summary and Verdict

**Verdict: Changes Requested**

This PR does the heavy lifting on the two most important asks — #209 (dashboard rework) and #158 (notebook) — and pushes #152, #153, #210 substantially forward. The stat-card definitions now match the Cases list's investigator-scoping and status rules, the notebook rewrite is a real fix (not an output strip), and the URL round-trip in `useCaseDashboard` cleanly delivers the "click count → filtered list" behavior. Test coverage of the new backend fields is solid.

The blockers are: (a) folding `STATUS_99_ABANDONED` into `CLOSED_STATUSES` breaks outcome-reconciliation on the Case Status Report and contradicts #219; (b) `checkUserCaseAccess` was not updated in step with `getAllCases`, so investigators can be denied on legitimate deep links; and (c) the frontend `reportsService.ts` error path and two test fixtures are missing the new required `openAssignedCases` field, which is a TS compile error waiting to happen. Two other issues (#201, #217) are entirely untouched despite being in the closing list — either drop them from the PR's closing set or split them out.

### Blocking

1. **`STATUS_99_ABANDONED` added to `CLOSED_STATUSES` breaks outcome-reconciliation and contradicts #219** — remove it from `CLOSED_STATUSES`; use a local `notActiveStatuses` where "not active" is needed.
2. **`checkUserCaseAccess` DRAFT divergence with `getAllCases`** — update the guard to include `STATUS_00_DRAFT` in the investigator OR clause.
3. **`openAssignedCases` missing from `reportsService.ts` error fallback + 2 test fixtures** — TS compile / runtime `undefined`.
4. **`getCaseAgeingData` `safeFallback(x, 0)` erases the null contract** — preserve null through to the UI so "no data" can render as `N/A`.
5. **Issues #201 and #217 in the closing list are effectively untouched** — either drop them from the closing set or scope them into this PR.

### Non-blocking but recommended

6. **`computeStatusDetails` silently hides `STATUS_03_RETURNED`** — document or drop the exclusion.
7. **`formatStatusName` output shape changed** — call out in the PR description; verify export headers and downstream consumers.
8. **Investigator visibility widened to all DRAFT / READY_FOR_ASSIGNMENT** — capture product sign-off in the PR description; add JSDoc.
9. **Swagger `@ApiQuery` enum missing `'all'`** — add for API-doc consumers.
10. **`NON_CONTAINER_CASE_FILTER` widened for in-flight FRAUD_AND_AML containers** — prefer a local admission over widening the shared filter.
11. **Deduplicate closed-status list and color helpers** — CodeRabbit trivials.
12. **`isInvestigator` claim computation duplicated in 3 controller methods** — extract.

[↑ Back to top](#pr-review-cms-242--fix-case-dashboard-count)

---

## GitHub Review Comment

`````markdown
**Changes Requested**

Solid progress on the dashboard rework (#209) and the transaction-viz notebook (#158); URL round-trip for dashboard tiles (#152/#153) and the Task-modal Visualizations tab (#210) all land cleanly. Blocking items are: a data-integrity change that contradicts #219, an authorization divergence CodeRabbit flagged in pass 1, a missing frontend field on the error path, and two closing-list issues that aren't actually touched.

---

### Blocking

**1. `STATUS_99_ABANDONED` added to `CLOSED_STATUSES` breaks outcome-reconciliation and contradicts #219**

`backend/src/modules/report/report.service.ts` line 171:

```typescript
private static readonly CLOSED_STATUSES: CaseStatus[] = [
  CaseStatus.STATUS_71_AUTOCLOSED_CONFIRMED,
  CaseStatus.STATUS_72_AUTOCLOSED_REFUTED,
  CaseStatus.STATUS_81_CLOSED_REFUTED,
  CaseStatus.STATUS_82_CLOSED_CONFIRMED,
  CaseStatus.STATUS_83_CLOSED_INCONCLUSIVE,
  CaseStatus.STATUS_99_ABANDONED, // ← remove
];
```

Impact:
- `resolvedThisMonth` now counts abandoned cases as resolutions.
- `computeOutcomes` has no bucket for abandoned, so `closedCases > sum(outcomes)` — the Outcomes pie no longer reconciles against Closed Cases on the same page.
- Issue #219's explicit recommendation is to treat abandoned as if the case never existed.

Please remove `STATUS_99_ABANDONED` from `CLOSED_STATUSES`. Where the dashboard truly needs a "not-active" filter (e.g. `openAssignedCases`), define a local `const notActiveStatuses = [...CLOSED_STATUSES, STATUS_99_ABANDONED]` and use it there only. Everywhere that means "resolved" (Case Status Report `Closed Cases`, `resolvedThisMonth`, outcome pie, avg-resolution) should stay on the five real closed statuses.

**2. `checkUserCaseAccess` DRAFT divergence with `getAllCases` (CodeRabbit major, pass 1)**

`backend/src/modules/case/services/case-query.service.ts` L567/L585 updated `getAllCases` to admit `STATUS_00_DRAFT`:

```typescript
{ status: { in: [CaseStatus.STATUS_00_DRAFT, CaseStatus.STATUS_02_READY_FOR_ASSIGNMENT] } }
```

But `checkUserCaseAccess` (around L838-887) still guards on `STATUS_02_READY_FOR_ASSIGNMENT` only. An investigator will see a DRAFT in the Cases list but be denied when routing through the access check (e.g. from an alert-driven deep link). Please update the `checkUserCaseAccess` `whereCondition` with the same set.

**3. `openAssignedCases` missing from `frontend/src/features/reports/services/reportsService.ts` error fallback and two test fixtures (CodeRabbit critical, pass 2)**

The new required field on `CaseStatusStats` (L61-67) is not present in the error-fallback stats object or in two fixtures — TS compile fails, and if the compile is permissive, runtime hits `undefined`. Please add `openAssignedCases: 0` (or the appropriate `safeFallback`) in all three spots.

**4. `getCaseAgeingData` `safeFallback(x, 0)` erases the nullable-null contract (CodeRabbit major, pass 2)**

Backend now returns `avgCaseAge: null` / `avgResolutionTime: null` for empty populations (per #219's recommendation, this is deliberate). The frontend in `reportsService.ts` L235-238 coerces both to `0`, which makes "no data" indistinguishable from a real zero-day mean and prevents the card from rendering `N/A`. Preserve `null` through to the UI.

**5. Closing-list mismatch: #201 and #217 are not actually addressed**

- **#201 (ageing exports show User ID in Investigator column)** — no changes to `getCaseAgeing`, `CaseAgeingReport.tsx`, or the ageing export path in `exportUtils.ts` (the only diff there is a header rename on the Case Status report). Sandy's 2026-06-30 comment already flagged this as "not fixed".
- **#217 (Case Ageing Report requirements)** — `getCaseAgeing` and its three copy-pasted OR-clause blocks (report.service.ts L966, L1091, L1167) are untouched; no FRAUD_AND_AML container exclusion, no STATUS_99 handling, no shared-helper extraction, no widget-level fixes.

Please either land the fixes for these two in this PR or remove them from the closing set so they don't get auto-closed on merge.

---

### Non-blocking (please address in this PR if possible)

**6. `computeStatusDetails` silently hides `STATUS_03_RETURNED`**

`report.service.ts` around L353:

```typescript
.filter((status) => !ReportsService.CLOSED_STATUSES.includes(status)
                    && status !== CaseStatus.STATUS_03_RETURNED)
```

Neither #219 nor #217 asks for this exclusion. Supervisors lose visibility of returned cases in that report. Please drop the exclusion or add a comment linking the decision.

**7. `formatStatusName` output shape changed silently**

Rewritten from `"10 ASSIGNED"` to `"10 Assigned"` (Title Case). This changes every status label on the Case Status Details table and its exports. Please call this out in the PR description and verify any downstream matching / paste-back consumers.

**8. Investigator visibility widened without an ownership guard**

`applyInvestigatorScope` in `report.service.ts` L163-170 now unions all DRAFT and all READY_FOR_ASSIGNMENT (any owner) into the investigator's view. Previously drafts required `case_owner_user_id ∈ (null, me)`. Please capture product sign-off in the PR description and add a JSDoc explaining the intent so future readers don't reintroduce the old guard.

**9. Swagger `@ApiQuery` enum on the report controllers does not include `'all'`**

Backend supports it, DTO enum has it, but the OpenAPI docs will lie to consumers. One-line fix.

**10. `NON_CONTAINER_CASE_FILTER` widened to admit DRAFT / PENDING_APPROVAL FRAUD_AND_AML containers**

Moves in the opposite direction from #217/#219. If a specific widget genuinely needs in-flight containers, admit them locally rather than widening the shared filter.

**11. Deduplicate the closed-status list in `CaseFilters.tsx` and color helpers in `AlertSummaryItem.tsx`**

Both are CodeRabbit trivials from pass 1.

**12. `isInvestigator` claim computation duplicated across 3 controller methods**

Extract to a helper (CodeRabbit nitpick, pass 2).
`````

[↑ Back to top](#pr-review-cms-242--fix-case-dashboard-count)
