# GitHub Issue & PR Templates — Issue #217

---

## GitHub Issue Body

### Problem

The Case Ageing Report page carries ten cross-cutting defects catalogued by @Justus-at-Tazama in the current issue body. Two of them — `FRAUD_AND_AML` container being counted as a case (quirk 7) and the corresponding `STATUS_84_COMPLETED` divergence (quirk 4) — have **already been resolved on `dev`** by applying `ReportsService.withNonContainerCaseFilter` to all three query builders inside `getCaseAgeing`, and by dropping `FRAUD_AND_AML` from the per-type resolution loop. Those quirks are consequently descoped from this issue and tracked separately under issue #214.

Of the remaining eight quirks, the two with the largest blast radius are:

- **Quirk 6 — investigator scope conflates "my cases" with "claimable cases."** The investigator OR clause is copy-pasted into three query builders inside `getCaseAgeing`. Each copy contains four arms: owned-by-me, task-assigned-to-me, `case_owner_user_id: null`, and `status: STATUS_02_READY_FOR_ASSIGNMENT`. The last two arms sum the entire tenant's shared claimable pool into every ageing widget for the investigator.
- **Quirk 9 — abandoned `STATUS_99_ABANDONED` cases are in ageing scope.** The main `findMany` applies no status filter, so abandoned draft-only cases inflate Avg Case Age, `>15` / `>30` cards, the distribution pie, the by-status bar, and the details table. `CLOSED_STATUSES` excludes `99`, so abandoned cases never leave the scope by way of "resolved" — they age forever and resolve never.

The remaining quirks (bucket boundary inconsistency at 30 days, overlapping stat cards, per-close-day "trend" mislabelled as monthly, empty-population averages collapsing to `0`, ALL-CAPS status labels, unstable status axis, per-type serial queries, dead `dateRange`, duplicated owner columns in the table, backend-formatted `en-US` dates) are per-widget presentation and scope defects.

### Root Cause

`getCaseAgeing` treats the Case Ageing Report as a single-scope operation over one unfiltered dataset. There is no lifecycle discipline (open vs closed), no shared age-band definition, no shared investigator-scope helper for the ageing report specifically, no status-set exclusion, and per-widget scope wiring is either absent or contradictory. Every downstream symptom flows from this: the four-arm OR clause is copy-pasted because there is no helper; abandoned cases sit in every widget because there is no query-level exclusion; open and closed cases share the same widgets because the report is not lifecycle-scoped.

### Impact

- Investigator's personal Avg Case Age, `>15` / `>30` counts, distribution pie, by-status bars, and details table are inflated by tenant-wide claimable work that they do not own.
- Dead abandoned drafts show up in every ageing widget for both supervisor and investigator callers.
- Bucket boundaries silently disagree between the two stat cards, the by-status bar, and the pie — a case aged exactly 30 days lands in `30+`, not `16-30`.
- The `>15` and `>30` cards overlap (`30+ ⊂ 15+`), double-counting the oldest cases.
- The trend chart's field is called `month` but is a per-close-day category axis over a fixed 6-month window.
- Empty populations report `0`, indistinguishable from a genuine zero-day mean.
- Status labels overflow the x-axis in ALL-CAPS and the status axis silently changes shape from one request to the next.
- The `dateRange` picker on the page silently does nothing for the core dataset.

### Proposed Fix

**Track A — Investigator scope helper + abandoned exclusion (surgical, backend-only).**

- Extract a new `applyAgeingInvestigatorScope` helper that keeps only two arms: `case_owner_user_id = me` and `tasks.some.assigned_user_id = me`.
- Replace the three inline OR clauses inside `getCaseAgeing` (shared `findMany`, per-type `caseTypeResolution`, `resolutionTrend`) with calls to the helper.
- Add `status: { not: CaseStatus.STATUS_99_ABANDONED }` to the shared base filters. The per-type query already restricts to `CLOSED_STATUSES` (which excludes `99`), so no change needed there.
- **Files:** `backend/src/modules/report/report.service.ts` (single file).
- **Schema migration:** No.
- **Frontend changes:** No.
- **Effort:** ~2.5 hours.

**Track B — Full nine-widget rework (backend + frontend).**

- Split `getCaseAgeing` into an **open-snapshot** dataset (Avg Case Age, `>15` / `>30`, by-status bar, distribution pie, details table) and a **closed-in-window** dataset (Avg Resolution Time, trend, case-type bar), with `dateRange` bound to the closed dataset only.
- Single shared age-band helper with Hamilton largest-remainder rounding, consumed by the bar and the pie.
- Make `>15` and `>30` cards non-overlapping or clearly cumulative; settle on one boundary convention.
- Return `null` (not `0`) for empty means; render as `N/A` on the cards.
- Replace the per-close-day trend with a monthly median + IQR band on a continuous temporal axis.
- Replace the per-type `Promise.all` with a single grouped `case_type` aggregation.
- Stop pre-formatting `status` and `createdDate` server-side; emit ISO 8601 dates and code-prefixed title-case status on the wire.
- Seed the by-status axis from the canonical open-status set (so statuses with zero rows still render).
- Drop the duplicated raw "User ID" column in the details table.
- **Files:** ~10 across backend and frontend.
- **Schema migration:** No.
- **Frontend changes:** Yes.
- **Effort:** ~6-8 days.

Container / `STATUS_84` retirement (quirks 4 and 7) is out of scope here and tracked under issue #214.

