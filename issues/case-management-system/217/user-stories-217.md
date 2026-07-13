# User Stories — Issue #217: Case Ageing Report

Track A is one atomic story. Track B is broken into discrete widget-scope stories that must land together in a single coordinated PR.

---

## US-217-01 — Investigator ageing scope + abandoned exclusion (Track A)

**Body**
The Case Ageing Report's `getCaseAgeing` method uses an inline four-arm OR clause — copy-pasted into three query builders — to scope an investigator's view. Two of the four arms match the tenant-wide claimable pool (`case_owner_user_id: null`, `status: STATUS_02_READY_FOR_ASSIGNMENT`), so an investigator's Avg Case Age, `>15` / `>30` counts, distribution pie, by-status bars, and details table are inflated by shared work they do not own. Meanwhile the main `findMany` applies no status filter, so `STATUS_99_ABANDONED` cases sit in every widget as dead-drafts-as-live-work. This story extracts a shared helper that keeps only two ownership arms (owned by me + task assigned to me) and adds a query-level exclusion for `STATUS_99_ABANDONED`.

**Scope**
- Track: A
- Files: `backend/src/modules/report/report.service.ts` (and its test file if present).
- Change: introduce private helper `applyAgeingInvestigatorScope`; replace three inline OR clauses; add `status: { not: STATUS_99_ABANDONED }` to the shared base filters.
- Migration required: No
- Frontend changes: No

**Acceptance Criteria**
- [ ] `applyAgeingInvestigatorScope(baseFilters, requestingUserId?)` exists on `ReportsService`.
- [ ] The helper returns `baseFilters` unchanged when `requestingUserId` is undefined.
- [ ] When `requestingUserId` is defined, the helper wraps `baseFilters` in an `AND` with a two-arm `OR`: `case_owner_user_id = requestingUserId` and `tasks.some.assigned_user_id = requestingUserId`.
- [ ] `getCaseAgeing` calls the helper in three places (shared `findMany`, per-type resolution, recent-closed trend); no inline OR clause remains.
- [ ] `getCaseAgeing`'s shared `baseFilters` include `status: { not: CaseStatus.STATUS_99_ABANDONED }`.
- [ ] Unit tests cover: investigator sees only owned + task-assigned cases; supervisor sees tenant-wide; abandoned cases absent for both.

**Testing**
- Backend unit tests as above.
- Manual: log in as investigator, verify the details table shows only owned + task-assigned cases; log in as supervisor, verify no abandoned cases in any widget.

---

## US-217-02 — Split into open-snapshot and closed-in-window datasets (Track B)

**Body**
`getCaseAgeing` today runs every widget against one unbounded dataset, so `dateRange` is inert for the core query, and open and closed cases share the same widgets. Split into two lifecycle-scoped datasets: an open-snapshot (Avg Case Age, `>15` / `>30`, by-status bar, distribution pie, details table) and a closed-in-window (Avg Resolution Time, trend, case-type bar) with `dateRange` bound to the latter.

**Scope**
- Track: B
- Files: `backend/src/modules/report/report.service.ts`, `backend/src/modules/report/report.controller.ts` (swagger schema).
- Change: introduce `buildOpenSnapshotWhere` and `buildClosedInWindowWhere` private builders; wire `dateRange` through the controller to the closed-in-window dataset.
- Migration required: No
- Frontend changes: No (payload shape unchanged for this story)

**Acceptance Criteria**
- [ ] Open snapshot excludes closed and abandoned statuses at the query level.
- [ ] Closed-in-window applies `updated_at` between `windowStart` and `windowEnd` derived from `dateRange`.
- [ ] The shared `findMany` no longer feeds the three closed-widget outputs.
- [ ] Supervisor / admin numbers for open-snapshot widgets are unchanged in a tenant with no abandoned cases; closed-widget numbers shift only when `dateRange` is picked.

**Testing**
- Unit tests per builder.
- Manual: change `dateRange`; verify only the three closed widgets shift.

---

## US-217-03 — Shared age-band helper + Hamilton rounding (Track B)

**Body**
The four age bands are hard-coded in the by-status bar and re-hard-coded in the distribution pie, with a `>` vs `>=` boundary inconsistency between the bar buckets and the `>30` card. Introduce a single band definition and a Hamilton largest-remainder rounding helper for the pie's percentages.

**Scope**
- Track: B
- Files: `backend/src/modules/report/report.service.ts`.
- Change: `AGE_BANDS` constant; `bucketAge(ageDays)` helper; `hamiltonPercentages(counts)` helper.
- Migration required: No
- Frontend changes: No

**Acceptance Criteria**
- [ ] One boundary convention across all four bands and the two stat cards.
- [ ] Pie's four percentages sum to exactly 100 in every non-empty case.
- [ ] Unit tests: `hamiltonPercentages([1,1,1,1])` returns `[25,25,25,25]`; `hamiltonPercentages([1,2,3])` sums to 100; empty counts → `[0,0,0,0]`.

**Testing**
- Unit tests as above.

---

## US-217-04 — Non-overlapping `>15` / `>30` cards (Track B)

**Body**
Every case in `>30` is also in `>15` today. Either make them non-overlapping (`16-30` and `30+`) or label them as cumulative. Ship as `16-30` and `30+` unless product decides otherwise.

**Scope**
- Track: B
- Files: `backend/src/modules/report/report.service.ts`, `frontend/src/features/reports/components/CaseAgeingStatsCards.tsx`.
- Change: two-tier count on the backend; card labels updated on the frontend.
- Migration required: No
- Frontend changes: Yes

**Acceptance Criteria**
- [ ] The `casesOver15Days` field is retired or renamed to `cases16to30`.
- [ ] Cards labelled to match.
- [ ] Sum of `cases16to30 + cases30Plus` equals the count of open cases older than 15 days.

