# PR Review: CMS #262 — fix: scope Case Ageing report to owned/assigned work and align its widgets with Case Status

**Repo:** tazama-lf/case-management-system
**Branch:** `paysys/dashboard-fixes` → `dev`
**Author:** sobia-rizwan1567 (Sobia Rizwan)
**Date Reviewed:** 2026-07-17
**Label:** bug
**Size:** +506 / -257 lines across 14 files
**Commits:** 1 (`12a0b20a`)
**State:** OPEN
**HEAD SHA verified:** `12a0b20ab2212da8fe59b1a3f1200173037b54a1`
**Existing approvals:** none — CodeRabbit rate-limited (no review performed); all other CI checks green.

---

## Table of Contents

- [Overview](#overview)
- [What Changed (Detailed)](#what-changed-detailed)
  - [1. backend/src/modules/report/report.service.ts — new scope helper, closed-throughput consolidation, resolutionByOutcome, dynamic trend, status formatter](#1-backendsrcmodulesreportreportservicets)
  - [2. backend/src/modules/report/report.controller.ts — Swagger updates](#2-backendsrcmodulesreportreportcontrollerts)
  - [3. backend/test/report.service.spec.ts — spec updates for the new shape](#3-backendtestreportservicespects)
  - [4. frontend CaseAgeingStatsCards.tsx — rewrite atop shared StatsCard](#4-frontend-caseageingstatscardstsx)
  - [5. frontend CaseAgeingReport.tsx — layout re-slot, wire up resolutionByOutcome](#5-frontend-caseageingreporttsx)
  - [6. frontend ResolutionByOutcomeChart.tsx (new)](#6-frontend-resolutionbyoutcomecharttsx)
  - [7. frontend CaseAgeingTable.tsx — drop User ID column](#7-frontend-caseageingtabletsx)
  - [8. frontend PieChart.tsx — legend label collapsed to one span](#8-frontend-piecharttsx)
  - [9. frontend types + service + tests](#9-frontend-types--service--tests)
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

The PR retunes the **Case Ageing Report** so it answers questions about an investigator's *own* backlog instead of also folding in the shared claimable pool, and pulls its closed-throughput widgets into a single, `dateRange`-consistent set. Concretely:

1. A narrower `applyOwnedOrAssignedScope` helper replaces `applyInvestigatorScope` inside `getCaseAgeing`. Every other report is untouched — this is a Case Ageing-only tightening.
2. The three closed-throughput widgets (`avgResolutionTime`, `resolutionTrend`, and a new `resolutionByOutcome`) now all read from a single `updated_at`-windowed `findMany` — no more three separately-windowed queries silently answering for three different periods. `caseTypeResolution` deliberately stays type-agnostic (it's the FRAUD-vs-AML comparison widget) and runs its own filtered query.
3. `resolutionTrend` switches from a hardcoded 6-month axis to calendar-month buckets across the actual `dateRange`, capped at 24 buckets.
4. `formatStatusName` moves to code-prefixed title case (`"02 Ready For Assignment"` instead of `"02 READY FOR ASSIGNMENT"`), and the stat-card row moves to the shared `StatsCard` component used by the Case Status report.
5. The "User ID" column disappears from the Case Ageing details table.

**Base branch:** targets `dev` — correct for this repo's flow.

**CI:** All checks passing at HEAD (`12a0b20a`). CodeRabbit hit its per-developer PR-review limit and did not review this PR, so extra care is warranted on the reviewer side.

| File | Nature of Change |
|------|------------------|
| `backend/src/modules/report/report.controller.ts` | Swagger: new `resolutionByOutcome` schema, updated `dateRange`/`resolutionTrend`/`caseTypeResolution` descriptions, example status strings switched to title case. |
| `backend/src/modules/report/report.service.ts` | New `applyOwnedOrAssignedScope`; `getCaseAgeing` reworked (consolidated closed-throughput query, dynamic trend buckets, new `resolutionByOutcome`, tier counts read off `ageBuckets`); `formatStatusName` title-cases the trailing words. |
| `backend/test/report.service.spec.ts` | Spec updates: two-query assertion (was three), title-cased status expectations, three new trend-bucketing tests, `resolutionByOutcome` shape test, caseType-filter isolation test for `caseTypeResolution`. |
| `frontend/src/features/reports/components/CaseAgeingStatsCards.tsx` | Rewritten atop the shared, lazy-loaded `StatsCard`; grows from three to four cards (now includes Avg. Resolution Time). |
| `frontend/src/features/reports/components/CaseAgeingTable.tsx` | Drops the raw "User ID" column and its cell; `colSpan={8}` → `7`. |
| `frontend/src/features/reports/components/PieChart.tsx` | Legend label + percentage collapsed into a single `<span>` (removes the separate `label:` span). |
| `frontend/src/features/reports/components/ReportStatsCards.tsx` | Trailing-comma-only cleanup. |
| `frontend/src/features/reports/components/ResolutionByOutcomeChart.tsx` | New file: recharts bar chart of avg resolution days grouped by closed status. |
| `frontend/src/features/reports/pages/CaseAgeingReport.tsx` | Wires up `resolutionByOutcome`, drops the inline `DaysStatsCard`, moves `<CaseAgeingStatsCards>` above the "Open Backlog" section header. |
| Frontend tests (2 test files) | Mock the lazy `StatsCard`, drop User-ID assertions. |
| `frontend/src/features/reports/services/reportsService.ts` | Threads `resolutionByOutcome` through the client's mapping (both happy path and error fallback). |
| `frontend/src/features/reports/types/reports.types.ts` | New `ResolutionByOutcome` interface and field on `CaseAgeingData`. |

[↑ Back to top](#pr-review-cms-262--fix-scope-case-ageing-report-to-ownedassigned-work-and-align-its-widgets-with-case-status)

---

## What Changed (Detailed)

### 1. backend/src/modules/report/report.service.ts

**New scope helper (Case Ageing only).**

```typescript
private applyOwnedOrAssignedScope(baseFilters: any, requestingUserId?: string): any {
  if (!requestingUserId) return baseFilters;
  return {
    AND: [
      baseFilters,
      { OR: [
        { case_owner_user_id: requestingUserId },
        { tasks: { some: { assigned_user_id: requestingUserId } } },
      ] },
    ],
  };
}
```

Narrower than the pre-existing `applyInvestigatorScope`, which also OR-includes the shared claimable pool (unowned `DRAFT` / `READY_FOR_ASSIGNMENT` cases and unowned pending-approval cases). This helper is only used in `getCaseAgeing`; all other reports still call `applyInvestigatorScope` unchanged.

**Swap-in inside `getCaseAgeing`.** Verified via `grep`: all three current call sites (`openWhere`, `closedWhere`, `closedWhereAllTypes`) now use the new helper — no siblings were missed:

```
1158:    const openWhere = this.applyOwnedOrAssignedScope(...);
1242:    const closedWhere = this.applyOwnedOrAssignedScope(...);
1272:    const closedWhereAllTypes = this.applyOwnedOrAssignedScope(...);
```

The old separate trend query is *removed*; `resolutionTrend` now buckets `closedCases`, which is already scoped by the same helper.

**Closed-throughput consolidation.**

```diff
-    // --- Closed throughput: windowed on the closed-at proxy (updated_at). ---
+    // --- Closed throughput: a single close-anchored window (updated_at within
+    // dateRange), shared by avgResolutionTime, resolutionTrend, and
+    // resolutionByOutcome below. Case Type Resolution deliberately uses its
+    // own query that skips the caseType filter - see the comment there.
```

One `findMany` fetches `closedCases` (now selecting `status` in addition to `case_type` / `created_at` / `updated_at`), feeding all three widgets that were previously fetching independently.

**caseTypeResolution isolated query.** New `commonFiltersAllTypes` (priority + investigator + tenant, no `caseType`) is fed to `closedWhereAllTypes`. If a `caseType` filter is active, a second `findMany` is issued; otherwise `closedCasesAllTypes = closedCases` (reference reuse — no re-query).

**Dynamic resolutionTrend.**

```typescript
const MAX_TREND_BUCKETS = 24;
const rangeStartMonth = new Date(startDate.getFullYear(), startDate.getMonth(), 1);
const rangeEndMonth = new Date(endDate.getFullYear(), endDate.getMonth(), 1);
const monthsSpan =
  (rangeEndMonth.getFullYear() - rangeStartMonth.getFullYear()) * 12 +
  (rangeEndMonth.getMonth() - rangeStartMonth.getMonth()) + 1;
const bucketCount = Math.min(Math.max(monthsSpan, 1), MAX_TREND_BUCKETS);
const trendMonths = Array.from(
  { length: bucketCount },
  (_, i) => new Date(rangeEndMonth.getFullYear(), rangeEndMonth.getMonth() - (bucketCount - 1 - i), 1),
);
```

Buckets are always anchored at `rangeEndMonth` and walk back — for `dateRange='all'` (startDate = epoch) the count clamps to 24 most-recent months.

**resolutionByOutcome.** Groups `closedCases` by raw `CaseStatus`, then formats via `formatStatusName`. Note: this leaves `STATUS_82_CLOSED_CONFIRMED` and `STATUS_71_AUTOCLOSED_CONFIRMED` as *separate* bars (not merged into a single "Confirmed"); the PR description phrasing "confirmed/refuted/inconclusive/autoclosed variants" reads as intentional.

**Tier count refactor.**

```diff
-    const casesOver15Days = openCasesWithAge.filter((c) => c.ageDays > 15 && c.ageDays < 30).length;
-    const casesOver30Days = openCasesWithAge.filter((c) => c.ageDays >= 30).length;
+    // Reads off the same ageBuckets() the bar/donut use ...
+    const { age16to30: casesOver15Days, age30Plus: casesOver30Days } = ageBuckets(openCasesWithAge);
```

`ageBuckets.age16to30` uses `c.ageDays > 7 && c.ageDays <= 15` (that's `age8to15`) and `c.ageDays > 15 && c.ageDays < 30` (that's `age16to30`) — verified consistent with the removed literals. Nice deduplication.

**formatStatusName.**

```diff
-    return status.replace('STATUS_', '').replace(/_/gv, ' ');
+    const [code, ...words] = status.replace('STATUS_', '').split('_');
+    const titleCased = words.map((word) => word.charAt(0).toUpperCase() + word.slice(1).toLowerCase()).join(' ');
+    return `${code} ${titleCased}`;
```

Every consumer of the reports service goes through this method (verified — 4 call sites all in `report.service.ts`), so the Case Status and Case Ageing reports emit identical status strings.

### 2. backend/src/modules/report/report.controller.ts

Swagger schema changes (docs-only, no runtime effect):

- The `dateRange` description updated to name the widgets it actually affects: `avgResolutionTime, resolutionTrend, resolutionByOutcome`, with a note that `caseTypeResolution` ignores `caseType` but is still windowed.
- Example status strings switched to title case (`"20 In Progress"`) to reflect the new formatter.
- New `resolutionByOutcome` array schema added to the response body.
- `caseTypeResolution` description updated to spell out the intentional caseType-filter exemption.
- `resolutionTrend` description updated to explain the dynamic 24-bucket window.

### 3. backend/test/report.service.spec.ts

Test-side updates track the refactor:

- `whereClauses.length toBeGreaterThanOrEqual(3) → 2` — the trend used to be its own `findMany`; now folded into `closedCases`, so at minimum only the open and closed queries fire.
- `expect(result.ageingByStatus.some((row) => row.status.includes('RETURNED')))` softened to `.toUpperCase().includes('RETURNED')` — accommodates the title-case formatter.
- Three new resolution-trend bucketing tests: `last30` (2 months), `all` (capped at 24), `today` (single-month no-op).
- New `resolutionByOutcome` shape assertion.
- New assertion that a `caseType='AML'` filter still yields a second `findMany` without `case_type`, feeding the caseType-agnostic widget.
- `formatStatusName` expectation updated to `'20 In Progress'`, plus a new multi-word title-case test.

### 4. frontend CaseAgeingStatsCards.tsx

The whole file switches from two locally-defined card components (`DaysStatsCard`, `CountStatsCard`) with hand-rolled classes to a `React.lazy(() => import('../../dashboard/components/StatsCard'))` wrapper. Four cards now (Avg. Case Age, 16-29, 30+, Avg. Resolution Time). The named `DaysStatsCard` export is removed.

### 5. frontend CaseAgeingReport.tsx

- Drops `DaysStatsCard` import and its inline instantiation for "Avg. Resolution Time".
- Wires up `resolutionByOutcome` from the hook, and renders `<ResolutionByOutcomeChart>` in the closed-throughput row.
- Moves `<CaseAgeingStatsCards>` above the "Open Backlog" section header (previously it was inside the section).

### 6. frontend ResolutionByOutcomeChart.tsx

New recharts BarChart component. Consumes `ResolutionByOutcome[]` and renders one bar per closed status. Also applies its own `formatStatusName` (`.replace('STATUS_', '').replace(/_/gu, ' ')`) — see Issue 3.

### 7. frontend CaseAgeingTable.tsx

Drops the "User ID" column header and its `<td>`. `colSpan` on the empty-state row updated from `8` → `7`. Verified: no other consumer references `investigatorId` as a visible column in this file.

### 8. frontend PieChart.tsx

```diff
-          <span className="text-gray-700">{item.label}:</span>
-          <span className="font-medium text-gray-900">
-            {(total > 0 ? (item.value / total) * 100 : 0).toFixed(1)}%
-          </span>
+          <span className="font-medium text-gray-900">{item.label}: {(total > 0 ? (item.value / total) * 100 : 0).toFixed(1)}%</span>
```

Legend label loses the muted colour it used to have (`text-gray-700`), and label+percentage share a single span — the whole `label: X%` now renders in the bolder `text-gray-900`. Verified there is only one consumer (`CaseAgeingPieChart`).

### 9. frontend types + service + tests

- `ResolutionByOutcome` interface added and appended to `CaseAgeingData`.
- `reportsService.ts` threads `resolutionByOutcome` through both the happy-path mapping and the empty error fallback (nice — no half-mapped shape).
- Component tests updated with a synchronous `StatsCard` mock (Suspense would otherwise never resolve in JSDOM); `waitFor` used for the assertions.
- `CaseAgeingReport.test.tsx` drops the `DaysStatsCard` mock.

[↑ Back to top](#pr-review-cms-262--fix-scope-case-ageing-report-to-ownedassigned-work-and-align-its-widgets-with-case-status)

---

## Code Quality Analysis

### Strengths

- **Parallel-siblings hunt clean.** All three `findMany` call sites in `getCaseAgeing` were converted to the new scope helper; no sibling left calling `applyInvestigatorScope`.
- **Single source of truth for the tier boundaries.** Deriving `casesOver15Days` / `casesOver30Days` from `ageBuckets()` instead of a second set of literal comparisons eliminates the exact "labels vs code drift" defect the 3.1 label-drift hunt looks for.
- **Explicit "why" comments** at every non-obvious decision: why `caseTypeResolution` skips the `caseType` filter, why the trend caps at 24 buckets, why `applyOwnedOrAssignedScope` is Case Ageing only, and why `formatStatusName` is now the shared formatter.
- **Response shape symmetry** across the client mapping — `resolutionByOutcome` is added to both the happy path and the empty error fallback, so consumers never see a half-populated `CaseAgeingData`.
- **Tests updated to match the new shape** rather than being loosened. The `whereClauses.forEach` still asserts the `AND` fragment on every call.

### Issues and Observations

#### Issue 1 — `resolutionTrend` still loads full-range `closedCases` even when the trend axis is capped to 24 months

**Severity: Minor (Performance)**

For `dateRange='all'`, `getDateRange` returns `startDate = new Date(0)` (1970-01-01). `closedCases` is fetched for `[1970-01-01, now]` — potentially the entire close history of the tenant — but the trend axis only surfaces the most recent 24 months (`bucketCount = min(monthsSpan, 24)`). Everything older is fetched, materialised, iterated by `avgResolutionTime` (still meaningful) and `resolutionByOutcome` (also meaningful over the full window), but *never surfaces* through the trend view. This is the acknowledged design ("shared close-anchored window"), but note it doubles as a load-multiplier on `dateRange=all`: every reload of the Case Ageing report on `all` scans the full history.

**Fix (optional):** either narrow the shared `closedCases` fetch to the *displayed* trend range as a floor (`gte: max(startDate, trendFirstMonth)`), or add a select-count-only pre-check. Given this is a report endpoint and the investigator scope narrows the set considerably, this is a Minor observation rather than a blocker.

#### Issue 2 — `caseTypeResolution` re-runs the full `findMany` for the caseType-agnostic set instead of pushing `caseType` down to an in-memory filter on `closedCases`

**Severity: Informational (Performance / Code Quality)**

When a `caseType` filter is active, the code issues a second `prisma.case.findMany` with `commonFiltersAllTypes` (same scope, no `caseType`) — that's an extra full round-trip. Since `caseType` is already selected on `closedCases`, an alternative would be to run *one* wider query (no `caseType`) for the shared closed-throughput fetch and then filter to `caseType` in memory for `avgResolutionTime` / `resolutionByOutcome`. The current shape is fine; noting it as an option because "the same closed-and-windowed query as Avg. Resolution Time and the trend" (from the PR description) is not quite true when a caseType filter is applied — under a filter, `caseTypeResolution` uses a distinct query.

#### Issue 3 — `ResolutionByOutcomeChart` re-runs `formatStatusName` on already-formatted server output

**Severity: Minor (Code Quality)**

```typescript
// ResolutionByOutcomeChart.tsx:20
const formatStatusName = (status: string): string =>
  status.replace('STATUS_', '').replace(/_/gu, ' ').trim();
```

The backend already emits `status` via `this.formatStatusName(status)` (`report.service.ts:1315`), which now produces title-cased strings like `"82 Closed Confirmed"` — no `STATUS_` prefix and no underscores remain. The client-side `formatStatusName` here is a no-op that title-cases nothing (there are no underscores to replace) and strips no prefix. Same pattern already exists in `CaseAgeingBarChart.tsx:32` (pre-existing, not introduced by this PR). Not a bug, but the new file inherits the same dead-defense.

**Fix:** either drop the client-side formatter and use `item.status` directly, or (if the intent is to defend against a hypothetical future backend that stops formatting) leave a comment noting so.

#### Issue 4 — `resolutionByOutcome` renders sibling closed statuses as separate bars (`Closed Confirmed` vs `Autoclosed Confirmed`)

**Severity: Informational (UX)**

`closedDaysByStatus` groups by the raw `CaseStatus` enum. `STATUS_82_CLOSED_CONFIRMED` and `STATUS_71_AUTOCLOSED_CONFIRMED` produce two separate bars — same for the refuted and inconclusive variants. The PR description phrases the widget as "grouped by closed status (confirmed/refuted/inconclusive/autoclosed variants)", which reads as intentional. Flagging it here so the reviewer is aware that a user asked to compare "confirmed vs refuted resolution time" will see four-to-six bars, not four.

If a merge is desired, key `closedDaysByStatus` off a coarser conclusion tag (Confirmed / Refuted / Inconclusive / Autoclosed) computed once at ingest.

#### Issue 5 — `PieChart` legend loses the muted-label visual hierarchy

**Severity: Informational (UX)**

Previously the legend rendered `label:` in `text-gray-700` and the percentage in bold `text-gray-900`, giving the eye a hierarchy (label = context, number = value). After the change, the whole entry is a single bold `text-gray-900` span — the label and percentage now compete for weight, and the intra-item colon separator is a plain glyph rather than a styled prefix. This is a shared component (`PieChart.tsx`) — the only current consumer is `CaseAgeingPieChart` but any future consumer inherits the styling change. If the intent was purely to compact the DOM, prefer keeping the two spans with their existing classes.

#### Issue 6 — Test comment says "the two matched" but the assertion is on the union of every recorded `findMany` call, which may now include the `closedWhereAllTypes` query (3+ calls under a caseType filter)

**Severity: Informational (Test Coverage / Docs)**

`report.service.spec.ts:614` (updated) now says `since a single toHaveBeenCalledWith only proves one of the two matched.` But under a `caseType` filter, `getCaseAgeing` issues *three* `findMany` calls (open, closed, closedAllTypes). The current test drives it with `{ tenantId: 'tenant-123' }` only, so two calls fire — the comment is accurate for that particular test but under-sells the guarantee. Consider updating the comment to `two or three` or asserting under both conditions.

#### Issue 7 — Removed pre-existing test `'shows N/A for User ID when the case is unassigned'` is not replaced with an equivalent for the surviving "Investigator" column

**Severity: Minor (Test Coverage)**

The dropped column had a dedicated coverage test. The replacement column (Investigator name) does have a "shows Unassigned when the case has no owner" test earlier in the file (`expect(screen.getByText('Unassigned')).toBeInTheDocument();`), so coverage isn't lost outright — this is Minor, not a gap.

#### Issue 8 — `CaseAgeingStatsCards` still exports the removed `DaysStatsCard` from the barrel via the reports feature index

**Severity: Informational (Dead code check)**

`frontend/src/features/reports/index.ts:46` still exports `CaseAgeingStats`, which is fine. The named `DaysStatsCard` export from `CaseAgeingStatsCards.tsx` is removed and the only in-repo consumer (`CaseAgeingReport.tsx`) is updated, so no imports break. Verified via `grep -rn 'DaysStatsCard'` — no stray consumers remain. Just confirming.

[↑ Back to top](#pr-review-cms-262--fix-scope-case-ageing-report-to-ownedassigned-work-and-align-its-widgets-with-case-status)

---

## Security Assessment

| Concern | Assessment |
|---------|-----------|
| Tenant / role scoping on new/modified queries | All three `findMany` in `getCaseAgeing` go through `buildCommonCaseFilters` (tenant + priority + investigator + caseType + non-container filter) and then `applyOwnedOrAssignedScope`. `applyOwnedOrAssignedScope` correctly AND-wraps the base filter around the OR clause, so tenant scoping is preserved. Verified. |
| New endpoint / new controller decorators | No new route added — only the response shape and Swagger docs changed for `GET /reports/case-ageing`. Existing guards and rate limits are inherited. |
| Unvalidated input reaching Prisma | Only `dateRange`, `caseType`, `priority`, `investigator`, `tenantId`, `requestingUserId` reach the service. All flow through `buildCommonCaseFilters` and the existing enum parsers (`parseCaseType`, `parsePriority`). No new user-controlled string reaches Prisma raw. |
| PII / secrets in responses | The details-table change *removes* the raw investigator user ID column client-side, but the field is still on the API response (`investigatorId` in `CaseAgeingDetail`) — used for export/hover fallback. Net: no new PII surface; the visual change reduces PII visibility. |
| SSRF / open-redirect / XSS | No new `fetch`, `URL`, `redirect`, `dangerouslySetInnerHTML`, or template rendering of user content introduced. |
| CI security scanners | CodeQL (actions + JS/TS), njsscan, dependency-review, hadolint, encoding-check, dco, gpg-verify — all SUCCESS at HEAD. |

**Investigator scope note (worth flagging even though it's the point of the PR).** The narrower `applyOwnedOrAssignedScope` means an investigator viewing Case Ageing no longer sees unowned DRAFT / READY_FOR_ASSIGNMENT cases. That is a *shrinkage* of what the caller sees — from a security standpoint it's a tightening, not a leak. If any downstream feature was inadvertently depending on the wider surface (e.g. exporting from Case Ageing while a case is still in the claimable pool), that would now silently drop rows. The PR description confirms this is the intended behaviour ("its numbers are meant to describe the investigator's *own* backlog").

No new security vulnerabilities introduced by this PR.

[↑ Back to top](#pr-review-cms-262--fix-scope-case-ageing-report-to-ownedassigned-work-and-align-its-widgets-with-case-status)

---

## Test Coverage

**What is tested:**

- `applyOwnedOrAssignedScope` is exercised implicitly through every existing `getCaseAgeing` test (they still pass, so the scope wrap doesn't drop rows in the happy path).
- Two-`findMany` shape is asserted on the shared abandoned-filter test.
- Resolution trend bucketing has three new cases (last30 → 2 months, all → 24-capped, today → 1 bucket).
- `resolutionByOutcome` shape is asserted (grouped-by-status, avgDays).
- `caseTypeResolution` caseType-filter isolation is asserted with a distinct `findMany` call.
- `formatStatusName` has both the basic case (`'20 In Progress'`) and a multi-word case (`'02 Ready For Assignment'`).
- Frontend stats-cards tests updated to render four cards (with the lazy `StatsCard` mocked synchronously).
- Frontend table test asserts the User-ID column is gone and the investigator name remains.

**What is not tested:**

- **The core scoping change** — that `applyOwnedOrAssignedScope` actually strips unowned DRAFT / READY_FOR_ASSIGNMENT cases from the open backlog for an investigator (compared to the wider `applyInvestigatorScope`). A regression that reverts to the old helper on any of the three call sites would go undetected — the `AND: [..., { OR: [ownerMatch, taskMatch] }]` shape is not asserted anywhere. Minor gap.
- **Cap behaviour at the boundary** — the trend test covers 2 / 24 / 1 buckets; a 25-bucket range that clamps to 24 (spanning >24 months but not 'all') would exercise the `Math.min` boundary directly. Informational.
- **`resolutionByOutcome` empty and single-status paths** — the new test uses one mocked case with `STATUS_82_CLOSED_CONFIRMED` and asserts `arrayContaining` — a completely-empty closed set path is not asserted (but is covered by the pre-existing `avgResolutionTime === null` test flowing through the same code).

Coverage does not compromise merge. Every `Major` in the Issues section has a matching test (there are no `Major` items). Non-blocking gaps noted for the record.

**Checklist state in the PR body:** Locally ✅, Development Environment ✅, Not needed ⬜, Husky ✅, Unit tests / docs ✅. Complete.

[↑ Back to top](#pr-review-cms-262--fix-scope-case-ageing-report-to-ownedassigned-work-and-align-its-widgets-with-case-status)

---

## CodeRabbit Activity

CodeRabbit *did not* review this PR — the user hit the per-developer PR-review rate limit at 12:17:37 UTC on 2026-07-17 (the same minute the PR opened). CodeRabbit's own comment:

> `@sobia-rizwan1567`, you've reached your PR review limit, so we couldn't start this review.

No findings, corroborated or otherwise. The Section 3.1 hunts (parallel-siblings, type/prop drift, label/boundary drift, guard/scope asymmetry) were run manually against the touched files to fill the gap:

- **Parallel-siblings:** all three `findMany` in `getCaseAgeing` migrated to the new helper — clean.
- **Type/prop drift:** `CaseAgeingStats.avgResolutionTime` was already `number | null`, so the fourth card's consumption of it is compatible. `CaseAgeingData` grows a `resolutionByOutcome` field wired through the service, the hook, the page, and the types file — no consumer left on the old shape.
- **Label/boundary drift:** the status-label change from ALL CAPS to title case percolates to (a) Swagger example strings — updated, (b) unit tests — updated, and (c) callers matching on the label. Verified: no non-test caller does an exact-match compare on formatted status strings. The `CaseAgeingBarChart` test file has hard-coded `'02 READY FOR ASSIGNMENT'` strings, but they are passed into the *local* `formatStatusTickLabel` unit test — they don't come from the backend, so the test does not break. Confirmed via `grep -rn 'IN PROGRESS\|READY FOR ASSIGNMENT\|CLOSED CONFIRMED'`.
- **Guard/scope asymmetry:** no new endpoints or resolvers — only a shape change to the existing `GET /reports/case-ageing`.

[↑ Back to top](#pr-review-cms-262--fix-scope-case-ageing-report-to-ownedassigned-work-and-align-its-widgets-with-case-status)

---

## Summary and Verdict

**Verdict: Approved**

This is a focused, well-motivated fix. The narrower scope helper is the right shape, is applied consistently across every sibling query in `getCaseAgeing`, and does not touch any other report. The closed-throughput consolidation reduces three separately-windowed queries to a shared window that the widgets can actually be compared against — real correctness gain. Comments explain the *why* at every non-obvious decision (why `caseTypeResolution` skips the caseType filter, why the trend caps at 24, why the tier counts read off `ageBuckets`), which the reviewer particularly appreciates in this file. Tests were updated (not loosened) alongside the code, and the frontend follows through with the shape change end-to-end.

There are no blocking items. The observations below are quality/UX/perf notes for follow-up or discussion, none of which need to gate merge. CodeRabbit didn't run, so the reviewer independently ran the Section 3.1 hunts against the diff.

### Blocking

None.

### Non-blocking but recommended

1. **`resolutionTrend` still loads full-range `closedCases`** — on `dateRange=all` the shared close-throughput fetch scans the full close history to render only the most recent 24 months of the trend. See Issue 1.
2. **`ResolutionByOutcomeChart` re-runs `formatStatusName` on already-formatted server output** — pre-existing pattern in `CaseAgeingBarChart` too, but the new file is the moment to fix it. See Issue 3.
3. **Sibling closed statuses render as separate bars in `resolutionByOutcome`** — worth confirming whether "confirmed" and "autoclosed confirmed" are meant to be one bar or two before this ships. See Issue 4.

[↑ Back to top](#pr-review-cms-262--fix-scope-case-ageing-report-to-ownedassigned-work-and-align-its-widgets-with-case-status)

---

## GitHub Review Comment

````markdown
**Approved**

Focused, well-motivated fix. `applyOwnedOrAssignedScope` is applied consistently across every sibling `findMany` in `getCaseAgeing` and does not leak to any other report. The closed-throughput consolidation into a single `updated_at`-windowed query is a real correctness win — the three widgets now genuinely share the same time window. Tests and Swagger docs are updated alongside the code, and the frontend shape change is wired end-to-end (mapping, hook, page, types) without leaving a half-populated response. CodeRabbit was rate-limited and did not review; I ran the parallel-siblings / type-drift / label-drift / guard-scope hunts manually and they're all clean.

No blocking items. A few non-blocking notes below.

---

### Non-blocking (please address in this PR if possible)

**1. `resolutionTrend` still loads full-range `closedCases` on `dateRange=all`**

`getDateRange('all')` returns `startDate = new Date(0)`, so [`backend/src/modules/report/report.service.ts:1251-1254`](backend/src/modules/report/report.service.ts#L1251-L1254) scans the entire close history of the tenant, but the trend axis at [`backend/src/modules/report/report.service.ts:1329-1337`](backend/src/modules/report/report.service.ts#L1329-L1337) only surfaces the most recent 24 months. Everything older is fetched, materialised, and iterated (correctly by `avgResolutionTime` and `resolutionByOutcome`, wastefully by the trend). Consider a `gte` floor on `closedWhere.updated_at` derived from the earliest trend bucket, or a select-count pre-check to bound the payload. This isn't a merge-blocker — the report is already investigator-scoped — but on `dateRange=all` for a large tenant this is the largest fetch in the endpoint.

**2. `ResolutionByOutcomeChart` re-runs `formatStatusName` on already-formatted server output**

In [`frontend/src/features/reports/components/ResolutionByOutcomeChart.tsx:19-21`](frontend/src/features/reports/components/ResolutionByOutcomeChart.tsx#L19-L21):

```typescript
const formatStatusName = (status: string): string =>
  status.replace('STATUS_', '').replace(/_/gu, ' ').trim();
```

The backend already emits `status` via `this.formatStatusName(status)` at [`backend/src/modules/report/report.service.ts:1315`](backend/src/modules/report/report.service.ts#L1315) — the string is already `"82 Closed Confirmed"` with no `STATUS_` prefix and no underscores. The client-side formatter is a no-op that does nothing. Either drop it and use `item.status` directly, or add a comment explaining why it exists as a defensive shim. `CaseAgeingBarChart.tsx:32` has the same pattern (pre-existing) — not part of this PR's scope to fix, just noting the new file inherits it.

**3. Sibling closed statuses render as separate bars in `resolutionByOutcome`**

At [`backend/src/modules/report/report.service.ts:1310-1317`](backend/src/modules/report/report.service.ts#L1310-L1317), `closedDaysByStatus` keys off the raw `CaseStatus` enum, so `STATUS_82_CLOSED_CONFIRMED` and `STATUS_71_AUTOCLOSED_CONFIRMED` produce two separate bars — same for the refuted and inconclusive variants. The PR description reads as intentional ("confirmed/refuted/inconclusive/autoclosed variants"), but the widget title *"Resolution Time by Outcome"* would suggest four outcomes to a user, not six-to-eight. If the intent is to merge, consider keying on a coarser conclusion tag computed once from the closed status. If the intent is truly one bar per closed status, no change needed — worth confirming before this ships.

Also flagged in the review file (I keep a longer one locally):
- `caseTypeResolution` re-runs the full `findMany` under a caseType filter — could be an in-memory filter on `closedCases`.
- The `PieChart` legend collapsed to a single bold span loses the muted-label / bold-percentage hierarchy the previous two-span layout had.

Further minor observations are recorded in the review file.
````

[↑ Back to top](#pr-review-cms-262--fix-scope-case-ageing-report-to-ownedassigned-work-and-align-its-widgets-with-case-status)
