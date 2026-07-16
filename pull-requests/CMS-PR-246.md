# PR Review: CMS #246 — feat: redesign Case Ageing report, wire report filters, and add case assignee visibility

**Repo:** tazama-lf/case-management-system
**Branch:** `paysys/dashboard-fixes` → `dev`
**Author:** sobia-rizwan1567 (Sobia Rizwan)
**Date Reviewed:** 2026-07-15
**Label:** enhancement
**Size:** +1927 / -837 lines across 46 files
**Commits:** 5 (7ad4b13, 459d5a6, ff291ac, df9b6e7, e740227)
**State:** OPEN
**HEAD SHA verified:** `e7402270a7073ef2a0c250616d429db5a7fb2d02`
**Existing approvals:** none — CodeRabbit COMMENTED (one issue raised then resolved)

## Table of Contents

- [Overview](#overview)
- [What Changed (Detailed)](#what-changed-detailed)
  - [1. backend/report.controller.ts — new filter query params + expanded Swagger schema](#1-backendreportcontrollerts--new-filter-query-params--expanded-swagger-schema)
  - [2. backend/report.service.ts — `EXCLUDE_ABANDONED_FILTER`, shared filter helpers, Case Ageing rewrite](#2-backendreportservicets--exclude_abandoned_filter-shared-filter-helpers-case-ageing-rewrite)
  - [3. backend/report.types.ts — `resolutionTrend` reshape (median/p25/p75/n)](#3-backendreporttypests--resolutiontrend-reshape-medianp25p75n)
  - [4. backend/report.service.spec.ts — every-mock-call assertions + new coverage](#4-backendreportservicespects--every-mock-call-assertions--new-coverage)
  - [5. frontend Cases dashboard — Assignee column and filter](#5-frontend-cases-dashboard--assignee-column-and-filter)
  - [6. frontend/reports — Case Ageing UI split, section headers, new Resolution Trend chart](#6-frontendreports--case-ageing-ui-split-section-headers-new-resolution-trend-chart)
  - [7. frontend/reports — filter wiring end-to-end, All Time option](#7-frontendreports--filter-wiring-end-to-end-all-time-option)
  - [8. frontend/reports — helpers: `getClientDateRange`, `mapEvidenceToTasks`](#8-frontendreports--helpers-getclientdaterange-mapevidencetotasks)
- [Code Quality Analysis](#code-quality-analysis)
  - [Strengths](#strengths)
  - [Issues and Observations](#issues-and-observations)
- [Authorization Architecture Analysis](#authorization-architecture-analysis)
- [Security Assessment](#security-assessment)
- [Test Coverage](#test-coverage)
- [CodeRabbit Activity](#coderabbit-activity)
- [Summary and Verdict](#summary-and-verdict)
- [GitHub Review Comment](#github-review-comment)
- [Follow-up Review (2026-07-16)](#follow-up-review-2026-07-16)
  - [Resolution Status](#resolution-status)
  - [New Issues Found in Updated Commits](#new-issues-found-in-updated-commits)
  - [Updated Verdict](#updated-verdict)
  - [GitHub Review Comment (Follow-up)](#github-review-comment-follow-up)

---

## Overview

This PR does four related things:

1. **Redesigns the Case Ageing report** in [backend/src/modules/report/report.service.ts](repos/case-management-system/backend/src/modules/report/report.service.ts) into two lifecycle-scoped datasets — a live "Open Backlog" snapshot (ignores `dateRange`, feeds `avgCaseAge`/tier cards/status bar/distribution donut/details table) and a windowed "Closed Throughput" set (feeds `avgResolutionTime`, `caseTypeResolution`, `resolutionTrend`). The trend switches from a per-day scattered series to a fixed 6-calendar-month axis with median + P25-P75 band + sample-size (n) bars. Percentages are reconciled with the largest-remainder method so they sum to exactly 100.
2. **Wires the Case Type / Priority / Investigator filters end-to-end** on the Case Ageing, Investigator Workload, and Evidence Findings tabs (previously visible in the UI but silently ignored — only Case Status honoured them). Also fixes the Evidence Findings date-range filter, which was accepted but not applied.
3. **Adds a first-class Assignee column and filter** on the Cases Dashboard. The backend `ownerId` filter already existed but was not exposed in the UI; the assignee id is now surfaced raw and resolved to a display name client-side (same pattern as the Reports feature).
4. **Excludes abandoned cases everywhere** via a new `EXCLUDE_ABANDONED_FILTER` that composes with `withNonContainerCaseFilter`, so every case-scoped Prisma query in the reports/dashboard pipeline drops STATUS_99_ABANDONED. Also adds an "All Time" date-range option and makes Case Status reports default to it.

Targets `dev` — correct for this repo's flow. Every CI check is green (CodeQL, DCO, dependency-review, encoding, hadolint, node-ci build/style/tests, GPG, njsscan, PR title conventional-commit). Merge state is BLOCKED only because no human reviewer has approved yet.

| File                                                                                                                       | Nature of Change                                                                                                                                                                                                                                                 |
| -------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `backend/src/modules/report/report.controller.ts`                                                                        | New`caseType`/`priority`/`investigator` query params on Case Ageing + Investigator Workload endpoints; expanded Swagger schema documenting open/closed split.                                                                                              |
| `backend/src/modules/report/report.service.ts`                                                                           | Adds`EXCLUDE_ABANDONED_FILTER`; `buildCommonCaseFilters` + `applyInvestigatorScope` helpers; rewrites `getCaseAgeing` into open-backlog vs closed-throughput datasets; adds `reconcilePercentages`, `percentile`, `resolutionDays` static helpers. |
| `backend/src/modules/report/types/report.types.ts`                                                                       | `resolutionTrend` reshaped: `avgDays` → `n`, `median`, `p25`, `p75`.                                                                                                                                                                                |
| `backend/test/report.service.spec.ts`                                                                                    | `getAllCaseWhereClauses()` helper asserts every mock call for a filter; new `getCaseAgeing` coverage (tier boundaries, seeded status axis, reconciled percentages, 6-bucket trend, per-filter query shape).                                                  |
| `frontend/src/features/cases/components/CaseDashboardContainer.tsx` + `CaseDashboardContent.tsx` + `CaseFilters.tsx` | Assignee filter dropdown, populated from`useInvestigatorSupervisorList`.                                                                                                                                                                                       |
| `frontend/src/features/cases/components/CasesTable.tsx` + `casesTable.utils.ts`                                        | Adds Assignee column;`assignee` now holds the raw `case_owner_user_id`, resolved to a display name via `getAssigneeFullName`.                                                                                                                              |
| `frontend/src/features/cases/hooks/useCaseDashboard.ts` + `services/caseService.ts`                                    | `assigneeFilter` → `ownerId` query param.                                                                                                                                                                                                                   |
| `frontend/src/features/reports/components/CaseAgeingBarChart.tsx`                                                        | Horizontal 100%-stacked layout; count folded into the single Y-axis tick label instead of a second category axis (fixes tooltip/axis desync bug — regression-covered).                                                                                          |
| `frontend/src/features/reports/components/CaseAgeingPieChart.tsx`                                                        | Donut styling; uses backend-reconciled percentages.                                                                                                                                                                                                              |
| `frontend/src/features/reports/components/CaseAgeingStatsCards.tsx`                                                      | Splits into three open-backlog cards (avgCaseAge + non-overlapping 15-30 / 30+ tiers); Avg Resolution Time moves to the Closed Throughput section.                                                                                                               |
| `frontend/src/features/reports/components/CaseAgeingTable.tsx`                                                           | Renders raw investigator id + resolved name as separate columns; ISO date localised client-side.                                                                                                                                                                 |
| `frontend/src/features/reports/components/ResolutionTimeTrendChart.tsx`                                                  | Rewritten as a`ComposedChart` — median line, P25-P75 area band, n sample-size bars, custom tooltip.                                                                                                                                                           |
| `frontend/src/features/reports/components/ReportSectionHeader.tsx`                                                       | New shared "Open Backlog" / "Closed Throughput" section-header component.                                                                                                                                                                                        |
| `frontend/src/features/reports/components/ReportFilters.tsx`                                                             | Adds`all` to the `dateRange` union.                                                                                                                                                                                                                          |
| `frontend/src/features/reports/hooks/useReports.ts`                                                                      | `useCaseAgeing`/`useInvestigatorWorkload`/`useEvidenceFindings` accept a `filters` param; folded into the query key.                                                                                                                                     |
| `frontend/src/features/reports/pages/CaseAgeingReport.tsx`                                                               | Reorganised into Open Backlog / Closed Throughput sections; forwards`filters`.                                                                                                                                                                                 |
| `frontend/src/features/reports/pages/CaseStatusReport.tsx`                                                               | `dateRange` default flipped from `last30` to `all`; passes `filters` to all four report pages.                                                                                                                                                           |
| `frontend/src/features/reports/pages/EvidenceFindingsReport.tsx` + `InvestigatorWorkloadReport.tsx`                    | Forward`filters`; add `'all'` to the `dateRange` union.                                                                                                                                                                                                    |
| `frontend/src/features/reports/services/reportsService.ts`                                                               | Appends`caseType`/`priority`/`investigator` query params; applies `dateRange` + filters **client-side** in `getEvidenceFindingsData` (no dedicated backend endpoint).                                                                            |
| `frontend/src/features/reports/helpers/getClientDateRange.ts`                                                            | **New** — mirrors backend `getDateRange` for the Evidence Findings client-side filter.                                                                                                                                                                  |
| `frontend/src/features/reports/helpers/mapEvidenceToTasks.ts`                                                            | **New** — extracted from `reportsService.ts` to keep it under the max-lines lint budget.                                                                                                                                                                |
| `frontend/src/features/reports/types/reports.types.ts`                                                                   | `ResolutionTrend` reshape; `CaseAgeingDetail.investigatorId` replaces `investigator`.                                                                                                                                                                      |
| `frontend/src/shared/constants/options.ts`                                                                               | Adds`all` to `DATE_RANGE_OPTIONS` and `DATE_RANGE_LABELS`.                                                                                                                                                                                                 |
| `frontend/src/shared/utils/exportUtils.ts`                                                                               | Case Ageing export uses`investigatorId` first, with legacy field fallbacks retained.                                                                                                                                                                           |
| Various`__tests__/*` files                                                                                               | Updated for the new data shapes and behaviours.                                                                                                                                                                                                                  |

[↑ Back to top](#pr-review-cms-246--feat-redesign-case-ageing-report-wire-report-filters-and-add-case-assignee-visibility)

---

## What Changed (Detailed)

### 1. backend/report.controller.ts — new filter query params + expanded Swagger schema

`getInvestigatorWorkload` and `getCaseAgeing` gain three optional query params. Swagger documents the open/closed split for Case Ageing explicitly, and clarifies that `investigator` is a supervisor-only filter (ignored for investigator-role callers).

```diff
- async getInvestigatorWorkload(@Req() req, @Query('dateRange') dateRange?: string) {
-   const { tenantId } = req.user.token;
-   return await this.reportsService.getInvestigatorWorkload(dateRange, tenantId);
- }
+ async getInvestigatorWorkload(
+   @Req() req,
+   @Query('dateRange') dateRange?: string,
+   @Query('caseType') caseType?: string,
+   @Query('priority') priority?: string,
+   @Query('investigator') investigator?: string,
+ ): Promise<unknown> {
+   const { tenantId } = req.user.token;
+   return await this.reportsService.getInvestigatorWorkload(dateRange, { tenantId, caseType, priority, investigator });
+ }
```

For `getCaseAgeing`, the investigator filter is deliberately suppressed for investigator-role callers (they are already scoped to their own cases via `requestingUserId`; adding an explicit investigator filter on top would only narrow that further in confusing ways):

```typescript
return await this.reportsService.getCaseAgeing(dateRange, {
  caseType,
  priority,
  investigator: isInvestigator ? undefined : investigator,
  tenantId,
  requestingUserId: isInvestigator ? userId : undefined,
});
```

### 2. backend/report.service.ts — `EXCLUDE_ABANDONED_FILTER`, shared filter helpers, Case Ageing rewrite

**New static filter — abandoned exclusion composes with the non-container filter:**

```typescript
private static readonly EXCLUDE_ABANDONED_FILTER: Prisma.CaseWhereInput = {
  status: { not: CaseStatus.STATUS_99_ABANDONED },
};

private static withNonContainerCaseFilter(where: Prisma.CaseWhereInput = {}): Prisma.CaseWhereInput {
  const andFilters = where.AND ? (Array.isArray(where.AND) ? where.AND : [where.AND]) : [];
  return {
    ...where,
    AND: [...andFilters, ReportsService.NON_CONTAINER_CASE_FILTER, ReportsService.EXCLUDE_ABANDONED_FILTER],
  };
}
```

Every downstream helper (`withNonContainerTaskCaseFilter`, `buildCommonCaseFilters`) routes through this, so **every case-scoped Prisma query in the reports pipeline picks up the abandoned exclusion**. The updated `getCaseStatus` test now asserts this across every mock call, not just the first.

**New shared filter builder:**

```typescript
private buildCommonCaseFilters(filters?: {
  caseType?: string;
  priority?: string;
  investigator?: string;
  tenantId?: string;
}): Prisma.CaseWhereInput {
  const where: Record<string, any> = {};
  if (filters?.caseType) where.case_type = filters.caseType;
  if (filters?.priority) where.priority = filters.priority;
  if (filters?.investigator) where.case_owner_user_id = filters.investigator;
  if (filters?.tenantId) where.tenant_id = filters.tenantId;
  return ReportsService.withNonContainerCaseFilter(where);
}
```

**`getCaseAgeing` rewrite — the two-dataset split:**

```typescript
// --- Open backlog: live snapshot, as-of-now, ignores dateRange. ---
const openWhere = this.applyInvestigatorScope(
  { ...commonFilters, status: { notIn: ReportsService.CLOSED_STATUSES } },
  filters?.requestingUserId,
);

const openCases = await this.prisma.case.findMany({ where: openWhere, ... });
// avgCaseAge, casesOver15Days (>15 && <30), casesOver30Days (>=30), ageBuckets,
// ageingByStatus (seeded from open-eligible enum values), ageingDistribution
// (reconciled to 100%), caseDetails all derive from openCases.

// --- Closed throughput: windowed on the closed-at proxy (updated_at). ---
const { startDate, endDate } = getDateRange(dateRange);
const closedWhere = this.applyInvestigatorScope(
  { ...commonFilters, status: { in: ReportsService.CLOSED_STATUSES }, updated_at: { gte: startDate, lte: endDate } },
  filters?.requestingUserId,
);
const closedCases = await this.prisma.case.findMany({ where: closedWhere, ... });
// avgResolutionTime, caseTypeResolution (grouped in-memory)

// --- Trend: fixed 6-month axis independent of dateRange. ---
const trendMonths = Array.from({ length: 6 }, (_, i) => new Date(now.getFullYear(), now.getMonth() - (5 - i), 1));
// ...per-bucket median + p25 + p75 + n
```

**Non-overlapping tiers** — 15-30 stops short of 30, so a case counts in exactly one card (was overlapping before: `> 15` and `>= 30` both matched a 30-day case).

**Status axis seeded from the enum, not the fetched cases** — a status with zero open cases now renders as an empty row instead of vanishing. STATUS_03_RETURNED is deliberately excluded from the bar (kept in avgCaseAge / tiers / donut / details), matching `computeStatusDetails` above it. Test at spec line 668 asserts exactly 8 rows.

**Largest-remainder (Hamilton) rounding** — `reconcilePercentages` guarantees the donut sums to exactly 100:

```typescript
private static reconcilePercentages<T extends { count: number }>(buckets: T[]): Array<T & { percentage: number }> {
  const total = buckets.reduce((sum, bucket) => sum + bucket.count, 0);
  if (total === 0) return buckets.map((bucket) => ({ ...bucket, percentage: 0 }));
  const shares = buckets.map((bucket) => (bucket.count / total) * 100);
  const floors = shares.map((share) => Math.floor(share));
  const remainder = 100 - floors.reduce((sum, floor) => sum + floor, 0);
  const byRemainder = shares.map((share, index) => ({ index, fraction: share - floors[index] }))
    .sort((a, b) => b.fraction - a.fraction);
  const percentages = [...floors];
  for (let i = 0; i < remainder; i += 1) percentages[byRemainder[i].index] += 1;
  return buckets.map((bucket, index) => ({ ...bucket, percentage: percentages[index] }));
}
```

**Linear-interpolation percentile** — standard C = 7 method, `null` for empty samples:

```typescript
private static percentile(sortedValues: number[], p: number): number | null {
  if (sortedValues.length === 0) return null;
  if (sortedValues.length === 1) return Math.round(sortedValues[0]);
  const index = p * (sortedValues.length - 1);
  const lower = Math.floor(index);
  const upper = Math.ceil(index);
  if (lower === upper) return Math.round(sortedValues[lower]);
  const weight = index - lower;
  return Math.round(sortedValues[lower] * (1 - weight) + sortedValues[upper] * weight);
}
```

**`caseTypeResolution` no longer runs one findMany per case type** — the closed set is fetched once and grouped in-memory via a `Map`, dropping N+1 queries.

### 3. backend/report.types.ts — `resolutionTrend` reshape (median/p25/p75/n)

```typescript
export interface resolutionTrend {
  /** calendar-month bucket, e.g. "2026-06" */
  month: string;
  /** count of cases closed in this bucket */
  n: number;
  /** median days-to-close; null when no cases closed that month (renders as a gap) */
  median: number | null;
  /** 25th percentile days-to-close; null when no cases closed that month */
  p25: number | null;
  /** 75th percentile days-to-close; null when no cases closed that month */
  p75: number | null;
}
```

Breaking shape change for the API. No consumers outside this repo touch it (only the frontend under `frontend/src/features/reports/` which is updated in the same PR).

### 4. backend/report.service.spec.ts — every-mock-call assertions + new coverage

New `getAllCaseWhereClauses()` helper collects `where.AND` from **every** recorded Prisma mock call (count, groupBy, findMany), so a single passing `toHaveBeenCalledWith` no longer masks other queries that missed the filter. Applied to both the FRAUD_AND_AML container filter test and the new abandoned-exclusion test — this is the direct fix for the CodeRabbit finding.

New `getCaseAgeing` tests cover:

- Abandoned exclusion asserted across every recorded findMany call (≥3 calls: open, closed window, trend).
- Open backlog uses `notIn CLOSED_STATUSES`; closed set uses `in CLOSED_STATUSES` **with** `updated_at` window.
- Non-overlapping 15-30 / 30+ tiers.
- 8-row `ageingByStatus` seeded from open-eligible enum values.
- STATUS_03_RETURNED absent from the by-status bar.
- Ageing distribution percentages sum to exactly 100.
- `caseDetails` surfaces raw `investigatorId` only.
- 6-bucket trend with `n`, `median`, `p25`, `p75` present on every bucket.
- Each of caseType / priority / investigator filters flows to both the open and closed queries.

### 5. frontend Cases dashboard — Assignee column and filter

`CaseFilters.tsx` gets an Assignee `<select>` populated from `useInvestigatorSupervisorList`, and `CasesTable.tsx` renders a new Assignee column resolved via `getAssigneeFullName`. `casesTable.utils.ts` stops fabricating the display string:

```diff
-    assignee: backendCase.user_role === 'owner' ? 'Current User' : 'Assigned User',
+    // Raw owner id - resolved to a display name client-side (CasesTable),
+    // same pattern as the Reports feature. undefined means unassigned.
+    assignee: backendCase.case_owner_user_id ?? undefined,
```

`useCaseDashboard.ts` threads `assigneeFilter` through as an `ownerId` query param, and clears it in the reset handler.

### 6. frontend/reports — Case Ageing UI split, section headers, new Resolution Trend chart

**New `ReportSectionHeader`** marks the two dataset scopes:

```tsx
<ReportSectionHeader title="Open Backlog" badge="Live snapshot · as-of-now" badgeColor="green" />
// stats cards + bar/donut + details table
<ReportSectionHeader title="Closed Throughput" badge="Windowed · closed_at in period" badgeColor="blue" />
// avg resolution days card + type-resolution bar + median/P25-P75/n composed chart
```

**`CaseAgeingStatsCards.tsx`** — a null `avgCaseAge` renders as `N/A` (previously coerced to `0`, which was misleading — no open cases and a fresh 0-day open case both showed the same tile). `CountStatsCard` is a new sibling for integer counts. Exports `DaysStatsCard` for the Closed Throughput placement.

**`CaseAgeingBarChart.tsx`** — the important regression fix here is folding the count into the single Y-axis tick label:

```typescript
export const formatStatusTickLabel = (status: string, total: number): string =>
  `${status} (${total})`;
```

A prior version used a second category YAxis (`dataKey="total"`) purely to display counts. That desynced the Tooltip's hover-row resolution from the hovered bar — hovering "Pending Case Creation Approval" showed Draft's counts. The fix is regression-covered in the new test block (spec 3+ tests under `describe('formatStatusTickLabel …')`).

**`ResolutionTimeTrendChart.tsx`** — full rewrite from `LineChart(avgDays)` to `ComposedChart` with:

- median as the primary `Line` (`connectNulls={false}`, so an empty month is a gap).
- P25-P75 rendered as two stacked `Area`s (transparent base, coloured height).
- sample size (n) as a right-axis `Bar` scaled to `maxN * 4` so it never dominates the chart visually.
- custom tooltip showing `Median / P25-P75 / n=…`, or "No cases closed" for a null bucket.

### 7. frontend/reports — filter wiring end-to-end, All Time option

`useReports.ts` — all three report hooks now accept `filters` and fold it into the TanStack Query key:

```typescript
export const useCaseAgeing = (
  dateRange?: string,
  filters?: { caseType: string; priority: string; investigator: string },
): ReturnType<typeof useQuery<CaseAgeingData>> =>
  useQuery<CaseAgeingData>({
    queryKey: ['reports', 'case-ageing', dateRange, filters],
    queryFn: async () => await reportsService.getCaseAgeingData(dateRange, filters),
    ...
  });
```

`reportsService.ts` uses `URLSearchParams` to build query strings and appends each filter only if truthy (avoids sending empty string params).

`CaseStatusReport.tsx` — `dateRange` default changes from `'last30'` to `'all'`; every child report page receives `filters` from the same `useState` reference (stable — TanStack Query's deep-equal comparison keeps the cache warm).

`options.ts` — `'all'` added to both `DATE_RANGE_OPTIONS` and `DATE_RANGE_LABELS`.

### 8. frontend/reports — helpers: `getClientDateRange`, `mapEvidenceToTasks`

**`getClientDateRange.ts`** — mirrors the backend's `getDateRange` for Evidence Findings, which has no dedicated backend endpoint and applies filtering to the `/cases/all` population client-side:

```typescript
const { startDate, endDate } = getClientDateRange(dateRange);
const cases = allCases.filter((caseItem) => {
  const createdAt = caseItem.created_at ? new Date(caseItem.created_at as string) : null;
  if (createdAt && (createdAt < startDate || createdAt > endDate)) return false;
  if (filters?.caseType && caseItem.case_type !== filters.caseType) return false;
  if (filters?.priority && caseItem.priority !== filters.priority) return false;
  if (filters?.investigator && caseItem.case_owner_user_id !== filters.investigator) return false;
  return true;
});
```

The `'all'` case in `getClientDateRange` sets `startDate = new Date(0)` so nothing is dropped.

**`mapEvidenceToTasks.ts`** — extracted from `reportsService.getEvidenceFindingsData` purely to stay under the max-lines lint budget. Byte-for-byte the same logic.

[↑ Back to top](#pr-review-cms-246--feat-redesign-case-ageing-report-wire-report-filters-and-add-case-assignee-visibility)

---

## Code Quality Analysis

### Strengths

1. **Filter composition through a single choke point.** `withNonContainerCaseFilter` already existed; the new `EXCLUDE_ABANDONED_FILTER` composes with it so every consumer picks up both. There is no place where you have to remember to add the abandoned exclusion — the type system and the helper force it. The updated test iterates every mock call to enforce that behaviour rather than trusting a single match.
2. **The two-dataset split is well-motivated and well-documented.** The block comment above `getCaseAgeing` (`Case Ageing is fed by two lifecycle-scoped datasets, not one shared query…`) is exactly the kind of "why" that survives when the code drifts. The UI section headers (`Live snapshot · as-of-now` vs `Windowed · closed_at in period`) make the split visible to the reader, not only the developer.
3. **Non-overlapping tier ranges.** The previous 15-30 / 30+ tiles both counted a 30-day case (`> 15` and `>= 30`). The new `> 15 && < 30` for the middle tier is a real correctness fix, with explicit test coverage.
4. **Percentages reconciled to sum to exactly 100.** Largest-remainder is the right choice here — donuts with `50.0 + 25.0 + 20.0 + 5.0 = 100.0` look fine, but with 33% / 33% / 34% -type samples the old independent-round pattern drifts. Tested.
5. **N+1 queries dropped in `caseTypeResolution`.** The old code did `Object.values(CaseType).filter(…).map(async …)` — one `findMany` per case type. The new code fetches the closed set once and groups it in-memory via a `Map`.
6. **Regression coverage for the earlier tooltip/axis desync bug.** `formatStatusTickLabel` is exported for direct unit testing, and the test file has a specific `describe` block explaining what the second-YAxis pattern used to break.
7. **`avgCaseAge: null` propagates through the UI as N/A** instead of `0`. A cold report tile that reads `0 days` is a genuinely misleading signal.
8. **Backend Swagger schema documents the open/closed split explicitly** (the `description` on the `stats` object literally names the exception: "…avgResolutionTime is the one exception: windowed on dateRange, from the closed set."). Downstream consumers reading the Swagger get the semantics without reading the code.
9. **Investigator role handling is preserved and explicit.** The controller ignores the `investigator` query param for investigator-role callers (they're already scoped via `requestingUserId`), and the comment above that line explains the "same rule as case-status" precedent.
10. **`URLSearchParams` on the frontend service** replaces string interpolation and correctly omits empty/missing filters — safer than the old `?dateRange=${dateRange ?? 'last30'}` template.
11. **`connectNulls={false}` on the median line + `null`-preserving percentile** makes an empty month a genuine gap on the axis instead of a fabricated straight line through zero.

### Issues and Observations

#### Issue 1 — Unvalidated enum casts on `case_type` and `priority` filters

**Severity: Minor (Code Quality / Data Integrity)**

`buildCommonCaseFilters` and `getInvestigatorWorkload` both assign the raw query-string value straight into the Prisma `where`:

```typescript
// report.service.ts:138
if (filters?.caseType) where.case_type = filters.caseType;
if (filters?.priority) where.priority = filters.priority;

// report.service.ts:214 (getInvestigatorWorkload)
if (filters?.caseType) scopeFilters.case_type = filters.caseType as CaseType;
if (filters?.priority) scopeFilters.priority = filters.priority as Priority;
```

The controller declares them as `?: string`, so any arbitrary string reaches Prisma. Prisma will throw at query time if the string isn't a valid enum, so this doesn't produce corrupt data — it just surfaces as a 500 rather than a 400. The cost of a small runtime guard (`Object.values(CaseType).includes(filters.caseType as CaseType) ? filters.caseType : undefined`) is low and would give a nicer error boundary. Not blocking. `investigator` (`case_owner_user_id`) is a free string in the schema so it doesn't need the same treatment.

#### Issue 2 — `useReports.ts` filter type shape is misleading

**Severity: Informational (Code Quality)**

The hook signatures declare:

```typescript
filters?: { caseType: string; priority: string; investigator: string }
```

Marking the individual keys as required (rather than `caseType?: string`, etc.) implies every caller must set all three; in practice the shape is populated from a `useState({ caseType: '', priority: '', investigator: '' })` seed so empty strings satisfy the type. This works — the `if (filters?.caseType)` checks skip empty strings — but a more honest signature would be `{ caseType?: string; priority?: string; investigator?: string }`. Non-blocking.

#### Issue 3 — `getEvidenceFindingsData` still fetches every case up-front

**Severity: Informational (Performance, Pre-existing)**

The Evidence Findings report calls `apiClient.get('/api/v1/cases/all')` to get the full case list, then filters client-side by dateRange + caseType + priority + investigator. This was already the pre-existing pattern; the PR just wires the filters into the client-side pass rather than adding a backend endpoint. As tenants grow, this fetch will become the bottleneck of the whole report. Not in scope for this PR — flagging it as a follow-up so nobody assumes it was addressed here.

#### Issue 4 — `caseTypeResolution` return type widened from `'FRAUD' | 'AML'` to `string`

**Severity: Informational (Code Quality)**

```diff
- caseTypeResolution: Array<{ caseType: 'FRAUD' | 'AML'; avgDays: number }>;
+ caseTypeResolution: Array<{ caseType: string; avgDays: number }>;
```

This is more honest — the new grouped-in-memory implementation reads whatever `case_type` string comes back from Prisma, not a hard-coded literal. But it does mean a caller can no longer exhaustively match on the union. The Swagger example still says `example: 'FRAUD'`, which is fine. No caller in this repo depends on the narrower literal type, so no immediate breakage. Non-blocking.

#### Issue 5 — Trend axis format label uses month abbreviation only (no year disambiguation)

**Severity: Informational (UX)**

```typescript
const formatMonthLabel = (bucket: string): string => {
  const [year, month] = bucket.split('-').map(Number);
  if (!year || !month) return bucket;
  return new Date(year, month - 1, 1).toLocaleDateString('en-US', { month: 'short' });
};
```

`Feb` renders identically for Feb 2026 and Feb 2025. In the 6-month window this is currently fine (you'll never see the same month twice), but if the window ever expands the axis will become ambiguous. Consider `{ month: 'short', year: '2-digit' }` for future-proofing. Non-blocking.

[↑ Back to top](#pr-review-cms-246--feat-redesign-case-ageing-report-wire-report-filters-and-add-case-assignee-visibility)

---

## Authorization Architecture Analysis

The PR does not change the auth guards themselves, but it does interact with role-scoped visibility inside the reports pipeline. Two things to record explicitly:

1. **Investigator-role callers cannot see other investigators' work via the new filter.** The controller sets `investigator: isInvestigator ? undefined : investigator` before passing filters to the service. The service then applies `applyInvestigatorScope(baseFilters, requestingUserId)` for the investigator role, restricting the OR clause to (a) their own cases, (b) cases with a task assigned to them, (c) all DRAFT / READY_FOR_ASSIGNMENT cases (the visible pool), and (d) unowned PENDING_CASE_CREATION_APPROVAL cases. An investigator setting `?investigator=OTHER_USER_ID` in the URL is silently ignored — not proxied to Prisma. This matches the existing "same rule as case-status" precedent in the codebase, and the comment above the line documents it. ✅ Correct.
2. **The Dashboard `assigneeFilter` reaches the backend via the existing `ownerId` query param.** The backend `ownerId` filter already existed on `/api/v1/cases/all` before this PR — the change here is exposing it in the UI. Whatever role-scoping the `ownerId` filter had before still applies (the PR does not touch that codepath). ✅ Unchanged.

Who can access what after this PR:

- **Supervisor** — can filter reports and the dashboard by any investigator's owner id, and by caseType/priority. Unchanged capability except that the filters are actually applied now.
- **Investigator** — cannot filter reports by a different investigator (query param ignored); can still see the pool of DRAFT / READY_FOR_ASSIGNMENT + their own cases in every filtered view. Newly blocked from setting `?investigator=X` to peek at X's workload — this is the intended tightening, not a regression.
- **Compliance officer** — no auth-relevant change from this PR.

[↑ Back to top](#pr-review-cms-246--feat-redesign-case-ageing-report-wire-report-filters-and-add-case-assignee-visibility)

---

## Security Assessment

| Concern                                 | Assessment                                                                                                                                                                                                                                                                                                                        |
| --------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Unvalidated user input reaching Prisma  | `caseType` / `priority` query params flow into Prisma as strings without an early enum guard (Issue 1). Prisma will reject invalid enum values at query time, so this surfaces as a 500 rather than data leakage — no injection risk because Prisma parameterises. Minor cleanup, not a vulnerability.                       |
| Cross-tenant / cross-role data leakage  | `tenantId` is always taken from the auth token, never from the query string. The controller strips the `investigator` param for investigator-role callers before passing it to the service. `applyInvestigatorScope` wraps the where-clause when `requestingUserId` is set. No change in the guarantees the PR relies on. |
| URL construction / open-redirect / SSRF | `URLSearchParams` is used correctly on the frontend service; no `document.location` mutation or unvalidated URL concatenation.                                                                                                                                                                                                |
| Dependency risk                         | No dependency changes (no lockfile churn in the diff).                                                                                                                                                                                                                                                                            |
| Sensitive data exposure                 | The Cases table now surfaces the raw`case_owner_user_id` on hover of the Case Ageing details "User ID" column. This is the same information the report already had internally — no new leak. Investigator scoping still applies to what reaches the UI.                                                                        |
| njsscan / CodeQL                        | Both green on the current HEAD SHA`e7402270`.                                                                                                                                                                                                                                                                                   |

No new security vulnerabilities introduced by this PR.

[↑ Back to top](#pr-review-cms-246--feat-redesign-case-ageing-report-wire-report-filters-and-add-case-assignee-visibility)

---

## Test Coverage

**Backend (`backend/test/report.service.spec.ts`):**

- ✅ Abandoned exclusion asserted across **every** case-scoped Prisma call (the CodeRabbit-corrected pattern).
- ✅ FRAUD_AND_AML container filter asserted across every call.
- ✅ New `getCaseAgeing` tests cover: two-dataset window shape, tier boundaries, seeded status axis (8 rows exactly), STATUS_03_RETURNED exclusion, distribution percentages sum to 100, raw investigatorId on details, 6-bucket trend, per-filter application on both open and closed queries.
- ✅ New `getInvestigatorWorkload` tests cover the caseType/priority/investigator filter path.
- Percentile helper and `reconcilePercentages` are exercised indirectly through the higher-level tests (a direct unit test for each would be a nice-to-have but not required — the invariant "percentages sum to 100" is asserted).

**Frontend:**

- ✅ `CaseFilters.test.tsx` — assignee dropdown renders investigators + supervisors and fires the change handler.
- ✅ `CasesTable.test.tsx` — Assignee column, resolved name, N/A for unassigned, correct colspans.
- ✅ `casesTable.utils.test.ts` — `assignee` returns raw `case_owner_user_id` or `undefined`.
- ✅ `useCaseDashboard.test.ts` — `assigneeFilter` maps to `ownerId` query param and is omitted when cleared.
- ✅ `caseService.test.ts` — `ownerId` param inclusion/omission.
- ✅ `CaseAgeingBarChart.test.tsx` — new `formatStatusTickLabel` regression block for the desync bug.
- ✅ `CaseAgeingPieChart.test.tsx` — backend-reconciled percentages, `50% · 20 cases` legend format.
- ✅ `CaseAgeingStatsCards.test.tsx` — three-card layout, N/A for null `avgCaseAge`, resolution card no longer here.
- ✅ `CaseAgeingTable.test.tsx` — raw User ID + resolved Investigator as separate columns, N/A for unassigned.
- ✅ `ResolutionTimeTrendChart.test.tsx` — updated for `ComposedChart` + median/P25-P75 band + n bars + null bucket.
- ✅ `useReports.test.tsx` — filter forwarding for all three hooks.
- ✅ `reportsService.test.ts` — filter query params in the URL, client-side date-range filtering for Evidence Findings.
- ✅ `CaseAgeingReport.test.tsx` — updated data shapes.

**PR checklist:**

- [X] Locally
- [X] Development Environment
- [ ] Not needed
- [X] Husky successfully run
- [X] Unit tests passing and Documentation done

**CI:** All checks green on HEAD (CodeQL, DCO, dependency-review, encoding, hadolint, node-ci build/style/tests, gpg-verify, njsscan, PR title validation).

[↑ Back to top](#pr-review-cms-246--feat-redesign-case-ageing-report-wire-report-filters-and-add-case-assignee-visibility)

---

## CodeRabbit Activity

CodeRabbit ran two passes on this PR. The single actionable finding was resolved in the final push.

### Pass 1 — Initial review

**Commit reviewed:** `d6731bee` — *"fix: Assigne Filter"*
**Findings:** 1 actionable comment

| Finding                                                                                                                                                                                                                                                                                                                                                                                                                                | Severity                 | Status                                                                                                                                                                                         |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| The new`excludes abandoned cases…` test in `report.service.spec.ts` uses a single `toHaveBeenCalledWith`, which matches one invocation; other case-scoped `count`/`groupBy`/`findMany` queries could still slip through untested. Iterate over all mock calls and assert `where.AND` contains the exclusion on each. ([discussion](https://github.com/tazama-lf/case-management-system/pull/246#discussion_r3587776073)) | 🟡 Minor (Test Coverage) | ✅ Resolved in commit`e7402270` — the follow-up `fix: Code Rabbit issues` introduced `getAllCaseWhereClauses()` and applied it to both the FRAUD_AND_AML and abandoned-exclusion tests. |

### Pass 2 — Confirmation

**Commit reviewed:** `fad65552`
**Findings:** 0 actionable comments — CodeRabbit posted "Confirmed — the updated test now uses `getAllCaseWhereClauses()` to collect `where.AND` from every count, groupBy, and findMany call and asserts the abandoned-status exclusion across all of them via `forEach`."

**Reconciled against my own findings:** the CodeRabbit finding is corroborated by my Test Coverage section — the `getAllCaseWhereClauses()` pattern is exactly the right shape and is now applied consistently. No divergence.

[↑ Back to top](#pr-review-cms-246--feat-redesign-case-ageing-report-wire-report-filters-and-add-case-assignee-visibility)

---

## Summary and Verdict

**Verdict: Approved**

This is a substantial and well-shaped PR. The Case Ageing rewrite is the most consequential piece — splitting the report into "as-of-now open backlog" and "windowed closed throughput" removes a genuine reporting ambiguity (stat cards reflecting cases from months ago while the chart next to them showed today's snapshot). Every case-scoped Prisma query now composes through `withNonContainerCaseFilter`, which in turn applies both the container-case exclusion and the new abandoned-case exclusion, so the invariant is enforced at the type-and-helper level rather than by convention. The tests catch every one of those queries, not just the first one that happens to match — that is a real hardening of the test suite.

The filter-wiring fix (previously the UI showed Case Type / Priority / Investigator dropdowns that were silent no-ops on three of four tabs) is a straightforward correctness win, and the new Assignee column + filter on the Dashboard closes an obvious workflow gap. The regression coverage for the earlier tooltip/axis desync bug in `CaseAgeingBarChart` is exactly the discipline you want to see after a UI bug — a bespoke exported helper with its own targeted test block that explains what used to break.

The five items in Issues and Observations are all Minor / Informational — small type-safety tightenings, a stylistic reflection about the `useReports` hook signature, and a couple of pre-existing / cosmetic concerns. None of them block merge. CI is fully green, CodeRabbit's one actionable finding was resolved, and the author's five-commit sequence is coherent (`abandoned`, `case ageing`, `dashboard assignee`, `filter`, `coderabbit fix`).

### Blocking

None.

### Non-blocking but recommended

1. **Guard the `caseType` / `priority` enum casts** — a short `Object.values(CaseType).includes(...)` check would turn a Prisma 500 into an early 400 for bad query-string values (Issue 1).
2. **Loosen the `useReports.ts` filter type** — mark the inner keys optional to match how callers actually use them (Issue 2).
3. **Add the year to the trend axis month label** — `{ month: 'short', year: '2-digit' }` future-proofs the axis if the 6-month window is ever widened (Issue 5).

[↑ Back to top](#pr-review-cms-246--feat-redesign-case-ageing-report-wire-report-filters-and-add-case-assignee-visibility)

---

## GitHub Review Comment

````markdown
**Approved**

Substantial and well-shaped PR. The Case Ageing rewrite splits the report into an as-of-now open backlog and a windowed closed-throughput set, which removes a genuine reporting ambiguity, and the abandoned-case exclusion is enforced through the single `withNonContainerCaseFilter` choke point so every case-scoped Prisma query picks it up by construction. The updated tests assert the filter across **every** recorded mock call — not just the first — which is exactly the right pattern. Filter wiring on Case Ageing / Investigator Workload / Evidence Findings is a real correctness fix (they were silent no-ops before), and the new Assignee column + filter on the Dashboard closes an obvious workflow gap. CI green, CodeRabbit's one actionable finding resolved. Nothing blocking — a few small non-blocking tightenings below.

---

### Non-blocking (please address in this PR if possible)

**1. Guard the `caseType` / `priority` enum casts**

In `backend/src/modules/report/report.service.ts` — `buildCommonCaseFilters` and `getInvestigatorWorkload` assign the raw query-string value straight to Prisma's `case_type` / `priority`:

```typescript
if (filters?.caseType) where.case_type = filters.caseType;                       // buildCommonCaseFilters
if (filters?.caseType) scopeFilters.case_type = filters.caseType as CaseType;   // getInvestigatorWorkload
```

If someone hits the endpoint with `?caseType=BOGUS`, Prisma throws at query time and it surfaces as a 500 rather than a 400. A tiny guard would turn that into an early rejection with a nicer error boundary:

```typescript
const caseType = Object.values(CaseType).includes(filters?.caseType as CaseType)
  ? (filters?.caseType as CaseType)
  : undefined;
if (caseType) where.case_type = caseType;
```

Same pattern for `priority`. No injection risk — Prisma parameterises — this is purely about the error surface.

**2. Loosen the `useReports.ts` filter type to match how callers use it**

In `frontend/src/features/reports/hooks/useReports.ts` — `useCaseAgeing`, `useInvestigatorWorkload`, `useEvidenceFindings` all declare:

```typescript
filters?: { caseType: string; priority: string; investigator: string }
```

The inner keys are marked required, but callers seed them from `useState({ caseType: '', priority: '', investigator: '' })` and rely on the `if (filters?.caseType)` truthiness check. The declared shape implies every caller must set all three; a more honest signature is:

```typescript
filters?: { caseType?: string; priority?: string; investigator?: string }
```

**3. Add the year to the resolution-trend month label**

In `frontend/src/features/reports/components/ResolutionTimeTrendChart.tsx#formatMonthLabel`:

```typescript
return new Date(year, month - 1, 1).toLocaleDateString('en-US', { month: 'short' });
```

Fine for the current 6-month window (no month appears twice), but ambiguous if the window is ever widened. Consider:

```typescript
return new Date(year, month - 1, 1).toLocaleDateString('en-US', { month: 'short', year: '2-digit' });
```

---

### Nice-to-haves (out of scope, follow-up)

- **Evidence Findings still pulls the full `/api/v1/cases/all` and filters client-side.** This PR wires the filters into the client-side pass rather than adding a backend endpoint, which is fine as an incremental step, but the fetch will become the bottleneck of the whole report as tenants grow. Worth a follow-up ticket for a proper backend evidence-findings endpoint.

Great work on this one — the "why" comments above `getCaseAgeing` and the exported `formatStatusTickLabel` regression coverage are the kind of discipline that survives a lot of future churn.
````

[↑ Back to top](#pr-review-cms-246--feat-redesign-case-ageing-report-wire-report-filters-and-add-case-assignee-visibility)


---
---
---

## Follow-up Review (2026-07-16)

**Reviewed commit:** `ab1dfe24` — *"fix: Change Request"* (2026-07-16)
**Reviewed against:** CHANGES_REQUESTED on commit `e7402270` by `ahmad-paysys` (three items posted as non-blocking guidance)
**Delta reviewed:** `git diff e7402270..ab1dfe24 --stat` → 4 files, +27 / -12
**Developer response** ([issuecomment](https://github.com/tazama-lf/case-management-system/pull/246#issuecomment-3587776073)):

> 1) Guard the caseType / priority enum casts: Added parseCaseType/parsePriority static validators in **report.service.ts** that throw BadRequestException (400) instead of letting Prisma blow up with a 500 on bad caseType/priority query values. Applied in both buildCommonCaseFilters and getInvestigatorWorkload.
>
> 2) Loosened filter types: Made caseType/priority/investigator optional in **useReports.ts** and, since it surfaced a matching mismatch, also in **reportsService.ts** Verified against all real callers and tests all pass optional fields fine.
>
> 3) Month label year added year: '2-digit' to formatMonthLabel in **ResolutionTimeTrendChart.tsx**

### Resolution Status

#### Item 1 — Guard the `caseType` / `priority` enum casts

**Status: RESOLVED**

Two new static validators added at [backend/src/modules/report/report.service.ts:107-119](repos/case-management-system/backend/src/modules/report/report.service.ts#L107-L119):

```typescript
private static parseCaseType(value: string): CaseType {
  if (!Object.values(CaseType).includes(value as CaseType)) {
    throw new BadRequestException(`Invalid caseType: ${value}`);
  }
  return value as CaseType;
}

private static parsePriority(value: string): Priority {
  if (!Object.values(Priority).includes(value as Priority)) {
    throw new BadRequestException(`Invalid priority: ${value}`);
  }
  return value as Priority;
}
```

Applied at both call sites:

- `buildCommonCaseFilters` ([report.service.ts:152-153](repos/case-management-system/backend/src/modules/report/report.service.ts#L152-L153)) — used by `getCaseAgeing` and other consumers of the shared filter helper.
- `getInvestigatorWorkload` ([report.service.ts:685-686](repos/case-management-system/backend/src/modules/report/report.service.ts#L685-L686)) — the second path that was previously casting raw strings.

`BadRequestException` is already imported at line 1 of the file, so NestJS surfaces a 400 with the message instead of a Prisma-thrown 500 for `?caseType=BOGUS` or `?priority=BOGUS`. Exactly the requested shape.

#### Item 2 — Loosen the `useReports.ts` filter type

**Status: RESOLVED**

All four hook signatures in [frontend/src/features/reports/hooks/useReports.ts](repos/case-management-system/frontend/src/features/reports/hooks/useReports.ts) — `useReports`, `useInvestigatorWorkload`, `useCaseAgeing`, `useEvidenceFindings` — now use optional inner keys:

```diff
- filters?: { caseType: string; priority: string; investigator: string },
+ filters?: { caseType?: string; priority?: string; investigator?: string },
```

The author also propagated the same loosening to the service layer at [frontend/src/features/reports/services/reportsService.ts](repos/case-management-system/frontend/src/features/reports/services/reportsService.ts) (four method signatures: `getReportsData`, `getInvestigatorWorkloadData`, `getCaseAgeingData`, `getEvidenceFindingsData`) — a good catch, since a mismatch would have forced callers to satisfy the stricter shape at one boundary or the other. The type shape now honestly reflects usage.

#### Item 3 — Add the year to the trend axis month label

**Status: RESOLVED**

[frontend/src/features/reports/components/ResolutionTimeTrendChart.tsx:24-28](repos/case-management-system/frontend/src/features/reports/components/ResolutionTimeTrendChart.tsx#L24-L28):

```diff
  return new Date(year, month - 1, 1).toLocaleDateString('en-US', {
    month: 'short',
+   year: '2-digit',
  });
```

`Feb 26` vs `Feb 25` now disambiguates the axis if the 6-month window is ever widened. Exact fix requested.

| # | Item                                                      | Status      |
| - | --------------------------------------------------------- | ----------- |
| 1 | Guard the `caseType` / `priority` enum casts              | ✅ Resolved |
| 2 | Loosen the `useReports.ts` filter type (+ service parity) | ✅ Resolved |
| 3 | Add the year to the trend axis month label                | ✅ Resolved |

### New Issues Found in Updated Commits

CodeRabbit posted a follow-up review at 2026-07-16 07:16 UTC on `ab1dfe24` — 9 actionable + 3 nitpicks. I have reconciled every finding against current code below. **One is a genuine Major issue that changes this round's verdict; the rest are Minor/Informational.**

#### Issue F1 — `getInvestigatorWorkload` — `activeCases` count bypasses `scopeFilters`

**Severity: Major (Data Integrity)**

Corroborated with CodeRabbit — [discussion r3593362913](https://github.com/tazama-lf/case-management-system/pull/246#discussion_r3593362913).

At [backend/src/modules/report/report.service.ts:713-722](repos/case-management-system/backend/src/modules/report/report.service.ts#L713-L722) the active-case count omits `scopeFilters`, but its sibling queries in the same block do apply them:

```typescript
const [activeCases, pendingTasks] = await Promise.all([
  this.prisma.case.count({
    where: ReportsService.withNonContainerCaseFilter({
      case_owner_user_id: caseOwnerUserId,
      created_at: { gte: startDate, lte: endDate },
      status: { notIn: [...ReportsService.CLOSED_STATUSES, CaseStatus.STATUS_00_DRAFT] },
      // ← scopeFilters (tenantId / caseType / priority) NOT spread here.
    }),
  }),
  this.prisma.task.count({
    where: ReportsService.withNonContainerTaskCaseFilter(
      { assigned_user_id: caseOwnerUserId, status: { in: [...] } },
      scopeFilters,   // ← scopeFilters passed through here ✅
    ),
  }),
]);
// ...and efficiencyData just below (line 758) spreads ...scopeFilters into its findMany ✅
```

Concretely: a supervisor requesting `?caseType=FRAUD` sees the `activeCases` metric include AML cases as well, while `pendingTasks` and `efficiencyData` on the same row correctly exclude them. The row totals are internally inconsistent. Tenant scoping (`tenantId`) has the same defect — although `tenantId` is derived from the auth token so this is functionally masked in single-tenant callers, it is still a correctness gap.

**Fix:** spread `scopeFilters` into the active-case count so the three siblings share one scope.

```diff
  this.prisma.case.count({
    where: ReportsService.withNonContainerCaseFilter({
      case_owner_user_id: caseOwnerUserId,
      created_at: { gte: startDate, lte: endDate },
      status: { notIn: [...ReportsService.CLOSED_STATUSES, CaseStatus.STATUS_00_DRAFT] },
+     ...scopeFilters,
    }),
  }),
```

And extend the existing spec block at [backend/test/report.service.spec.ts:501-524](repos/case-management-system/repos/case-management-system/backend/test/report.service.spec.ts#L501-L524) to assert that the `case.count` call for `activeCases` includes `tenant_id`, `case_type`, and `priority` in its `where` — my prior-round Test Coverage praise (per-filter application on both open and closed queries) missed this specific query. Corroborated by CodeRabbit [r3593362919](https://github.com/tazama-lf/case-management-system/pull/246#discussion_r3593362919).

#### Issue F2 — Report-page prop types didn't get the `?` loosening

**Severity: Minor (Code Quality)**

Corroborated with CodeRabbit — [discussion r3593362939](https://github.com/tazama-lf/case-management-system/pull/246#discussion_r3593362939) (raised against `CaseAgeingReport.tsx`; the same defect exists in the other two report pages).

The prior round's item 2 loosened the shape in `useReports.ts` and `reportsService.ts`, but the three page-level prop declarations were missed:

```typescript
// CaseAgeingReport.tsx:24
filters?: { caseType: string; priority: string; investigator: string };
// InvestigatorWorkloadReport.tsx:21
filters?: { caseType: string; priority: string; investigator: string };
// EvidenceFindingsReport.tsx:37
filters?: { caseType: string; priority: string; investigator: string };
```

These pages are called from `CaseStatusReport.tsx` with a `useState({ caseType: '', priority: '', investigator: '' })` seed so nothing breaks at runtime, but the whole point of item 2 was to make the type honest across every layer. Trivial to apply the same diff.

```diff
- filters?: { caseType: string; priority: string; investigator: string };
+ filters?: { caseType?: string; priority?: string; investigator?: string };
```

#### Issue F3 — Age-band label "16-30 days" doesn't match the `> 15 && < 30` boundary

**Severity: Minor (Functional Correctness / UX)**

Corroborated with CodeRabbit — [r3593362916](https://github.com/tazama-lf/case-management-system/pull/246#discussion_r3593362916) (backend) and [r3593362926](https://github.com/tazama-lf/case-management-system/pull/246#discussion_r3593362926) (frontend).

Implementation at [report.service.ts:1149-1153](repos/case-management-system/backend/src/modules/report/report.service.ts#L1149-L1153):

```typescript
age16to30: cases.filter((c) => c.ageDays > 15 && c.ageDays < 30).length,
age30Plus: cases.filter((c) => c.ageDays >= 30).length,
```

Day 30 belongs exclusively to `30+`. The bucket keyed `age16to30` counts 16-29. But the public label at [report.service.ts:1180](repos/case-management-system/backend/src/modules/report/report.service.ts#L1180) is `'16-30 days'`, which reads to a viewer as "30 goes here." Rename the label to `'16-29 days'` and mirror the change in the frontend labels used by `CaseAgeingStatsCards.tsx`, `CaseAgeingBarChart.tsx`, `CaseAgeingPieChart.tsx`, plus the Swagger `casesOver15Days` description at `report.controller.ts:450-455`. This is aligned with my prior-round "Non-overlapping tier ranges" strength note — CodeRabbit caught the label had drifted out of sync with the boundary I approved.

Internal keys (`age16to30`) can stay — this is a display-label change only.

#### Issue F4 — `caseDetails` is missing from the Swagger response schema

**Severity: Minor (API Contract)**

Corroborated with CodeRabbit — [r3593362906](https://github.com/tazama-lf/case-management-system/pull/246#discussion_r3593362906).

The service returns a `caseDetails` array (per [report.service.ts:1187-1198](repos/case-management-system/backend/src/modules/report/report.service.ts#L1187) — includes `caseId`, `type`, `investigatorId`, etc.), but the Swagger `@ApiResponse` schema in `report.controller.ts` at lines 486-513 documents only `caseTypeResolution` and `resolutionTrend`. Generated clients won't see `caseDetails`. Add the array + item shape to the schema.

#### Issue F5 — `mapEvidenceToTasks` fallback ID collision under same-tick execution

**Severity: Minor (Data Integrity)**

Corroborated with CodeRabbit — [r3593362931](https://github.com/tazama-lf/case-management-system/pull/246#discussion_r3593362931).

At [helpers/mapEvidenceToTasks.ts:27-38](repos/case-management-system/frontend/src/features/reports/helpers/mapEvidenceToTasks.ts#L27-L38), evidence items lacking every ID field fall through to `` `unknown_${Date.now()}` ``. Inside a synchronous `.map()` these all resolve in the same millisecond tick, giving duplicate React keys. The two-line fix is to accept the array `index` and append it to the fallback: `` `unknown_${Date.now()}_${index}` ``. Real bug but rare — requires evidence items with no id / evidenceId / evidence_id, which suggests upstream data hygiene is already off. Non-blocking.

#### Issue F6 — `CaseAgeingBarChart` "empty" helper bar appears in tooltip

**Severity: Minor (UX)**

Corroborated with CodeRabbit — [r3593362921](https://github.com/tazama-lf/case-management-system/pull/246#discussion_r3593362921).

The prior round's regression fix folds `total` into the tick label so the second YAxis is gone, but the "empty" spacer Bar still participates in the tooltip. Setting `tooltipType="none"` on that Bar excludes the helper from the hover payload without touching the formatter. One-line prop change.

#### Issue F7 — `exportUtils.ts` fallback uses `??` where `||` would defend against empty strings

**Severity: Informational (Code Quality)**

Partial disagreement with CodeRabbit — [r3593362950](https://github.com/tazama-lf/case-management-system/pull/246#discussion_r3593362950). CodeRabbit's premise is that `getAssigneeFullName` returns `''` for unknown IDs; the current hook implementation at [useInvestigatorSupervisorList.ts:113-134](repos/case-management-system/frontend/src/features/cases/hooks/useInvestigatorSupervisorList.ts#L113-L134) actually returns `assignee ?? 'N/A'`, never an empty string. So the `?? 'N/A'` in exportUtils is already dead code, not a live bug. Switching to `||` (as suggested) is still defensively correct if the hook ever changes and costs nothing, but I would not treat it as blocking or even strictly required — call it a defensive tightening.

#### Issue F8 — `CasesTable` assignee tooltip leaks raw user id (Nitpick)

**Severity: Informational (UX / Trivial)**

Corroborated with CodeRabbit nitpick. At [CasesTable.tsx:176-180](repos/case-management-system/frontend/src/features/cases/components/CasesTable.tsx#L176-L180), the `<span title={c.assignee ?? ''}>` shows the raw `case_owner_user_id` on hover. The full name is already displayed in the cell so the tooltip adds no value. Trivial to drop. Note: this is not a security leak in the auth sense — the row already carries the id — but a UX polish.

#### Issue F9 — Test coverage gaps flagged by CodeRabbit nitpicks

**Severity: Informational (Test Coverage)**

Two nitpick tests worth adding:

- [InvestigatorWorkloadReport.test.tsx:302-307](repos/case-management-system/frontend/src/features/reports/pages/__tests__/InvestigatorWorkloadReport.test.tsx#L302-L307): add a render-with-filters case so `filters` prop forwarding is covered ([r comment](https://github.com/tazama-lf/case-management-system/pull/246#discussion_r_nitpick_1)).
- [reportsService.test.ts:596-630](repos/case-management-system/frontend/src/features/reports/services/__tests__/reportsService.test.ts#L596-L630): exercise `caseType`, `priority`, `investigator` each with a fixture that differs from the match on exactly that one filter — the current test would still pass if two of the three filters silently broke.

Not blocking.

### Updated Verdict

**Verdict: Changes Requested (follow-up)**

The three items from my prior round are all resolved cleanly. However, CodeRabbit's follow-up review on `ab1dfe24` caught a genuine Major that both of our prior reviews missed: `activeCases` in `getInvestigatorWorkload` doesn't apply the caseType / priority / tenant scope that `pendingTasks` and `efficiencyData` in the same block do apply. That's a real internal-consistency bug in the filter wiring — the whole point of the round-1 fix — and it also invalidates my prior-round test-coverage praise, because the new spec doesn't assert scope on this specific `case.count` call. Fix is one line + one test-scope extension. Everything else in the CodeRabbit pass is Minor / Informational: three trivial polish items (page prop `?`, "16-30" → "16-29" label, missing `caseDetails` in Swagger), one edge-case ID-collision fix, one tooltip nit, plus test-coverage nits.

### Blocking

1. **`activeCases` filter bypass in `getInvestigatorWorkload`** — spread `...scopeFilters` at [report.service.ts:713-722](repos/case-management-system/backend/src/modules/report/report.service.ts#L713-L722) so tenant / caseType / priority apply to `activeCases` as they do to `pendingTasks` and `efficiencyData`. Extend the spec at [report.service.spec.ts:501-524](repos/case-management-system/backend/test/report.service.spec.ts#L501-L524) to assert those scopes on the `activeCases` `case.count` call.

### Non-blocking but recommended

2. **Loosen the three report-page prop types** — mirror the round-1 hook / service loosening in `CaseAgeingReport.tsx`, `InvestigatorWorkloadReport.tsx`, `EvidenceFindingsReport.tsx` (`caseType?: string; priority?: string; investigator?: string`). Trivial and completes the type honesty the last round started.
3. **Rename the "16-30 days" label to "16-29 days"** (service, controller Swagger, and the three frontend components: `CaseAgeingStatsCards.tsx`, `CaseAgeingBarChart.tsx`, `CaseAgeingPieChart.tsx`). Internal keys stay.
4. **Add `caseDetails` to the Swagger response schema** in `report.controller.ts` around lines 486-513.
5. **`mapEvidenceToTasks` fallback ID** — include the array index in the `` `unknown_${Date.now()}_${index}` `` fallback.
6. **`CaseAgeingBarChart` empty bar tooltip** — set `tooltipType="none"` on the "empty" helper `<Bar>`.
7. **`CasesTable` assignee tooltip** — drop the `title` attribute on the assignee span.
8. **Test-coverage nits** — filter-forwarding test in `InvestigatorWorkloadReport.test.tsx`; per-filter isolation in `reportsService.test.ts` evidence-findings block.

### CodeRabbit Reconciliation (Pass 3 on `ab1dfe24`)

| # | CodeRabbit finding | Verdict against current code | Also in my Issues section |
| - | ------------------ | ----------------------------- | ------------------------- |
| CR-1 | `activeCases` bypasses `scopeFilters` (r3593362913) | ✅ Corroborated, Major | Issue F1 |
| CR-2 | Test scope missing for `activeCases` (r3593362919) | ✅ Corroborated, Major | Issue F1 (bundled) |
| CR-3 | Page prop types still strict (r3593362939 + siblings) | ✅ Corroborated, Minor | Issue F2 |
| CR-4 | "16-30 days" label / boundary drift (r3593362916 + r3593362926) | ✅ Corroborated, Minor | Issue F3 |
| CR-5 | Missing `caseDetails` in Swagger (r3593362906) | ✅ Corroborated, Minor | Issue F4 |
| CR-6 | `Date.now()` collision in evidence fallback (r3593362931) | ✅ Corroborated, Minor | Issue F5 |
| CR-7 | `CaseAgeingBarChart` empty bar in tooltip (r3593362921) | ✅ Corroborated, Minor | Issue F6 |
| CR-8 | `exportUtils` `??` → `||` (r3593362950) | ⚠️ Partial — premise about `getAssigneeFullName` returning `''` doesn't match current hook impl; still fine as defensive tightening | Issue F7 |
| CR-9 (nit) | Raw id in assignee tooltip | ✅ Corroborated, Informational | Issue F8 |
| CR-10 (nit) | Filter-forwarding test in InvestigatorWorkloadReport | ✅ Corroborated, Informational | Issue F9 |
| CR-11 (nit) | Per-filter isolation in reportsService evidence test | ✅ Corroborated, Informational | Issue F9 |

[↑ Back to top](#pr-review-cms-246--feat-redesign-case-ageing-report-wire-report-filters-and-add-case-assignee-visibility)

---

### GitHub Review Comment (Follow-up)

````markdown
**Changes Requested (follow-up, HEAD `ab1dfe24`)**

All three items from the prior round are resolved cleanly — `parseCaseType` / `parsePriority` guard the enum casts, the filter type is loosened across the hook + service layers, and the trend axis month label now includes the year. Nice catch on propagating the type loosening to `reportsService.ts` as well.

CodeRabbit's follow-up pass on this commit, however, surfaced one real bug that both prior reviews missed: `activeCases` in `getInvestigatorWorkload` doesn't apply the same scope as its siblings. Blocking that one; the rest below are non-blocking polish.

---

### Blocking

**1. `activeCases` in `getInvestigatorWorkload` bypasses `scopeFilters`**

`backend/src/modules/report/report.service.ts` around line 713-722 — the `case.count` for `activeCases` doesn't spread `scopeFilters`, but `pendingTasks` on the task query does, and `efficiencyData` just below (line 758) does too. So a supervisor requesting `?caseType=FRAUD` sees `activeCases` include AML while `pendingTasks` and `efficiencyData` in the same row correctly exclude AML. Tenant scoping has the same defect (masked in single-tenant callers but still wrong).

```diff
 this.prisma.case.count({
   where: ReportsService.withNonContainerCaseFilter({
     case_owner_user_id: caseOwnerUserId,
     created_at: { gte: startDate, lte: endDate },
     status: { notIn: [...ReportsService.CLOSED_STATUSES, CaseStatus.STATUS_00_DRAFT] },
+    ...scopeFilters,
   }),
 }),
```

Please also extend the corresponding spec (`backend/test/report.service.spec.ts:501-524`) to assert `tenant_id` + `case_type` + `priority` on the `activeCases` `case.count` call — the current tests only check investigator discovery, which is what allowed this to slip through.

---

### Non-blocking (please address in this PR if possible)

**2. Loosen the report-page prop types**

The `useReports.ts` / `reportsService.ts` loosening from last round didn't reach the three page-level prop declarations:

```typescript
// CaseAgeingReport.tsx:24, InvestigatorWorkloadReport.tsx:21, EvidenceFindingsReport.tsx:37
filters?: { caseType: string; priority: string; investigator: string };
```

Trivial to apply the same diff:

```diff
- filters?: { caseType: string; priority: string; investigator: string };
+ filters?: { caseType?: string; priority?: string; investigator?: string };
```

**3. Rename the "16-30 days" ageing-distribution label to "16-29 days"**

The boundary is `> 15 && < 30` (day 30 belongs exclusively to the `30+` bucket), so the current label reads to users as "30 goes here." Rename in `backend/src/modules/report/report.service.ts:1180`, the Swagger `casesOver15Days` description in `report.controller.ts:450-455`, and the frontend label consumers (`CaseAgeingStatsCards.tsx`, `CaseAgeingBarChart.tsx`, `CaseAgeingPieChart.tsx`). Internal keys (`age16to30`) can stay — display only.

**4. Add `caseDetails` to the Swagger response schema**

The service already returns `caseDetails` (with `investigatorId` and friends), but the `@ApiResponse` schema in `report.controller.ts` around lines 486-513 stops at `resolutionTrend`, so generated clients don't see it.

**5. `mapEvidenceToTasks` fallback ID collision**

`helpers/mapEvidenceToTasks.ts:27-38` — same-tick `.map()` calls with `Date.now()` produce duplicate keys for evidence items lacking every id field. Include the array index:

```diff
- supportingEvidence: evidences.map((e) => {
+ supportingEvidence: evidences.map((e, index) => {
   ...
-   `unknown_${String(Date.now())}`
+   `unknown_${String(Date.now())}_${index}`
```

**6. Quiet the `CaseAgeingBarChart` empty-helper bar in the tooltip**

Set `tooltipType="none"` on the "empty" spacer `<Bar>` (around line 142-149) so the tooltip excludes the helper series without touching the formatter.

**7. Drop the `CasesTable` assignee `title` attribute**

`frontend/src/features/cases/components/CasesTable.tsx:176-180` — the tooltip shows the raw `case_owner_user_id`, but the full name is already visible in the cell. Removing `title={c.assignee ?? ''}` avoids exposing internal ids on hover.

**8. `exportUtils.ts` — swap `??` to `||` on the investigator fallback (defensive)**

`shared/utils/exportUtils.ts:348-350`. Note: `getAssigneeFullName` in `useInvestigatorSupervisorList` currently returns `assignee ?? 'N/A'`, never `''`, so the current `??` fallback is already dead — but `||` is the safer choice if the hook ever changes.

**9. Two test-coverage nits**

- `frontend/src/features/reports/pages/__tests__/InvestigatorWorkloadReport.test.tsx:302-307` — add a render-with-filters case to lock in `filters` prop forwarding.
- `frontend/src/features/reports/services/__tests__/reportsService.test.ts:596-630` — the evidence-findings test currently passes empty `priority`/`investigator`, so a regression in either would still pass. Add fixtures that differ from the match on exactly one filter each.

---

CodeRabbit's `exportUtils` finding (r3593362950) claims `getAssigneeFullName` returns `''` for unknown IDs — the current hook implementation actually returns `assignee ?? 'N/A'`, never an empty string. So (8) is a defensive tightening rather than a live bug — take it if you like, but it's the least important of the batch.

Great progress overall — the round-1 items look clean, and once (1) is patched with its test the rest is straightforward polish.
````

[↑ Back to top](#pr-review-cms-246--feat-redesign-case-ageing-report-wire-report-filters-and-add-case-assignee-visibility)
