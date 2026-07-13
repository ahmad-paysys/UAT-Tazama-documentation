# Impact Study — Issue #217: Update to the Reports - Case Ageing Report page

**Issue:** [#217](https://github.com/tazama-lf/case-management-system/issues/217)
**Study Date:** 2026-07-13
**Author:** Ahmad Khalid
**Repository:** tazama-lf/case-management-system

---

## Summary

Issue #217 is Justus's per-widget analysis of the Case Ageing Report, cataloguing ten cross-cutting quirks. Two of them — `FRAUD_AND_AML` container-as-Case (quirk 7) and `STATUS_84_COMPLETED` divergence (quirk 4) — are already resolved on `dev` inside `getCaseAgeing` and are tracked separately under issue #214. Of the remaining eight, the two dataset-level defects (quirk 6, investigator claimable pollution; quirk 9, `STATUS_99_ABANDONED` pollution) inflate every widget on the page for an investigator caller and are the most surgical thing to fix. Track A is that dataset-level fix, backend-only. Track B is the full nine-widget rework called for in section 5 of the issue.

---

## Confirmed Root Cause

Every claim below was verified against `origin/dev` of `tazama-lf/case-management-system` at the time of study.

- **Investigator OR clause is triplicated inside `getCaseAgeing`:**
  - Shared `findMany` — [report.service.ts:944-962](../../../repos/case-management-system/backend/src/modules/report/report.service.ts)
  - `caseTypeResolution` per-type query — [report.service.ts:1063-1082](../../../repos/case-management-system/backend/src/modules/report/report.service.ts)
  - `resolutionTrend` recent-closed query — [report.service.ts:1131-1153](../../../repos/case-management-system/backend/src/modules/report/report.service.ts)
- **Each copy has the same four arms:** owned by me, task assigned to me, `case_owner_user_id: null` (any unassigned), `status: 'STATUS_02_READY_FOR_ASSIGNMENT'` (any ready-for-assignment). The last two arms are the "claimable pool" arms that pollute an investigator's personal metrics.
- **An existing `applyInvestigatorScope` helper is present** at [report.service.ts:101-149](../../../repos/case-management-system/backend/src/modules/report/report.service.ts) but its semantics differ (it also matches drafts/pending-approval by `case_creator_user_id`); `getCaseAgeing` does not use it.
- **The main `findMany` applies no `status` filter** ([report.service.ts:967-978](../../../repos/case-management-system/backend/src/modules/report/report.service.ts)) — abandoned cases (`STATUS_99_ABANDONED`) are counted.
- **`CLOSED_STATUSES` excludes `STATUS_99_ABANDONED`** ([report.service.ts:24-30](../../../repos/case-management-system/backend/src/modules/report/report.service.ts)) — abandoned cases never leave the ageing scope by way of "resolved".
- **Container quirks 4 and 7 are already resolved on `dev`:** `withNonContainerCaseFilter` is applied to all three query builders ([report.service.ts:938,1058,1126](../../../repos/case-management-system/backend/src/modules/report/report.service.ts)) and the per-type loop skips `FRAUD_AND_AML` ([report.service.ts:1046](../../../repos/case-management-system/backend/src/modules/report/report.service.ts)). Container work continues under issue #214.
- **Bucket boundary inconsistency (quirk 2):** `casesOver15Days = ageDays > 15`, `casesOver30Days = ageDays >= 30` ([report.service.ts:998-999](../../../repos/case-management-system/backend/src/modules/report/report.service.ts)).
- **Empty averages (quirk 10):** `avgCaseAge` and `avgResolutionTime` short-circuit to `0` ([report.service.ts:986,990-996](../../../repos/case-management-system/backend/src/modules/report/report.service.ts)); wire type is `number` ([reports.types.ts:180-181](../../../repos/case-management-system/frontend/src/features/reports/types/reports.types.ts)).
- **`resolutionTrend` is per-close-day (quirk 5):** one row per closed case keyed by `updated_at.toLocaleDateString('en-US', ...)` over a fixed 6-month window ([report.service.ts:1112-1172](../../../repos/case-management-system/backend/src/modules/report/report.service.ts)).
- **ALL-CAPS status labels + unstable axis:** `formatStatusName` at [report.service.ts:1211-1213](../../../repos/case-management-system/backend/src/modules/report/report.service.ts); status axis derived from `Object.entries(statusGroups)` — missing statuses drop out silently ([report.service.ts:1011-1019](../../../repos/case-management-system/backend/src/modules/report/report.service.ts)).

---

## Track A — Investigator scope + abandoned exclusion

### What Changes

One backend file: `backend/src/modules/report/report.service.ts`. Introduce `applyAgeingInvestigatorScope` (kept local to the service, ~15 lines), replace the three inline OR clauses inside `getCaseAgeing` (~-45 lines / +3 helper calls), and add `status: { not: CaseStatus.STATUS_99_ABANDONED }` to the shared and per-type queries (`resolutionTrend` already filters to `CLOSED_STATUSES` which excludes `99`). Net delta ~60 lines.

### Impact

| Dimension | Value |
| --- | --- |
| Files changed | 1 (backend) |
| Schema migration required | No |
| Frontend changes required | No |
| Downtime required | No |
| Risk of regression | Low — the helper is a strict subset of today's OR clause; supervisor path is unchanged apart from the abandoned exclusion |
| Reversibility | Trivial (revert one commit) |

**What Track A fixes immediately:**
- Investigator's Avg Case Age no longer averages tenant-wide unassigned + ready-for-assignment cases into their personal mean (quirk 6).
- Investigator's `> 15` / `> 30` cards, distribution pie, by-status bars, and details table are no longer inflated by cases nobody owns.
- The three copy-pasted OR clauses inside `getCaseAgeing` are consolidated into a single helper — future scope refinements land in one place.
- Abandoned cases stop showing up as ageing rows in every widget fed by the shared query and in the per-type resolution bar.

**What Track A does not fix:**
- `dateRange` still ignored (quirk 1).
- Bucket boundary inconsistency at 30 days (quirk 2).
- Over-15 / over-30 cards still overlap (quirk 3).
- `resolutionTrend` is still a per-close-day scatter, not a monthly trend (quirk 5).
- `avgCaseAge` / `avgResolutionTime` still collapse to `0` for empty populations (quirk 10).
- ALL-CAPS status labels, unstable status axis, per-type serial queries, table duplicated owner columns, `en-US` server-formatted dates all untouched.
- Widgets still count open + closed cases together (no lifecycle-scoped datasets).
- `STATUS_03_RETURNED` treatment divergence with the Case Status Report (quirk 8) is untouched.

**Track A is safe to ship in isolation.**

---

## Track B — Full nine-widget rework

### What Changes

Split `getCaseAgeing` into two lifecycle-scoped datasets (open snapshot for 3.1/3.3/3.4/3.5/3.6/3.9, closed-in-window for 3.2/3.7/3.8), introduce a single shared age-band helper with Hamilton rounding, monthly-median trend with IQR, single grouped case-type aggregation, `null`-able averages, ISO-8601 dates, code-prefixed title-case status labels, horizontal 100% stacked bar, seeded open-status set, dropped duplicated `investigator`/`userId` projection.

### Schema Impact

| Change | Migration risk |
| --- | --- |
| None. All Track B changes are query-shape and presentation only. | — |

Note: the issue's section 7 mentions a possible dedicated `closed_at` column to future-proof against in-place writes to closed rows; the same section states that under last-close-wins, `updated_at` is functionally a `closed_at` today and no schema change is required for correctness. Deferred.

### Backend Code Impact

| File | Change kind | Effort |
| --- | --- | --- |
| `backend/src/modules/report/report.service.ts` — `getCaseAgeing` | Split into two dataset builders; shared band helper; monthly `groupBy` for trend; grouped `case_type` aggregation; `null` empty averages; stop server-formatting `status`/`createdDate`; drop duplicated `investigator` | 1.5 days |
| `backend/src/modules/report/report.controller.ts` — `getCaseAgeing` | Wire `dateRange` through to the closed-in-window dataset only; update swagger schema for nullable averages | 1 hour |
| `backend/src/modules/report/__tests__/report.service.spec.ts` (if present) | New fixtures per dataset; band-helper unit tests; monthly-trend unit tests | 4 hours |

### Frontend Code Impact

| File | Change kind | Effort |
| --- | --- | --- |
| `frontend/src/features/reports/types/reports.types.ts` — `CaseAgeingData`, `CaseAgeingStats` | `number \| null` for averages; new trend row shape; ISO 8601 dates; drop `userId` | 30 min |
| `frontend/src/features/reports/pages/CaseAgeingReport.tsx` | Consume new shapes | 30 min |
| `frontend/src/features/reports/components/CaseAgeingStatsCards.tsx` | Render `null` as `N/A`; label live-backlog vs closed-throughput | 1 hour |
| `frontend/src/features/reports/components/CaseAgeingBarChart.tsx` | Horizontal layout; 100% stacked; seeded open-status set; code-prefixed title-case; deterministic band order | 3 hours |
| `frontend/src/features/reports/components/CaseAgeingPieChart.tsx` | Consume shared band definition; Hamilton largest-remainder rounding | 1 hour |
| `frontend/src/features/reports/components/ResolutionTimeTrendChart.tsx` | Continuous temporal axis; median + IQR band via `ComposedChart` | 4 hours |
| `frontend/src/features/reports/components/CaseTypeResolutionChart.tsx` | Consume grouped result; add companion "Resolution Time by Outcome" bar | 3 hours |
| `frontend/src/features/reports/components/CaseAgeingTable.tsx` | Drop raw User ID column; ISO date on wire → localised on client; code-prefixed status | 1 hour |
| `frontend/src/features/reports/services/reportsService.ts` | Type-align with new payload | 30 min |

Test fixtures across the reports feature will need to be re-generated to match the new dataset shapes; scope this at ~half a day.

---

## Side Effects and Risks

### Risks of Track A Alone

| Risk | Likelihood | Mitigation |
| --- | --- | --- |
| An investigator loses visibility of cases they were incidentally seeing via the claimable arms (unassigned / ready-for-assignment) | Medium | This is the intended behaviour — claimable work belongs on the separate case queue / dashboard, not on a personal ageing report. Coordinate the rollout note with product. |
| An abandoned case an investigator previously used to see in the details table disappears | Low | Abandoned cases are draft-only, terminal, and were never investigated — surfacing them on an ageing report was noise. If ops need visibility of abandoned drafts, they belong on an exception report, not this one. |
| Supervisor / admin numbers shift because abandoned cases drop out of Avg Case Age, over-15/30, distribution, by-status bar, and details table | High (intended) | Communicate the change in the release notes. Supervisors will see slightly lower averages and lower over-15/30 counts as dead drafts stop skewing them upwards. |
| Extracted helper diverges from the OR clauses in other reports that legitimately keep the claimable arms | Low | The helper is scoped local to the ageing report and named `applyAgeingInvestigatorScope`; other reports keep their existing clauses. |

### Risks of Track B

| Risk | Likelihood | Mitigation |
| --- | --- | --- |
| Two-dataset refactor changes the payload shape and breaks the frontend if merged half-way | High | Track B lands as a single coordinated PR (backend + frontend together) — do not split. |
| `null`-able averages break any consumer that assumes `number` | Medium | Wire type change + card update land in the same PR; grep the codebase for `avgCaseAge` / `avgResolutionTime` before merge |
| Monthly-median calculation done backend-side changes the meaning of the chart even if the shape is preserved | Medium | Rename the field (`month` → `bucketStart`, `avgDays` → `median`) so any consumer reading the old field breaks loudly rather than silently |
| Hamilton rounding on the pie means percentages no longer match what the frontend today would compute per-band | Low (intended) | Document in the release notes; the point of Hamilton is that percentages sum to 100 |

### Cross-Issue Dependencies

- **Issue #214 (container / `STATUS_84`)** — already applied inside `getCaseAgeing` on dev, so #217 no longer conflicts with #214 in this file. #214 continues to operate on the container model itself (across BPMN, triage, case-creation) — no file overlap with either track of #217.
- **Issue #220 (SLA / Priority)** — touches `priority` semantics. The ageing report exposes `priority` verbatim in `caseDetails`; Track A does not change that. Track B keeps `priority` on the row and will inherit whatever #220 lands as the priority contract; no coordination needed between Track A and #220.
- **Issue #225 (BIAR investigator dashboard)** — external (BIAR / Superset), no source file overlap with either track here.

No file overlap between Track A of #217 and any other open issue.

---

## Effort Estimate

| Track | Files | Effort |
| --- | --- | --- |
| Track A | 1 backend + tests | ~2.5 hours |
| Track B | 2 backend + 7 frontend + tests | ~6-8 days |
| **Total** | | ~7-9 days |

---

## Acceptance Criteria (Verification Checklist)

### Track A

- [ ] `getCaseAgeing` contains no inline four-arm OR clause; the investigator scope is expressed through a single `applyAgeingInvestigatorScope` helper.
- [ ] The helper keeps only `case_owner_user_id = me` and `tasks.some.assigned_user_id = me`.
- [ ] All three query builders inside `getCaseAgeing` — shared `findMany`, per-type resolution, recent-closed trend — go through the helper.
- [ ] The shared `findMany` and per-type queries filter out `STATUS_99_ABANDONED` at the query level.
- [ ] An investigator caller no longer sees unassigned or ready-for-assignment cases in Avg Case Age, over-15/30, distribution, by-status, or the details table.
- [ ] An investigator caller no longer sees abandoned cases in any ageing widget.
- [ ] Supervisor / admin output is unchanged apart from the abandoned exclusion.
- [ ] Existing unit tests for `getCaseAgeing` pass with updated fixtures.

### Track B

- [ ] `getCaseAgeing` exposes an open-snapshot dataset and a closed-in-window dataset; `dateRange` is bound to the latter only.
- [ ] Single shared age-band helper drives the four buckets used by 3.3 / 3.4 / 3.5 / 3.6.
- [ ] `casesOver15Days` and `casesOver30Days` use the same boundary convention and are either non-overlapping (`16-30` / `30+`) or clearly labelled cumulative.
- [ ] `avgCaseAge` and `avgResolutionTime` return `null` for empty populations; both cards render `null` as `N/A`.
- [ ] `resolutionTrend` returns one row per calendar month with `{ bucketStart, median, p25, p75, n }`; the chart uses a continuous temporal axis with an IQR band.
- [ ] `caseTypeResolution` is produced by a single grouped aggregation and is bound to the same close-anchored window as the trend and Avg Resolution Time.
- [ ] `caseDetails.createdDate` is ISO 8601 on the wire; client formats for display; raw "User ID" column dropped; status is code-prefixed title-case.
- [ ] The by-status bar renders every in-scope open status even when its count is zero.
- [ ] The pie's four percentages sum to exactly 100 (Hamilton rounding).

---

## Recommended Sequencing

1. **Track A first, standalone PR.** ~2.5 hours end-to-end. Merge before Track B begins so Track B does not have to reason about the old four-arm OR clause or the old abandoned-in-scope behaviour.
2. **Track B as a single coordinated PR** (backend + frontend together — the payload shape changes are cross-cutting and cannot be usefully split). Sequence its internal steps roughly B1 → B14 as listed in `issue-217.md`.
3. Container / `STATUS_84` retirement (quirks 4 / 7) tracked separately on **issue #214**. No file overlap with either track of #217 remains.
4. `STATUS_03_RETURNED` treatment reconciliation (quirk 8) is deferred until the Case Status Report is settled; document the divergence in Track B's PR description but do not attempt to fix it inside #217.