**Testing**
- Manual: seed one case aged 20 days and one aged 40 days. First card = 1, second card = 1.

---

## US-217-05 — `null` empty averages with `N/A` rendering (Track B)

**Body**
`avgCaseAge` and `avgResolutionTime` collapse to `0` for empty populations today, indistinguishable from a real zero-day mean. Return `null` on the wire; render `N/A` on the card.

**Scope**
- Track: B
- Files: `backend/src/modules/report/report.service.ts`, `frontend/src/features/reports/types/reports.types.ts`, `frontend/src/features/reports/components/CaseAgeingStatsCards.tsx`.
- Change: return `null` from the service; wire type `number | null`; card renders `stats.avgCaseAge ?? 'N/A'`.
- Migration required: No
- Frontend changes: Yes

**Acceptance Criteria**
- [ ] Service returns `null` for empty population on both averages.
- [ ] Wire type is `number | null`.
- [ ] Card renders `N/A` (or `-`) for `null`.

**Testing**
- Manual: in an empty tenant, both cards render `N/A`.

---

## US-217-06 — Monthly-median resolution trend with IQR band (Track B)

**Body**
Replace the per-close-day scatter (currently 6-month window, one row per closed case, day-formatted string) with a monthly aggregation: median + P25 + P75 + `n` per bucket, computed backend-side from raw `resolutionDays`. Frontend renders as a continuous temporal axis via recharts `ComposedChart`.

**Scope**
- Track: B
- Files: `backend/src/modules/report/report.service.ts`, `frontend/src/features/reports/types/reports.types.ts`, `frontend/src/features/reports/components/ResolutionTimeTrendChart.tsx`.
- Change: new payload shape `{ bucketStart, median, p25, p75, n }`; `ComposedChart` with `Area` for IQR and `Line` for median.
- Migration required: No
- Frontend changes: Yes

**Acceptance Criteria**
- [ ] Backend emits one row per month across the window (empty months included with `n = 0`).
- [ ] Median, P25, P75 computed from raw per-case `resolutionDays` in each bucket (no average-of-averages).
- [ ] Chart uses a continuous temporal axis.
- [ ] IQR band visible around the median line.

**Testing**
- Unit tests: given seeded closes across three months, the returned array has one entry per month with correct median/P25/P75.

---

## US-217-07 — Single grouped case-type aggregation + companion by-outcome bar (Track B)

**Body**
Replace the per-type `Promise.all` of N queries with a single grouped aggregation on the closed-in-window dataset. Add a companion "Resolution Time by Outcome" grouped bar.

**Scope**
- Track: B
- Files: `backend/src/modules/report/report.service.ts`, `frontend/src/features/reports/components/CaseTypeResolutionChart.tsx`.
- Change: `groupBy: ['case_type']` + one aggregation call; new "by outcome" component.
- Migration required: No
- Frontend changes: Yes

**Acceptance Criteria**
- [ ] Only one Prisma call for the case-type bar.
- [ ] By-outcome sibling bar renders using the same close-anchored window.

**Testing**
- Backend: assert only one Prisma call in the case-type path.
- Manual: both bars visible on the page.

---

## US-217-08 — Stop backend pre-formatting; presentation on client (Track B)

**Body**
`caseDetails.status` today is ALL-CAPS via `formatStatusName`; `createdDate` is server-formatted as `en-US`; `investigator` is a duplicated projection of `case_owner_user_id`. Emit the raw enum, ISO 8601 date, and a single nullable `ownerUserId`; format on the client via the shared code-prefixed title-case helper and `getAssigneeFullName`.

**Scope**
- Track: B
- Files: `backend/src/modules/report/report.service.ts`, `frontend/src/features/reports/types/reports.types.ts`, `frontend/src/features/reports/components/CaseAgeingTable.tsx`.
- Change: payload shape update; client-side status + date + owner formatting; drop raw "User ID" column.
- Migration required: No
- Frontend changes: Yes

**Acceptance Criteria**
- [ ] `caseDetails.status` is the raw `CaseStatus` enum on the wire.
- [ ] `caseDetails.createdDate` is ISO 8601 on the wire.
- [ ] `caseDetails.ownerUserId` is `string | null`; `caseDetails.investigator` no longer present on the wire.
- [ ] Table renders `20 In Progress`-style labels.
- [ ] Client-side date localisation used.
- [ ] "User ID" column dropped; "Investigator" column resolves the name.

**Testing**
- Snapshot / render tests on the table component.
- Manual: switch client locale; date column reformats without a backend round-trip.

---

## US-217-09 — Horizontal, 100%-stacked, seeded by-status bar (Track B)

**Body**
Today the by-status bar's x-axis is vertical (ALL-CAPS labels overflow), the stack order does not match the legend direction, and statuses with zero rows silently disappear. Switch to a horizontal 100% stacked bar seeded from the canonical open-status set, sorted by status code, with the green→amber→orange→red severity ramp and explicit segment order (oldest on top).

**Scope**
- Track: B
- Files: `frontend/src/features/reports/components/CaseAgeingBarChart.tsx`.
- Change: `layout="vertical"` in recharts, seeded status list, code-prefixed title-case labels, deterministic band order, tooltip absolute counts.
- Migration required: No
- Frontend changes: Yes

**Acceptance Criteria**
- [ ] All in-scope open statuses render as labelled bars (empty bars visible if count is zero).
- [ ] Axis sorted by status code.
- [ ] 100% stacked; absolute counts in tooltip.
- [ ] Legend order and stack segment order match.

**Testing**
- Snapshot tests.
- Manual: seed a tenant with cases in only two open statuses; the chart still shows every in-scope open status as a bar.