### Acceptance Criteria

**Track A**
- [ ] `getCaseAgeing` contains no inline four-arm OR clause; a single `applyAgeingInvestigatorScope` helper handles investigator scope.
- [ ] The helper keeps only owned-by-me + task-assigned-to-me arms.
- [ ] All three query builders inside `getCaseAgeing` go through the helper.
- [ ] The shared `findMany` filters out `STATUS_99_ABANDONED` at the query level.
- [ ] An investigator no longer sees unassigned or ready-for-assignment cases in any ageing widget.
- [ ] An investigator no longer sees abandoned cases in any ageing widget.
- [ ] Supervisor / admin output is unchanged except for the abandoned exclusion.
- [ ] Unit tests for `getCaseAgeing` pass with updated fixtures.

**Track B**
- [ ] Open-snapshot and closed-in-window datasets are separated; `dateRange` bound to the latter.
- [ ] Single shared age-band helper for 3.3 / 3.4 / 3.5 / 3.6; one boundary convention.
- [ ] `>15` and `>30` are non-overlapping or labelled cumulative.
- [ ] `avgCaseAge` and `avgResolutionTime` are `number | null`; cards render `N/A`.
- [ ] `resolutionTrend` is monthly `{ bucketStart, median, p25, p75, n }` on a continuous axis with an IQR band.
- [ ] `caseTypeResolution` uses a single grouped aggregation on the closed-in-window dataset.
- [ ] `caseDetails.createdDate` is ISO 8601 on the wire; raw "User ID" column dropped; status is code-prefixed title-case.
- [ ] By-status bar seeded from the canonical open-status set (deterministic axis).
- [ ] Distribution pie sums to exactly 100 (Hamilton rounding).

---

## PR Descriptions

### PR 1 — Track A

#### PR Title

fix: scope investigator ageing report to owned work and drop abandoned cases

#### PR Body

**Summary**
- Extracts a new `applyAgeingInvestigatorScope` helper in `ReportsService` that keeps only `case_owner_user_id = me` and `tasks.some.assigned_user_id = me`.
- Replaces the three inline four-arm OR clauses inside `getCaseAgeing` (shared `findMany`, per-type resolution, recent-closed trend) with calls to the helper.
- Adds `status: { not: STATUS_99_ABANDONED }` at the query level for the shared base filters. Per-type query already filters to `CLOSED_STATUSES` (which excludes `99`); recent-closed trend query same.
- Investigator's Avg Case Age, `>15` / `>30`, distribution, by-status, and details table stop double-counting tenant-wide unassigned + ready-for-assignment cases.
- Abandoned drafts drop out of every ageing widget for both roles.

**Test Plan**
- [ ] Log in as investigator U1 with owned + task-assigned cases; ensure tenant has unassigned cases and one `STATUS_02_READY_FOR_ASSIGNMENT` case owned by U2. Open the Case Ageing Report. **Expect** the details table to list only U1's own cases; unassigned and ready-for-assignment rows are absent.
- [ ] Log in as supervisor. Seed one `STATUS_99_ABANDONED` case in the tenant. Open the Case Ageing Report. **Expect** no `99 Abandoned` bar in the by-status chart and no abandoned row in the details table.
- [ ] Compare an investigator's Avg Case Age before and after the fix on the same tenant. **Expect** the number to drop (claimable arms no longer counted).
- [ ] Run the backend unit tests (`report.service.spec.ts`) — pass.

**Related Issue**
Closes part of https://github.com/tazama-lf/case-management-system/issues/217 (Track A). Track B is a separate PR.

### PR 2 — Track B

#### PR Title

feat: rework Case Ageing Report — lifecycle-scoped datasets, monthly trend, N/A empties

#### PR Body

**Summary**
- Splits `getCaseAgeing` into an **open-snapshot** dataset and a **closed-in-window** dataset; `dateRange` is now bound to the closed dataset only.
- Introduces a single shared age-band helper with Hamilton rounding; consumed by both the by-status bar and the distribution pie.
- Returns `null` for empty averages; frontend cards render `N/A`.
- Replaces the per-close-day trend with a monthly median + IQR band on a continuous temporal axis via recharts `ComposedChart`.
- Replaces the per-type `Promise.all` with a single grouped `case_type` aggregation.
- Backend stops pre-formatting `status` (now emitted as the raw enum + code-prefixed title-case on the client) and `createdDate` (now emitted as ISO 8601).
- Duplicated raw "User ID" column in the details table dropped; owner name resolved on the client.
- By-status bar seeded from the canonical open-status set with a deterministic, code-sorted axis.

**Test Plan**
- [ ] Change the `dateRange` picker; **expect** only Avg Resolution Time, the trend line, and the case-type bar to shift; the open-snapshot widgets do not change.
- [ ] With no closed cases in scope, **expect** Avg Resolution Time to render `N/A`, not `0`.
- [ ] The distribution pie's four percentages sum to exactly 100.
- [ ] The by-status bar renders every in-scope open status as a labelled bar (empty bars visible), even if some have zero cases.
- [ ] Backend `report.service.spec.ts` and any impacted frontend component tests pass.

**Migration**
No schema migration.

**Sequencing Note**
This PR must be merged **after** the Track A PR. Track A retires the four-arm OR clause and adds abandoned exclusion; Track B builds on that.

**Related Issue**
Closes https://github.com/tazama-lf/case-management-system/issues/217 (Track B).
