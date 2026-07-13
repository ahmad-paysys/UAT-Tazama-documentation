# Issue #217 — Update to the Reports - Case Ageing Report page

**Repository:** tazama-lf/case-management-system
**Issue:** [Update to the Reports - Case Ageing Report page](https://github.com/tazama-lf/case-management-system/issues/217)
**Author:** Justus-at-Tazama (Justus Ortlepp)
**State:** Open
**Report Date:** 2026-07-13

---

## Executive Summary

The Case Ageing Report page carries ten cross-cutting quirks catalogued by the issue author (section 4 of the issue body). Two of those quirks — the `FRAUD_AND_AML` container being counted as a case (quirk 7) and the corresponding `STATUS_84_COMPLETED` divergence (quirk 4) — have **already been resolved on `dev`**: `ReportsService.withNonContainerCaseFilter` is applied at the query level in all three query builders inside `getCaseAgeing` ([report.service.ts:938,1058,1126](../../../repos/case-management-system/backend/src/modules/report/report.service.ts)), and the per-type resolution loop excludes `CaseType.FRAUD_AND_AML` ([report.service.ts:1046](../../../repos/case-management-system/backend/src/modules/report/report.service.ts)). Those quirks are consequently descoped from this issue and tracked under issue #214.

Of the remaining eight quirks, the two with the largest semantic blast radius are **quirk 6** (investigator scope conflates "my cases" with "claimable cases", inflating every widget's personal metrics) and **quirk 9** (abandoned `STATUS_99` cases in the ageing scope contribute pure noise to every widget). Both are dataset-level defects that flow into all seven widgets fed by the shared query. The remaining quirks (bucket-boundary inconsistency, overlapping stat cards, mislabelled trend, empty-average collapse to `0`, ALL-CAPS status labels, unstable status axis, per-type serial queries, dead `dateRange`) are per-widget presentation or scope defects and are deferred to Track B.

Track A is the surgical dataset-scope fix: extract the three copy-pasted investigator OR clauses inside `getCaseAgeing` into a shared helper that keeps only the two ownership arms (owned-by-me + task-assigned-to-me), and add `STATUS_99_ABANDONED` exclusion at the query level. Backend-only, no schema, no frontend. Track B is the full nine-widget rework called for in the issue.

---

## How It Works Today — Confirmed in Code

Note: line numbers below are cited from `origin/dev` at the time of writing.

### The shared query and the triplicated investigator OR clause

`getCaseAgeing` accepts a `dateRange` but never applies it. The main dataset is a single unbounded `findMany` ([report.service.ts:967-978](../../../repos/case-management-system/backend/src/modules/report/report.service.ts)):

```ts
const cases = await this.prisma.case.findMany({
  where: whereClause,
  select: { case_id, status, case_type, created_at, updated_at, priority, case_owner_user_id },
});
```

`whereClause` is built once at the top of the method ([report.service.ts:933-965](../../../repos/case-management-system/backend/src/modules/report/report.service.ts)) as `baseFilters` (tenant + `withNonContainerCaseFilter`) plus, for an investigator caller, this inline OR clause:

```ts
if (filters?.requestingUserId) {
  whereClause = {
    AND: [
      baseFilters,
      {
        OR: [
          { case_owner_user_id: filters.requestingUserId },      // owned by me
          { tasks: { some: { assigned_user_id: filters.requestingUserId } } },  // task assigned to me
          { case_owner_user_id: null },                          // any unassigned case
          { status: 'STATUS_02_READY_FOR_ASSIGNMENT' },          // any ready-for-assignment case
        ],
      },
    ],
  };
}
```

The same four-arm OR clause is **copy-pasted** into two more query builders in the same method:
- `caseTypeResolution` per-type query ([report.service.ts:1063-1082](../../../repos/case-management-system/backend/src/modules/report/report.service.ts))
- `resolutionTrend` recent-closed query ([report.service.ts:1131-1153](../../../repos/case-management-system/backend/src/modules/report/report.service.ts))

There is an existing `applyInvestigatorScope` helper on the service ([report.service.ts:101-149](../../../repos/case-management-system/backend/src/modules/report/report.service.ts)) but its semantics differ from the ageing report's needs (it includes draft/pending-approval cases gated by `case_creator_user_id`), so `getCaseAgeing` does not use it.

### No status filter on the main dataset

The main `findMany` above supplies **no `status` clause**, so `STATUS_99_ABANDONED` cases are included in `avgCaseAge`, `casesOver15Days`, `casesOver30Days`, `ageingByStatus`, `ageingDistribution`, and `caseDetails`. `STATUS_99_ABANDONED` is not in `CLOSED_STATUSES` ([report.service.ts:24-30](../../../repos/case-management-system/backend/src/modules/report/report.service.ts)), so it is never counted as resolved either. It ages forever, resolves never.

### Bucket boundary inconsistency (quirk 2)

```ts
const casesOver15Days = casesWithAge.filter((c) => c.ageDays > 15).length;   // >
const casesOver30Days = casesWithAge.filter((c) => c.ageDays >= 30).length;  // >=
```
([report.service.ts:998-999](../../../repos/case-management-system/backend/src/modules/report/report.service.ts))

`ageingByStatus` uses `age16to30 = > 15 && < 30` and `age30Plus = >= 30` ([report.service.ts:1014-1017](../../../repos/case-management-system/backend/src/modules/report/report.service.ts)); the pie ([report.service.ts:1021-1036](../../../repos/case-management-system/backend/src/modules/report/report.service.ts)) recomputes the same four bands independently.

### Empty-average collapse to `0` (quirk 10)

Both means fall back to `0` when the population is empty ([report.service.ts:986,990-996](../../../repos/case-management-system/backend/src/modules/report/report.service.ts)), so "no cases" is indistinguishable from a genuine zero-day mean. The wire type is `number` ([reports.types.ts:180-181](../../../repos/case-management-system/frontend/src/features/reports/types/reports.types.ts)) and the card renders the number verbatim ([CaseAgeingStatsCards.tsx:84,91](../../../repos/case-management-system/frontend/src/features/reports/components/CaseAgeingStatsCards.tsx)).

### `resolutionTrend` is per-close-day (quirk 5)

The backend emits one row **per closed case**, keyed by `updated_at` day-formatted string, over a fixed 6-month window ([report.service.ts:1112-1172](../../../repos/case-management-system/backend/src/modules/report/report.service.ts)); the field name `month` is misleading and the chart is a category axis of days that happened to have a closure.

### ALL-CAPS status names and unstable axis (quirks in 3.5)

`formatStatusName` strips `STATUS_` and replaces `_` with a space ([report.service.ts:1211-1213](../../../repos/case-management-system/backend/src/modules/report/report.service.ts)), producing labels like `READY FOR ASSIGNMENT`. `ageingByStatus` iterates `Object.entries(statusGroups)` over statuses **present in the returned rows** ([report.service.ts:1011-1019](../../../repos/case-management-system/backend/src/modules/report/report.service.ts)) — statuses with zero rows are silently dropped, so the x-axis changes shape from request to request.

### `caseTypeResolution` — N queries, no date bound (quirk in 3.8)

One `findMany` is issued per `CaseType` value via `Promise.all` ([report.service.ts:1044-1110](../../../repos/case-management-system/backend/src/modules/report/report.service.ts)); the query has no window at all — the widget is unbounded all-time.

### `caseDetails` duplicated owner projection (quirks in 3.9)

`userId` and `investigator` are two projections of the same `case_owner_user_id` ([report.service.ts:1174-1183](../../../repos/case-management-system/backend/src/modules/report/report.service.ts)) — one nullable, one falling back to the string `'Unassigned'`. `createdDate` is pre-formatted as an `en-US` locale date server-side.

---

## Root Cause Analysis

**Root cause (single sentence):** `getCaseAgeing` treats the Case Ageing Report as a single-scope operation over one unfiltered dataset, with no lifecycle discipline (open vs closed), no shared band definition, no shared investigator-scope helper, no status-set exclusion, and per-widget scope wiring.

Every downstream symptom follows from this:

- **Quirk 6** — no shared investigator-scope helper → the OR clause is copy-pasted three times → any per-widget refinement risks drifting across the three copies. The clause conflates ownership with claimability, so every widget's investigator view is polluted with the tenant-wide unassigned + ready-for-assignment pool.
- **Quirk 9** — no status-set exclusion at the query level → abandoned cases are averaged, bucketed, listed as if they were live work; the closed set excludes them, so they never leave the ageing scope.
- **Quirks 1, 2, 3, 5, 10** — no lifecycle discipline and no single band/boundary definition → open and closed cases share the same widgets, thresholds contradict each other, empty populations collapse to real-looking zeros, and the "trend" is a per-day scatter.
- **Presentation quirks (3.5, 3.9)** — presentation is baked into the service layer (`formatStatusName`, `toLocaleDateString`, `'Unassigned'` string) → the frontend cannot re-localise, re-sort, or re-format without a backend round-trip.

---

## Blast Radius — Full File Inventory

### Files that must change (Track A)

| File | What is affected | Lines |
| --- | --- | --- |
| `backend/src/modules/report/report.service.ts` | Extract shared ageing investigator-scope helper; replace three inline OR clauses; add `STATUS_99_ABANDONED` exclusion at the query level for the shared and per-type queries | 940-965, 1044-1085, 1116-1153 |

### Files indirectly affected (Track A)

| File | What is affected | Lines |
| --- | --- | --- |
| `backend/src/modules/report/__tests__/report.service.spec.ts` (if present) | Update fixtures that assume abandoned + claimable cases appear in an investigator's ageing view | tests |

### Files that must change (Track B — full nine-widget rework)

| File | What is affected | Lines |
| --- | --- | --- |
| `backend/src/modules/report/report.service.ts` | Split `getCaseAgeing` into an **open snapshot** dataset and a **closed-in-window** dataset; single shared age-band helper; per-widget scope; make `resolutionTrend` monthly with median + IQR; single grouped `groupBy case_type`; return `null` for empty means; stop pre-formatting `status`/`createdDate`; drop duplicated `investigator` projection | 923-1197 |
| `backend/src/modules/report/report.controller.ts` | Wire `dateRange` through to the closed-in-window dataset only; swagger schema updates for `null`-able averages | 387-459 |
| `frontend/src/features/reports/types/reports.types.ts` | `avgCaseAge`, `avgResolutionTime` → `number \| null`; `ageingByStatus[]` status field carries code + display separately; `resolutionTrend` shape `{ bucketStart, median, p25, p75, n }`; `caseDetails.createdDate` → ISO 8601; drop duplicated `userId` | 180-286 |
| `frontend/src/features/reports/pages/CaseAgeingReport.tsx` | Consume new shapes | 1-158 |
| `frontend/src/features/reports/components/CaseAgeingStatsCards.tsx` | Render `null` as `N/A`; label cards "live backlog" vs "closed throughput" | 1-120 |
| `frontend/src/features/reports/components/CaseAgeingBarChart.tsx` | Horizontal `layout="vertical"`, 100% stacked, seed all in-scope open statuses, code-prefixed title-case labels, deterministic band order | full file |
| `frontend/src/features/reports/components/CaseAgeingPieChart.tsx` | Consume shared band definition; Hamilton largest-remainder rounding | full file |
| `frontend/src/features/reports/components/ResolutionTimeTrendChart.tsx` | Continuous temporal axis, median line + IQR band via `ComposedChart` | full file |
| `frontend/src/features/reports/components/CaseTypeResolutionChart.tsx` | Consume grouped result; add companion "Resolution Time by Outcome" bar | full file |
| `frontend/src/features/reports/components/CaseAgeingTable.tsx` | Drop raw "User ID" column; consume ISO `createdDate`; code-prefixed status | full file |

---

## Side-Effect Map

| If left unfixed | Downstream consequence |
| --- | --- |
| Investigator OR clause keeps claimable arms | Every widget on an investigator's page double-counts the tenant-wide unassigned + ready-for-assignment pool as personal backlog; the investigator's Avg Case Age, over-15/30 counts, distribution, by-status bars, and detail table are all inflated |
| Abandoned `STATUS_99` cases remain in scope | Dead draft-only cases age forever and resolve never — pure noise in every ageing widget; the "abandoned" bar in `ageingByStatus` presents dead cases as live work |
| Triplicated OR clause not extracted | Any future per-widget scope refinement risks drifting across three copies; a fifth OR arm would already need to be added in three places |
| `dateRange` remains dead | The date picker on the page silently does nothing; users assume they are looking at a windowed view when they are not |
| Bucket boundary + card overlap left | A case aged exactly 30 days lands in `30+` (not `16-30`); every case in the `>30` card is also in the `>15` card — the two are cumulative but not labelled as such |
| `resolutionTrend` per-day scatter | The chart title says "Trend" but shows a category axis of days-with-closures; outlier-dominated, no sample-size indication, no rate-of-change reading |
| Empty averages read as `0` | "No qualifying cases" is indistinguishable from a genuine zero-day mean on the two average cards |
| ALL-CAPS labels + unstable axis | Status labels overflow the x-axis; the set of bars silently changes from one request to the next |

---

## Effort Assessment

### Track A — investigator scope + abandoned exclusion

- **What changes:** one backend file (`report.service.ts`). Extract a new private helper `applyAgeingInvestigatorScope` that keeps only two OR arms (`case_owner_user_id = me`, `tasks.some.assigned_user_id = me`). Replace the three inline OR clauses inside `getCaseAgeing`. Add `status: { not: CaseStatus.STATUS_99_ABANDONED }` to the shared `findMany` and per-type queries (`resolutionTrend` already restricts to `CLOSED_STATUSES`, so `99` is already excluded there).
- **Lines:** ~60 net (~40 for the OR consolidation, ~20 for the abandoned filter).

| Step | File | Effort |
| --- | --- | --- |
| A1. Add `applyAgeingInvestigatorScope` helper | `report.service.ts` | 15 min |
| A2. Replace the three inline OR clauses | `report.service.ts` | 20 min |
| A3. Add `STATUS_99_ABANDONED` exclusion at the shared and per-type queries | `report.service.ts` | 15 min |
| A4. Update service tests if present | `__tests__/report.service.spec.ts` | 30 min |
| A5. Manual smoke test as investigator and as supervisor | — | 30 min |

- **Schema migration required:** No.
- **Frontend changes required:** No.

### Track B — full nine-widget rework

- **What changes:** ~10 files across backend + frontend; splits `getCaseAgeing` into two datasets (open snapshot / closed-in-window), introduces a shared age-band helper, monthly-median trend with IQR, single grouped case-type query, `null`-able averages, ISO-8601 dates, code-prefixed title-case status labels, horizontal 100% stacked bar, seeded open-status set, Hamilton rounding on the pie, dropped duplicated `investigator` column.
- **Lines:** ~600-900 across backend + frontend.

| Step | File | Effort |
| --- | --- | --- |
| B1. Split `getCaseAgeing` into `getOpenAgeingSnapshot` + `getClosedResolutionMetrics` | `report.service.ts` | 1 day |
| B2. Shared age-band definition + Hamilton rounding | `report.service.ts` | 2 hours |
| B3. Monthly `groupBy` for `resolutionTrend` with `{ median, p25, p75, n }` | `report.service.ts` | 4 hours |
| B4. Single grouped `case_type` aggregation for the type bar | `report.service.ts` | 2 hours |
| B5. Return `null` for empty means; adjust wire type | `report.service.ts` + `reports.types.ts` | 1 hour |
| B6. Stop pre-formatting `status` / `createdDate`; drop duplicated `investigator` | `report.service.ts` + `reports.types.ts` | 2 hours |
| B7. `dateRange` wired through the controller to the closed-in-window dataset only | `report.controller.ts` | 1 hour |
| B8. `CaseAgeingStatsCards`: `N/A` render + basis labels | frontend | 1 hour |
| B9. `CaseAgeingBarChart` horizontal + seeded status set + code-prefixed title-case | frontend | 3 hours |
| B10. `CaseAgeingPieChart` consumes shared band definition | frontend | 1 hour |
| B11. `ResolutionTimeTrendChart` median line + IQR band via `ComposedChart` | frontend | 4 hours |
| B12. `CaseTypeResolutionChart` grouped consumer + companion "by outcome" bar | frontend | 3 hours |
| B13. `CaseAgeingTable` drop User ID column; consume ISO date; code-prefixed status | frontend | 1 hour |
| B14. Tests + UAT | — | 1 day |

- **Schema migration required:** No.
- **Frontend changes required:** Yes.

### Recommended sequencing

1. **Ship Track A first, in isolation.** It is backend-only and clears the two dataset-level polluters (quirks 6 and 9) that inflate every widget in the report for an investigator caller.
2. Track B is scoped separately and depends on Track A being merged first, so Track B's per-widget refinements (open-only scoping, single band definition) do not have to reason about the old four-arm OR clause or the old abandoned-in-scope behaviour.
3. Container / `STATUS_84` retirement (quirks 4 and 7) is tracked on **issue #214** and is out of scope for both tracks here — dev already applies `withNonContainerCaseFilter` inside `getCaseAgeing`.

---

## Acceptance Criteria

### Track A

- [ ] `getCaseAgeing` contains no inline four-arm OR clause; the investigator scope is expressed through a single `applyAgeingInvestigatorScope` helper.
- [ ] The helper keeps only `case_owner_user_id = me` and `tasks.some.assigned_user_id = me`; the two claimable arms (`case_owner_user_id: null` and `status: 'STATUS_02_READY_FOR_ASSIGNMENT'`) are gone.
- [ ] All three query builders inside `getCaseAgeing` — shared `findMany`, per-type resolution, recent-closed trend — go through the helper.
- [ ] The shared `findMany` and per-type queries filter out `STATUS_99_ABANDONED` at the query level.
- [ ] An investigator caller no longer sees unassigned or ready-for-assignment cases in Avg Case Age, over-15/30, distribution, by-status, or the details table.
- [ ] An investigator caller no longer sees abandoned cases in any ageing widget.
- [ ] Supervisor / admin (no `requestingUserId`) output is unchanged apart from the abandoned exclusion.
- [ ] Existing unit tests for `getCaseAgeing` pass with updated fixtures.

### Track B

- [ ] `getCaseAgeing` exposes an open-snapshot dataset and a closed-in-window dataset, with `dateRange` bound to the latter only.
- [ ] A single shared age-band helper drives the four buckets used by 3.3, 3.4, 3.5, 3.6.
- [ ] `casesOver15Days` and `casesOver30Days` use the same boundary convention and are either non-overlapping (`16-30` / `30+`) or clearly labelled cumulative.
- [ ] `avgCaseAge` and `avgResolutionTime` return `null` for an empty population; both cards render `null` as `N/A`.
- [ ] `resolutionTrend` returns one row per calendar month with `{ bucketStart, median, p25, p75, n }`; the chart uses a continuous temporal axis with an IQR band.
- [ ] `caseTypeResolution` is produced by a single grouped aggregation and is bound to the same close-anchored window as the trend and Avg Resolution Time.
- [ ] `caseDetails.createdDate` is emitted as ISO 8601; the client formats it for display; the raw "User ID" column is dropped in favour of the resolved investigator name; status is code-prefixed title-case.
- [ ] The by-status bar renders every in-scope open status even when its count is zero; the axis is deterministic and sorted by status code.
- [ ] The pie's four percentages sum to exactly 100 using largest-remainder rounding.
