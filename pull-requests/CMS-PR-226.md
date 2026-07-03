# PR Review: CMS #226 — fix: fixed the high priority issues

**Repo:** tazama-lf/case-management-system
**Branch:** `paysys/cms-bug-fixing` → `dev`
**Author:** ibadkhan088 (Ibad Ahmed Khan)
**Date Reviewed:** 2026-07-03
**Label:** bug
**Size:** +960 / -279 lines across 22 files
**Commits:** 8 (`6199e94`, `aff9d25`, `c0c6695`, `5b6f1b8`, `39f6f08`, `7bdef6e`, `a9734e9`, `d961a2e`)
**State:** OPEN
**Existing approvals:** None

---

## Table of Contents

- [Overview](#overview)
- [What Changed (Detailed)](#what-changed-detailed)
  - [1. alert.controller.ts — Add startDate/endDate query params](#1-alertcontrollerts--add-startdateenddate-query-params)
  - [2. alert.statistics.service.ts — Full service refactor with date filtering and search overhaul](#2-alertstatisticsservicets--full-service-refactor-with-date-filtering-and-search-overhaul)
  - [3. alert.statistics.service.spec.ts — Expanded test coverage](#3-alertstatisticsservicespects--expanded-test-coverage)
  - [4. triage.types.ts / triageservice.ts — Frontend type and service plumbing](#4-triagetypests--triageservicets--frontend-type-and-service-plumbing)
  - [5. useAlerts.ts — Server-side search and date range, stale-response guard](#5-usealertsts--server-side-search-and-date-range-stale-response-guard)
  - [6. AlertsDashboard.tsx — Loading skeleton condition and filter state fix](#6-alertsdashboardtsx--loading-skeleton-condition-and-filter-state-fix)
  - [7. AlertsSearchAndFilters.tsx — Date range validation and saved filter UX](#7-alertssearchandfiltersstsx--date-range-validation-and-saved-filter-ux)
  - [8. CaseDetailsTab.tsx — Alert detail modal link from case view](#8-casedetailstabtsx--alert-detail-modal-link-from-case-view)
  - [9. Cosmetic/minor changes across multiple files](#9-cosmeticminor-changes-across-multiple-files)
- [Code Quality Analysis](#code-quality-analysis)
  - [Strengths](#strengths)
  - [Issues and Observations](#issues-and-observations)
- [Security Assessment](#security-assessment)
- [Test Coverage](#test-coverage)
- [CodeRabbit Activity](#coderabbit-activity)
- [Summary and Verdict](#summary-and-verdict)
- [GitHub Review Comment](#github-review-comment)
- [Follow-up Review (2026-07-03)](#follow-up-review-2026-07-03)
  - [Resolution Status — All Outstanding Items](#resolution-status--all-outstanding-items)
  - [Final Verdict](#final-verdict)
  - [Final GitHub Review Comment](#final-github-review-comment)

---

## Overview

This PR addresses several high-priority issues in the CMS alerts feature. The core functional change is moving alert date-range filtering and full-text search from the client side to the server side, sending `startDate`/`endDate` ISO strings and a `search` query to the existing alert listing endpoint. The backend `AlertStatisticsService` is substantially refactored — the single 140-line `getAlertsForUser` method is decomposed into typed interfaces and private helper methods, with inverted-range validation and proper date parsing added.

A separate but related change adds a clickable "View alert details" button in the `CaseDetailsTab` that opens `AlertsDetailModal` for the row's associated alert. Several other files receive formatting-only changes (double-to-single quotes, boolean truthiness simplification, `let`→`const`).

The PR title (`fix: fixed the high priority issues`) is too generic. CodeRabbit flagged this as inconclusive in its pre-merge check.

| File | Nature of Change |
|------|-----------------|
| `backend/src/modules/alert/alert.controller.ts` | Add `startDate`/`endDate` Swagger query params and pass through to service |
| `backend/src/modules/alert/alert.statistics.service.ts` | Major refactor: typed interfaces, helper method decomposition, date range parsing, search overhaul |
| `backend/test/alert.statistics.service.spec.ts` | New tests for date range, inverted range, transaction ID search, prefix handling, partial enum matching |
| `frontend/src/features/alerts/types/triage.types.ts` | Add `startDate`/`endDate` to `AlertsFilter` |
| `frontend/src/features/alerts/services/triageservice.ts` | Append `startDate`/`endDate` to query string |
| `frontend/src/features/alerts/services/__tests__/triageservice.test.ts` | Assert URL-encoded date params |
| `frontend/src/features/alerts/hooks/useAlerts.ts` | Server-side search/date computation, stale-response guard via `useRef`, remove client-side search |
| `frontend/src/features/alerts/hooks/__tests__/useAlerts.test.ts` | Rewritten tests to assert server calls instead of client-side filtering |
| `frontend/src/features/alerts/pages/AlertsDashboard.tsx` | Skeleton condition fix (`!lastUpdated`), filter state overwrite fix |
| `frontend/src/features/alerts/pages/__tests__/AlertsDashboard.test.tsx` | New test for post-load refetch behavior |
| `frontend/src/features/alerts/components/AlertsSearchAndFilters.tsx` | Date range error state, clear filter reset, saved filter refresh after save, placeholder fix |
| `frontend/src/features/alerts/components/__tests__/AlertsSearchAndFilters.test.tsx` | Tests for saved filter reset, inverted date validation, placeholder attribute |
| `frontend/src/features/alerts/components/AlertDetails.tsx` | `.exec()` style regex |
| `frontend/src/features/alerts/components/AlertsDetailModal.tsx` | Quote style, minor formatting |
| `frontend/src/features/cases/components/view/CaseDetailsTab.tsx` | Add alert detail modal trigger |
| `frontend/src/features/cases/components/view/__tests__/CaseDetailsTab.test.tsx` | Test for modal open |
| `frontend/src/features/cases/components/TasksDetailsModal.tsx` | Boolean truthiness cleanup |
| `frontend/src/features/cases/components/ViewCaseModal.tsx` | Arrow function body formatting |
| `frontend/src/features/cases/components/modals/GenerateInvestigationReportModal.tsx` | Quote style, ESLint comment replacement |
| `frontend/src/features/cases/components/view/visualizations/alertnavigator/AlertNavigatorTab.tsx` | Arrow function formatting |
| `frontend/src/features/cases/utils/investigationUtils.tsx` | `let`→`const` for `investigatorName` |
| `frontend/src/shared/components/ErrorFallback.tsx` | Quote style |

[↑ Back to top](#pr-review-cms-226--fix-fixed-the-high-priority-issues)

---

## What Changed (Detailed)

### 1. alert.controller.ts — Add startDate/endDate query params

Two optional `@ApiQuery` decorators are added and the new `startDate`/`endDate` string parameters are destructured from `@Query()` and forwarded to the service unchanged.

```diff
+ @ApiQuery({ name: 'startDate', required: false, type: 'string', description: 'Filter alerts created on or after this ISO date/time' })
+ @ApiQuery({ name: 'endDate', required: false, type: 'string', description: 'Filter alerts created on or before this ISO date/time' })
  ...
+ @Query('startDate') startDate?: string,
+ @Query('endDate') endDate?: string,
  ...
+ startDate,
+ endDate,
```

The strings arrive as raw user input and are passed directly to the service without any pre-validation at the controller layer. Validation is intentionally delegated to `parseDateRange` in the service.

---

### 2. alert.statistics.service.ts — Full service refactor with date filtering and search overhaul

**Before:** A single 140-line `getAlertsForUser` method built the `where` clause inline with ad-hoc validation.

**After:** Three typed module-level interfaces (`GetAlertsForUserParams`, `DateRange`, `AlertsForUserResponse`) and six private helper methods decompose the logic: `validatePagination`, `validateSort`, `parseDateRange`, `buildWhereClause` (which delegates to `getReportStatusFilter`, `getPriorityFilter`, `getAlertTypeFilter`, `getDirectFilters`, `getSearchFilter`), plus `normalizeSearch`, `isDisplayAlertPrefixSearch`, `buildSearchConditions`, `addTransactionIdSearchConditions`, `getNumericSearch`, `addNumericSearchConditions`, and `addEnumSearchConditions`.

**Date range validation** (final state in commit `34431dd`):
```typescript
private parseDateRange(params: GetAlertsForUserParams): DateRange {
  const { startDate, endDate } = params;
  const parsedStartDate = startDate ? new Date(startDate) : undefined;
  if (parsedStartDate && Number.isNaN(parsedStartDate.getTime())) {
    throw new BadRequestException(`Invalid startDate: ${startDate}`);
  }
  const parsedEndDate = endDate ? new Date(endDate) : undefined;
  if (parsedEndDate && Number.isNaN(parsedEndDate.getTime())) {
    throw new BadRequestException(`Invalid endDate: ${endDate}`);
  }
  if (parsedStartDate && parsedEndDate && parsedStartDate > parsedEndDate) {
    throw new BadRequestException('startDate must be before or equal to endDate');
  }
  return { parsedStartDate, parsedEndDate };
}
```

The inverted range guard was added in response to CodeRabbit's initial finding and is present in the final commit.

**Prefix matching fix** (final state in commit `34431dd`):
```typescript
// DISPLAY_ALERT_PREFIXES = ['ALERT']
private isDisplayAlertPrefixSearch(searchString: string): boolean {
  const normalizedSearch = searchString.replace(/[-_\s]/gv, '').toUpperCase();
  return normalizedSearch !== '' && DISPLAY_ALERT_PREFIXES.some((prefix) => prefix.startsWith(normalizedSearch));
}
```

The initial commit used `DISPLAY_ALERT_PREFIX.includes(normalizedSearch)` (substring match), which allowed non-prefix substrings like `ERT` or `LER` to suppress search results. The final commit uses `prefix.startsWith(normalizedSearch)` (prefix match), which is correct. Note: the diff shown in CodeRabbit's review still contains the old `DISPLAY_ALERT_PREFIX.includes(...)` — this was corrected in the follow-up commit and confirmed correct in the final branch state.

**Search overhaul:** The prior logic searched `txtp` and `source` columns only, with a numeric fallback for `alert_id` and `case_id`. The new logic:
- Normalizes search to a string (handles numeric/boolean/bigint inputs)
- Short-circuits on empty string or display-alert-prefix-only inputs (e.g., `A`, `AL`, `ALE`, `ALERT`, `ALERT-`)
- Strips the `ALERT-` prefix and routes pure numeric results to `alert_id` exact match only
- Searches `txtp`, `source`, two transaction JSON paths (`FIToFIPmtSts.GrpHdr.MsgId`, `FIToFICstmrCdt.GrpHdr.MsgId`), plus partial enum matching for `Priority` and `CaseType` (minimum 3 chars)
- **Notable removal:** `case_id` is no longer included in numeric search conditions

---

### 3. alert.statistics.service.spec.ts — Expanded test coverage

New describe blocks and test cases:
- Invalid date string → `BadRequestException`
- `startDate > endDate` → `BadRequestException` (the inverted range guard)
- Date range filter applied to `created_at` in Prisma where clause
- Transaction ID exact match via both JSON paths
- Numeric-only `alert_id` search (without `case_id`)
- `ALERT-123` prefix search routing to `alert_id: { equals: 123 }`
- Prefix-only inputs (`A`, `ALE`, `ALERT`, `ALERT-`) → no `OR` filter applied
- Non-prefix substrings (`ERT`) are treated as normal text search
- Partial priority match (`cri` → `CRITICAL`) with 3-char minimum
- Partial alert type match (`fra` → `FRAUD`, `FRAUD_AND_AML`) with 3-char minimum
- 2-char search (`cr`) does not trigger partial enum matching

---

### 4. triage.types.ts / triageservice.ts — Frontend type and service plumbing

`AlertsFilter` gains two optional fields:
```diff
+ startDate?: string;
+ endDate?: string;
```

`triageservice.ts` appends them to the query string:
```diff
+ if (filters.startDate) params.append('startDate', filters.startDate);
+ if (filters.endDate) params.append('endDate', filters.endDate);
```

The test asserts URL-encoding: `startDate=2024-01-01T00%3A00%3A00.000Z`.

---

### 5. useAlerts.ts — Server-side search and date range, stale-response guard

**Stale-response guard:**
```typescript
const latestFetchId = useRef(0);

const fetchAlerts = useCallback(async () => {
  const fetchId = latestFetchId.current + 1;
  latestFetchId.current = fetchId;
  ...
  const response = await triageService.getAlerts(filters);
  if (fetchId !== latestFetchId.current) {
    return; // stale response, discard
  }
  ...
}, [...]);
```

This prevents race conditions where a slow unflitered request resolves after a faster filtered one, reverting the UI state.

**Custom date parsing (final state, `parseLocalDateInput`):**
```typescript
const parseLocalDateInput = (dateInput: string): Date => {
  const [year, month, day] = dateInput.split('-').map(Number);
  return new Date(year, month - MONTH_INDEX_OFFSET, day);
};
```

This correctly uses the `Date(year, month, day)` constructor (local time) instead of `new Date('YYYY-MM-DD')` (UTC midnight), avoiding a timezone off-by-one day bug.

**Removed:** Client-side `searchFilteredAlerts` memo, client-side pagination override for search, client-side date filtering. `paginatedAlerts` is now simply `state.filteredAlerts`.

**`setFilters` now merges only the provided fields** (`setFilters({ [key]: value })` in `AlertsDashboard`) — the diff shows `setFilters` in `useAlerts.ts` uses `dispatch({ type: 'SET_FILTERS', payload: filters })` which merges via the reducer.

---

### 6. AlertsDashboard.tsx — Loading skeleton condition and filter state fix

```diff
- if (loading && alerts.length === 0) {
+ if (loading && alerts.length === 0 && !lastUpdated) {
```

Previously, any post-load re-fetch (e.g., applying a filter) would re-render the full skeleton because `alerts.length` temporarily drops to 0 during the transition. The `!lastUpdated` guard ensures the skeleton only shows on the initial load, and filter controls remain mounted during subsequent fetches.

Filter change now overwrites only the changed field:
```diff
- setFilters({ ...filters, [key]: value });
+ setFilters({ [key]: value });
```

The `useReducer` merges fields, so spreading is redundant — this simplification is correct.

---

### 7. AlertsSearchAndFilters.tsx — Date range validation and saved filter UX

- **Client-side date range error state:** When the user sets `endDate < startDate`, an error toast fires and `onCustomDateRangeChange` is blocked. The error message is displayed inline.
- **`handleClearFilters`:** Wraps `onClearFilters` to also reset `selectedSavedFilterId` and `dateRangeError` local state.
- **Saved filter refresh:** After `filterService.createFilter`, `fetchSavedFilters()` is called so the newly saved filter appears in the dropdown without a page reload.
- **Placeholder fix:** The "Select a saved filter" `<option>` now has `disabled hidden` attributes, making it a proper placeholder that cannot be re-selected.
- **Label fallback:** `??` replaced by `||` for filter display name fallback — this changes behavior for empty-string values (empty string is falsy with `||`, not with `??`). For the current filter fields (`alertType`, `priority`, `source`, `timeRange`) this is a meaningful difference: a filter explicitly saved with an empty string will now show `ALL TYPES` etc. rather than the empty string. This is likely intentional for display purposes.

---

### 8. CaseDetailsTab.tsx — Alert detail modal link from case view

```diff
- <div className="font-medium text-gray-900">{row.alertId}</div>
+ <button
+   type="button"
+   onClick={() => { setIsAlertModalOpen(true); }}
+   className="inline-flex items-center gap-1 font-medium text-gray-900 hover:text-blue-800 hover:underline focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-offset-1 rounded"
+   title="View alert details"
+ >
+   <span>{row.alertId}</span>
+   <ArrowTopRightOnSquareIcon className="h-4 w-4" />
+ </button>
```

`AlertsDetailModal` is mounted below the case detail content, controlled by `isAlertModalOpen` state. The `alertId` passed is `row.alertId ?? null`.

**Known gap (CodeRabbit, not resolved):** When switching between cases/sub-cases inside the same `ViewCaseModal` session, `isAlertModalOpen` is not reset in the `useEffect` that resets `latestReports` and `viewingId` on `row.id` change. This can leave the alert modal open for a different alert than the one the user intends to view.

---

### 9. Cosmetic/minor changes across multiple files

- `AlertDetails.tsx`: `actionPerformed.match(...)` → `/regex/.exec(actionPerformed)` (static analysis preferred form)
- `AlertsDetailModal.tsx`: double-to-single quotes throughout, `getAlertStatusColor` refactored to concise ternary
- `TasksDetailsModal.tsx`: `=== false`/`=== true` → `!value`/`value` truthiness checks
- `ViewCaseModal.tsx`, `AlertNavigatorTab.tsx`: Arrow function bodies wrapped in braces
- `GenerateInvestigationReportModal.tsx`: Quote style; ESLint disable comment replaced with blank line (the `// eslint-disable-next-line` was removed but the suppressed code remains — minor)
- `investigationUtils.tsx`: `let investigatorName` → `const investigatorName` (variable is never reassigned)
- `ErrorFallback.tsx`: Quote style

[↑ Back to top](#pr-review-cms-226--fix-fixed-the-high-priority-issues)

---

## Code Quality Analysis

### Strengths

- **Decomposition is well done.** The `AlertStatisticsService` refactor converts a single monolith into focused private methods with clear single responsibilities. Typed interfaces (`GetAlertsForUserParams`, `AlertsForUserResponse`, `DateRange`) replace inline anonymous types, reducing duplication between the method signature and the controller.
- **Stale-response guard is correct.** The `useRef`-based fetch ID pattern is a standard and reliable solution to the race condition where slower unfiltered requests can overwrite faster filtered results.
- **`parseLocalDateInput` is the correct fix.** Using the `Date(year, month, day)` constructor instead of `new Date('YYYY-MM-DD')` eliminates the UTC midnight timezone shift bug.
- **Inverted date range rejection is properly implemented.** Both the backend (`parseDateRange` guard) and frontend (`handleCustomDateRangeChange` error state) now prevent the impossible `startDate > endDate` filter from reaching the database.
- **Test coverage breadth.** The backend spec gains ~9 new test cases covering edge cases (invalid dates, inverted ranges, prefix suppression, partial enum match, transaction ID search). Frontend tests are updated from client-side filter assertions to server-call assertions, which is the correct direction after moving filtering server-side.
- **Prefix matching logic is correct.** `DISPLAY_ALERT_PREFIXES.some(prefix => prefix.startsWith(normalizedSearch))` correctly suppresses results only when the user is mid-typing the alert ID prefix, not for arbitrary substrings.

### Issues and Observations

#### Issue 1 — `isAlertModalOpen` not reset on `row.id` change

**Severity: Minor (Bug)**

In `frontend/src/features/cases/components/view/CaseDetailsTab.tsx`, the existing cleanup effect resets `latestReports` and `viewingId` when `row.id` changes, but `isAlertModalOpen` is not included:

```typescript
// Current code (around line 101)
useEffect(() => {
  setLatestReports({});
  setViewingId(null);
  // isAlertModalOpen is NOT reset here
}, [row.id]);
```

When a user switches between cases within the same `ViewCaseModal` session (without a full remount), the alert modal can remain open showing the previous case's alert. The fix is one line:

```diff
  useEffect(() => {
    setLatestReports({});
    setViewingId(null);
+   setIsAlertModalOpen(false);
  }, [row.id]);
```

CodeRabbit flagged this. The test added in this PR (`CaseDetailsTab.test.tsx`) only covers the open path, not the close path or the row-change reset, so the gap will not be caught by CI.

---

#### Issue 2 — Modal close path untested

**Severity: Minor (Test Coverage)**

The `AlertsDetailModal` mock in `CaseDetailsTab.test.tsx` renders the modal when open, but never exercises the `onClose` handler. If `setIsAlertModalOpen(false)` is wired incorrectly, no test would catch it. This gap is compounded by Issue 1.

---

#### Issue 3 — `case_id` removed from numeric search without documentation

**Severity: Informational (Behavior Change)**

The old search code included:
```typescript
searchConditions.push({ case_id: { equals: Number(search) } });
```

The new code removes `case_id` from numeric search entirely. This is a behavioral change — users who previously searched by `case_id` number will no longer find results. The PR description does not call this out, and the test comment only says "should search by numeric alert_id" (the old test was renamed from "should search by numeric alert_id and case_id"). This may be intentional (alerts have a `case_id` but it is not a user-facing ID worth searching), but it should be explicitly confirmed.

---

#### Issue 4 — CI encoding check failures

**Severity: Minor (CI)**

Two CI jobs (`Encoding Check / 0_encoding-check` and `Encoding Check / encoding-check`) are failing. The failure output shows a UTF-16 BOM detection script running — at least one file committed in this PR may have been saved with a non-UTF-8 encoding or BOM. This will block the CI pipeline and should be resolved before merge. The CodeRabbit second pass (reviewing commit `34431dd`) noted this as a timeout rather than a full scan, but the failures are visible in the check run annotations.

---

#### Issue 5 — Test assertions for date range still use `expect.any(String)` for preset ranges

**Severity: Informational (Test Coverage)**

In `useAlerts.test.ts`, the test for `sends today time range dates to the server` asserts:
```typescript
expect(mockService.getAlerts).toHaveBeenCalledWith(
  expect.objectContaining({
    startDate: expect.any(String),
    endDate: expect.any(String),
  }),
);
```

This confirms that strings are sent but does not protect against off-by-one bugs in `getDateRangeForFilter` for preset ranges. CodeRabbit flagged this (outside diff range, so not posted inline). The custom range test does assert exact ISO strings by using `new Date(2024, 0, 1, 0, 0, 0, 0).toISOString()`, which is the correct approach.

[↑ Back to top](#pr-review-cms-226--fix-fixed-the-high-priority-issues)

---

## Security Assessment

| Concern | Assessment |
|---------|-----------|
| SQL/Prisma injection via `startDate`/`endDate` | Not a risk. Dates are parsed through `new Date(input)` and passed as `Date` objects to Prisma's typed where clause. Prisma parameterizes all values. Invalid strings throw `BadRequestException` before any query runs. |
| Unvalidated `search` input reaching the database | The `normalizeSearch` method coerces to string. All conditions use Prisma `contains`, `equals`, or JSON path operators — no raw SQL. No injection risk. |
| Timing/race condition via stale fetch | Addressed by the `useRef` fetch ID guard. A slow unfiltered response is discarded when a newer fetch has completed. |
| Regex denial of service | The regex `/[-_\s]/gv` in `isDisplayAlertPrefixSearch` operates on a trimmed, short string (the user's search input). No unbounded repetition. Not a risk. |
| XSS via `alertId` rendered in modal | `row.alertId` is rendered as `{row.alertId}` in JSX — React escapes it. Not a risk. |
| Command injection (OpenGrep flag on AlertDetails.tsx) | CodeRabbit's static analyzer flagged line 363 of `AlertDetails.tsx` for dynamic command injection via `exec`. This file receives a formatting-only change in this PR (`/.exec()` call style). The underlying concern (if genuine) is pre-existing and out of scope. |

No new security vulnerabilities introduced by this PR.

[↑ Back to top](#pr-review-cms-226--fix-fixed-the-high-priority-issues)

---

## Test Coverage

**Backend:**
- `alert.statistics.service.spec.ts` is substantially expanded. Date range validation, inverted range rejection, date range filter in the where clause, transaction ID search, ALERT-prefix routing, partial enum search, and minimum-length guard are all covered with dedicated test cases.
- The updated tests correctly reflect the new search behavior (no `case_id` in numeric search, correct OR condition count of 5 for numeric input).

**Frontend:**
- `useAlerts.test.ts`: Tests are rewritten from client-side filter assertions to server-call assertions. The stale-response race condition is explicitly tested with a deferred-promise approach, which is well-constructed.
- `AlertsSearchAndFilters.test.tsx`: New tests for inverted date range error message, saved filter reset on clear, and `fetchSavedFilters` called after save.
- `AlertsDashboard.test.tsx`: New test for the `!lastUpdated` skeleton guard — correctly asserts that filter controls remain mounted during refetches.
- `CaseDetailsTab.test.tsx`: Modal open path is covered; close path and row-change reset are not.
- `triageservice.test.ts`: URL-encoded date params are asserted.

**Gaps:**
- `CaseDetailsTab` close path and `row.id`-change reset are untested (see Issues 1 and 2).
- Preset time range tests (`today`, `yesterday`, `thisWeek`, etc.) only assert `expect.any(String)` for dates rather than concrete values (see Issue 5).
- The PR checklist state is not visible; no coverage screenshots or CI evidence attached given the encoding check failures.

[↑ Back to top](#pr-review-cms-226--fix-fixed-the-high-priority-issues)

---

## CodeRabbit Activity

### Pass 1 — Initial review of commit `c3284190`

**Reviewed:** `fe21ef7` → `c3284190`
**Findings:** 3 actionable comments (2 major inline, 2 minor outside-diff, 2 nitpick)

| Finding | Severity | Status |
|---------|----------|--------|
| `parseDateRange` does not reject `startDate > endDate` | Minor → Major | ✅ Resolved — guard added in `34431dd` |
| `isDisplayAlertPrefixSearch` uses substring match (`includes`) instead of prefix match (`startsWith`) | Major | ✅ Resolved — fixed to `DISPLAY_ALERT_PREFIXES.some(p => p.startsWith(...))` in `34431dd` |
| `useAlerts.ts` custom date parsing uses `new Date('YYYY-MM-DD')` (UTC shift) | Major | ✅ Resolved — `parseLocalDateInput` introduced in `34431dd` |
| `useAlerts.test.ts` date tests assert only `expect.any(String)` for preset ranges | Minor | ❌ Not resolved |
| `AlertsDashboard.test.tsx` skeleton test only checks page title, not the skeleton component | Minor | ❌ Not resolved — new test added verifies the opposite branch (`!lastUpdated`) but the original skeleton assertion still only checks the title |
| `CaseDetailsTab.test.tsx` modal close path not tested | Trivial | ❌ Not resolved |
| `CaseDetailsTab.tsx` `isAlertModalOpen` not reset on `row.id` change | Trivial | ❌ Not resolved |

### Pass 2 — Follow-up scan of commit `34431ddc` (review failed to post)

CodeRabbit's second pass failed to post inline comments due to a GitHub API timeout. The summary indicates 22 files were re-processed and 13 were skipped as similar to previous changes. The failing CI check (`Encoding Check`) was flagged.

[↑ Back to top](#pr-review-cms-226--fix-fixed-the-high-priority-issues)

---

## Summary and Verdict

**Verdict: Changes Requested**

The core functional changes in this PR are well-executed. Moving date filtering and text search server-side is the right architectural direction, the `AlertStatisticsService` refactor significantly improves readability and testability, and the three major CodeRabbit findings (inverted date range, substring vs prefix matching, UTC timezone shift) were all resolved in the follow-up commit. The stale-response guard is correctly implemented.

However, two issues require attention before merge. The `isAlertModalOpen` state is not reset when switching cases within the same `ViewCaseModal` session, which is a real UX bug that the existing test does not catch. The CI encoding check failures must be resolved regardless — if any committed file contains a BOM or non-UTF-8 encoding, it needs to be re-saved.

### Blocking

1. **CI encoding check failures** — Two `Encoding Check` jobs are failing. At least one file in this branch has a non-UTF-8 encoding or BOM. Re-save the offending file(s) as UTF-8 without BOM and push.
2. **`isAlertModalOpen` not reset on `row.id` change** — Switching between cases inside the same `ViewCaseModal` can leave the alert modal open for the wrong alert. Add `setIsAlertModalOpen(false)` to the `useEffect(() => { ... }, [row.id])` cleanup in `CaseDetailsTab.tsx`.

### Non-blocking but recommended

3. **Add modal close-path test** — The `CaseDetailsTab.test.tsx` mock only tests the open path. Add a test that triggers `onClose` and asserts the modal is removed.
4. **Assert concrete date values for preset time ranges** — The `useAlerts.test.ts` tests for `today`, `yesterday`, etc. use `expect.any(String)`. Freeze time and assert the exact ISO boundary strings to protect against off-by-one timezone regressions.
5. **Document the `case_id` removal from numeric search** — The removal is a behavior change. Confirm it is intentional and note it in the PR description so reviewers and QA can validate it.

[↑ Back to top](#pr-review-cms-226--fix-fixed-the-high-priority-issues)

---

## GitHub Review Comment

````markdown
**Changes Requested**

The core changes (server-side date/search filtering, service refactor, stale-response guard, timezone fix) are solid. Two blocking items need to be addressed before merge.

---

### Blocking

**1. CI encoding check failures**

Two `Encoding Check` CI jobs are failing. At least one file in this branch has a non-UTF-8 encoding or UTF-16 BOM. Find the offending file(s), re-save them as UTF-8 without BOM, and push.

---

**2. `isAlertModalOpen` not reset when switching cases (`CaseDetailsTab.tsx`)**

The `useEffect` that resets `latestReports` and `viewingId` when `row.id` changes does not reset `isAlertModalOpen`. If a user switches between cases inside the same `ViewCaseModal` session (without a full remount), the alert modal will stay open and display the newly selected case's alert — but the user didn't click to open it for that case.

```diff
  useEffect(() => {
    setLatestReports({});
    setViewingId(null);
+   setIsAlertModalOpen(false);
  }, [row.id]);
```

---

### Non-blocking (please address in this PR if possible)

**3. Add modal close-path test (`CaseDetailsTab.test.tsx`)**

The `AlertsDetailModal` mock only covers the open state. Extend the mock to expose a close trigger (call `onClose`) and add an assertion that the modal is removed from the DOM.

**4. Assert concrete date values for preset time ranges (`useAlerts.test.ts`)**

The tests for `today` and `sends today time range dates to the server` use `expect.any(String)` for `startDate`/`endDate`. These will pass even if `getDateRangeForFilter` returns the wrong day due to a timezone issue. Freeze time (e.g., `vi.useFakeTimers()`) and assert the exact ISO strings.

**5. Document the `case_id` removal from numeric search**

The old search included `{ case_id: { equals: Number(search) } }` in numeric results; the new code removes it. If users currently search alerts by `case_id`, this is a regression. Please confirm in the PR description whether this is intentional.
````

[↑ Back to top](#pr-review-cms-226--fix-fixed-the-high-priority-issues)

---
---
---

## Follow-up Review (2026-07-03)

**Reviewed commit:** `d961a2e` — *"fix: fixed the code rabbit issues"* (2026-07-03T12:43Z)
**Reviewed against:** CHANGES_REQUESTED on commit `34431dd` by `ahmad-paysys` (2026-07-03T07:23Z)
**Developer response:** Pushed one commit that touches only `frontend/src/features/cases/components/view/CaseDetailsTab.tsx` (+1/-0), applying the exact `setIsAlertModalOpen(false)` addition suggested in the review diff.

### Resolution Status — All Outstanding Items

### Item 1 — CI encoding check failures

**Status: RESOLVED**

`encoding-check / encoding-check` now reports `pass` on the latest commit. All 14 CI checks are green (`Analyze`, `CodeQL`, `dco-check`, `dependency-review`, `dockerfile-linter`, `encoding-check`, `gpg-verify`, `njsscan`, `node-ci / check style`, `node-ci / check tests`, `node-ci / run build`, `nodejsscan`, `conventional-commits`, `CodeRabbit`).

### Item 2 — `isAlertModalOpen` not reset on `row.id` change

**Status: RESOLVED**

The follow-up commit `d961a2e` applies the exact diff proposed in the review:

```diff
   useEffect(() => {
     setLatestReports({});
     setViewingId(null);
+    setIsAlertModalOpen(false);
   }, [row.id]);
```

Verified in `frontend/src/features/cases/components/view/CaseDetailsTab.tsx` around line 112–116.

### Item 3 — Add modal close-path test (`CaseDetailsTab.test.tsx`)

**Status: NOT RESOLVED** (non-blocking)

No changes to `CaseDetailsTab.test.tsx` in the follow-up commit. The mock still only exercises the open path.

### Item 4 — Assert concrete date values for preset time ranges (`useAlerts.test.ts`)

**Status: NOT RESOLVED** (non-blocking)

No changes to `useAlerts.test.ts` in the follow-up commit. Preset-range tests still assert `expect.any(String)`.

### Item 5 — Document the `case_id` removal from numeric search

**Status: NOT RESOLVED** (non-blocking)

No update to the PR description. The behavior change is still undocumented.

| # | Item | Status |
|---|------|--------|
| 1 | CI encoding check failures | ✅ Resolved |
| 2 | `isAlertModalOpen` not reset on `row.id` change | ✅ Resolved |
| 3 | Add modal close-path test | ❌ Not resolved (non-blocking) |
| 4 | Assert concrete date values for preset time ranges | ❌ Not resolved (non-blocking) |
| 5 | Document the `case_id` removal from numeric search | ❌ Not resolved (non-blocking) |

### Final Verdict

**Verdict: Approve with minor cleanup requested**

Both blocking items from the CHANGES_REQUESTED review are resolved. The encoding check passes and the `isAlertModalOpen` reset is in place with the exact fix suggested. All CI checks are green.

The three non-blocking items remain open. They are quality-of-test and documentation gaps rather than functional defects, so they do not block merge — but items 3 and 4 in particular are worth addressing in a follow-up so the test suite protects the new behavior against timezone regressions and modal close semantics.

[↑ Back to top](#pr-review-cms-226--fix-fixed-the-high-priority-issues)

---

## Final GitHub Review Comment

````markdown
**Approve with minor cleanup requested**

Both blocking items from the previous review are resolved — encoding-check is green and `setIsAlertModalOpen(false)` is now reset on `row.id` change in `CaseDetailsTab.tsx`. All CI checks pass. Approving for merge; the three non-blocking items below are worth picking up in a small follow-up PR but do not block this one.

---

### Non-blocking (nice to have in a follow-up)

**1. Add modal close-path test (`CaseDetailsTab.test.tsx`)**

The `AlertsDetailModal` mock only exercises the open path. Extend the mock to expose a close trigger (call `onClose`) and add an assertion that the modal is removed from the DOM.

**2. Assert concrete date values for preset time ranges (`useAlerts.test.ts`)**

The tests for the `today`/preset time-range paths still use `expect.any(String)` for `startDate`/`endDate`. Freeze time with `vi.useFakeTimers()` and assert the exact ISO boundary strings to protect against off-by-one timezone regressions in `getDateRangeForFilter`.

**3. Document the `case_id` removal from numeric search**

The old numeric search included `{ case_id: { equals: Number(search) } }`; the new implementation removes it. Please add a note to the PR description confirming this is intentional so QA can validate the behavior change.
````

[↑ Back to top](#pr-review-cms-226--fix-fixed-the-high-priority-issues)
