# Impact Study — Issue #214: FRAUD_AND_AML Container Cases Modelled as Peer Case Rows

**Issue:** [#214](https://github.com/tazama-lf/case-management-system/issues/214)  
**Study Date:** 2026-06-30  
**Author:** Ahmad Khalid  
**Repository:** tazama-lf/case-management-system

---

## Summary

Issue #214 identifies a domain modelling error where a `FRAUD_AND_AML` alert produces three `Case` rows — one ownerless container (`FRAUD_AND_AML` type) and two child investigations (`FRAUD` and `AML`) — when only the two child investigations represent real work items. The container has no owner, no verdicts, and no genuine lifecycle, yet it is modelled as a full `Case` entity. Five layers of compensating workaround code have accumulated throughout the codebase to make this arrangement function, and the report service has no awareness of the container, triple-counting every `FRAUD_AND_AML` alert in all reports.

The fix is delivered in two tracks: Track A (one file change) corrects the report inflation immediately; Track B removes the underlying model by introducing a lightweight `InvestigationGroup` table and deleting all propagation workarounds.

This impact study is based on verification of every affected file against the current codebase state.

---

## Confirmed Root Cause

All five compensating layers are confirmed in the codebase:

**1. Three Case rows per alert** (`triage.service.ts` L372, L612): After the container case is created, `createCaseWithInvestigationTask` is called twice (once for `FRAUD`, once for `AML`) with `parentCaseId = alert.case_id`. This produces three `Case` rows in the database per `FRAUD_AND_AML` alert.

**2. Synthetic `STATUS_84_COMPLETED`** (`CloseCaseModal.tsx` L99–107): The only code path that produces this status is a `useEffect` that hardcodes `STATUS_84_COMPLETED` as `recommendedOutcome` whenever `caseData.type === 'FRAUD_AND_AML'`. The BPMN has no `STATUS_84` value; it is entirely a frontend override.

**3. Permission bypass** (`case.repository.ts` L185–196): The `findCaseWithPermissionCheck` query has an OR-branch where `case_type: { in: ['FRAUD_AND_AML'] }` bypasses all owner and assignee checks. The bypass is required because the container is created with `case_owner_user_id: null`. Any user in the tenant can access any `FRAUD_AND_AML` case.

**4. Hand-coded status propagation across four services**: Every significant child case state change checks `if (updatedCase.parent_id)` and manually mirrors status to the parent. This logic exists in seven locations across `task.service.ts`, `task-lifecycle.service.ts`, `case.service.ts`, and `case-reopening.service.ts`.

**5. Report triple-count** (`report.service.ts`): Confirmed by grep — `parent_id` appears zero times in `report.service.ts`. All Prisma `case` queries count all three rows. `STATUS_84_COMPLETED` is in `CLOSED_STATUSES` (line 30) and `STATUS_DISTRIBUTION_MAP` (line 49), causing it to pollute closed totals without corresponding to a real investigator outcome.

---

## Track A — Fix Reporting Triple-Count

### What Changes

**One file:** `backend/src/modules/report/report.service.ts`

A static helper constant `LEAF_CASE_FILTER = { NOT: { case_type: CaseType.FRAUD_AND_AML } }` is added to the class. This constant is spread into `buildCommonCaseFilters`, which is the single entry point for all report queries in the file. Because every count, groupBy, trend query, and workload query calls this helper, the change propagates to all reports with one addition. Two direct queries that bypass the helper (`getInvestigatorWorkload` and `computeStatusDetails`) are patched individually. `STATUS_84_COMPLETED` is removed from the service's private `CLOSED_STATUSES` array and from `STATUS_DISTRIBUTION_MAP` — these are local copies inside the report service; the canonical constants in `case.constants.ts` are not touched in Track A.

### Impact

| Dimension | Effect |
|---|---|
| Files changed | 1 |
| Schema migration required | No |
| Frontend changes required | No |
| Downtime required | No |
| Risk of regression | Very low — additive WHERE clause; cannot make reports return more rows than before |
| Reversibility | Remove the constant and its spreads |

**What this fixes immediately:**
- `totalCases` no longer counts the ownerless container — a single `FRAUD_AND_AML` alert now contributes 2 to the total, not 3
- The `caseTypes` breakdown no longer shows a `FRAUD_AND_AML` bucket; only `FRAUD` and `AML` investigations appear
- `closedCases` no longer includes `STATUS_84_COMPLETED` container closes — closed totals reflect actual investigator verdicts
- `avgResolutionTime` is no longer distorted by container rows that close near-instantly relative to their children
- Investigator workload no longer attributes the ownerless container to the case creator

**What this does not fix:**
- The container `Case` row still exists and is still created for every new `FRAUD_AND_AML` alert
- The permission bypass on `case.repository.ts` remains
- Status propagation across the four services remains
- `STATUS_84_COMPLETED` remains in `case.constants.ts` and the enum — it is still produced and still load-bearing for container closure

**Track A is safe to ship in isolation.** It corrects data corruption in reports with zero risk to the case lifecycle.

---

## Track B — Full Re-Model

### What Changes

Track B removes the container `Case` row entirely, replacing it with a thin `InvestigationGroup` table, and deletes all seven propagation blocks and the permission bypass. It must ship as a single coordinated release; partial states are not compilable because the container creation, propagation checks, and permission bypass are tightly coupled.

### Schema Impact

**`backend/prisma/schema.prisma`:**

| Change | Description |
|---|---|
| Add `InvestigationGroup` model | New table: `id`, `alert_id` (unique), `tenant_id`, `created_at`. No status, no owner, no lifecycle. |
| Retain `Case.parent_id Int?` | Keep in place during Track B for migration purposes; drop in a follow-on cleanup migration |
| Retain `CaseType.FRAUD_AND_AML` | Alerts still arrive as that type; only the `Case` row of that type is eliminated |
| Retain `CaseStatus.STATUS_84_COMPLETED` | Must remain until data migration confirms zero rows; retire in Step B7 after validation |

Migration command: `npx prisma migrate dev --name "add_investigation_group"`

The migration adds one new table with no destructive changes and no column alterations on existing tables. Risk: very low.

### Backend Code Impact

**`triage.service.ts` (Lines 372, 612) — stop creating the container `Case` row:**

The two `FRAUD_AND_AML` branches stop passing the container's `case_id` as `parentCaseId` to `createCaseWithInvestigationTask`. Instead:
1. An `InvestigationGroup` record is upserted, linking `alert_id` to `tenant_id`
2. The two child case creation calls proceed as before, but with `parentCaseId = null`

This is a targeted, low-risk change — child case creation logic is unchanged; only the `parent_id` value passed to it changes from the container's ID to `null`.

**`case-creation.service.ts` (Lines 71, 83) — type signature update:**

`createCaseWithInvestigationTask`'s `parentCaseId` parameter type widens from `number` to `number | null`. The Prisma create call passes the value through unchanged — passing `null` results in `parent_id: null` on the created row, which is already the correct schema type (`Int?`). No logic change required.

**Four services — remove all seven `if (updatedCase.parent_id)` propagation blocks:**

| Service | Locations | Change |
|---|---|---|
| `task.service.ts` | L303 (caller) + L318–353 (method body) | Delete the call site and the entire `promoteParentCaseToInProgress` private method — it has no remaining callers |
| `task-lifecycle.service.ts` | L65–81, L168–184, L273–292 | Delete three `if (updatedCase.parent_id)` blocks |
| `case.service.ts` | L90–107, L229–246 | Delete two `if (updatedCase.parent_id)` blocks (suspend and resume) |
| `case-reopening.service.ts` | L68–73, L244–249 | Delete two `if (updatedCase.parent_id)` blocks |

Each block follows the same pattern and can be deleted with no change to surrounding code. Risk: low. Regression surface: any test that exercises suspension, resumption, reopening, or task assignment of a child case with a valid `parent_id` will need its fixture updated to remove the parent.

**`case.repository.ts` (Lines 185–196) — remove the permission bypass:**

The `FRAUD_AND_AML` OR-branch is replaced by the standard `OR` array containing only owner and task-assignee checks. With no container row in the database, no in-tenant user needs bypass access to a `FRAUD_AND_AML`-typed case.

**`case-closure-approval.service.ts` (Lines 112–146) — remove the container closure gate:**

The `isFraudAndAmlCase` branch (supervisor-only check + "both children closed" validation) is deleted. The `else` path — the standard single-case closure flow — becomes the only path for all cases. This is safe because child `FRAUD` and `AML` cases are now independent and close through the normal process.

**`case.constants.ts` (Step B7 — deferred):** `STATUS_84_COMPLETED` is removed from `CASE_CLOSURE_OUTCOMES`, `CLOSED_CASE_STATUSES`, and `INACTIVE_CASE_STATUSES` only after the data migration confirms zero rows with that value.

**`case-enum.ts` (Step B7 — deferred):** The `COMPLETE = 'STATUS_84_COMPLETED'` value is removed from `CaseClosureOutcome`.

**`schema.prisma` (Step B7 — deferred):** `STATUS_84_COMPLETED` is removed from the `CaseStatus` enum after validation.

**`case-creation-approval.service.ts` (Lines 84, 190, 477):** `FRAUD_AND_AML` branching in the approval flow must be reviewed. If these branches delegate to `createCaseWithInvestigationTask` with the container `case_id` as `parentCaseId`, they receive the same `null` update as the triage service. Verify each branch and update accordingly.

### Frontend Code Impact

**`CloseCaseModal.tsx` (Lines 99–107, 155–221, 348–357):**
- Delete the `useEffect` that hardcodes `STATUS_84_COMPLETED` — the only producer of this status
- Delete the sub-cases closure status table section (L155–L205) — the `FRAUD_AND_AML` branch that renders child case statuses
- Delete the "Final Outcome locked to COMPLETED" display section (L208–L221)
- Delete the supervisor-only "Close Case (AFTER report)" button (L348–L357) gated on `caseData.type === 'FRAUD_AND_AML'`
- The standard outcome select (`STATUS_81/82/83`) applies to all cases after these deletions

**`CaseFilters.tsx` (Line 68):** Remove `{ value: 'STATUS_84_COMPLETED', label: 'Completed' }` from `statusOptions`.

**`casesTable.utils.ts` (Line 47):** Remove the `STATUS_84_COMPLETED: 'bg-green-50 text-green-700'` entry from `getStatusColor`. The `FRAUD_AND_AML` entry in `getTypeColor` (L57) can remain — alerts are still typed `FRAUD_AND_AML`; the removal is case-specific.

**`ViewCaseModal.tsx` (Lines 112, 200) and `CaseModalsManager.tsx` (Lines 565, 686):** Remove or simplify the `FRAUD_AND_AML` case-type branches that load sub-cases. After Track B, sub-case display is replaced by `InvestigationGroup` lookup if grouping display is still needed.

**`triage.types.ts` (Lines 23, 133, 193, 247):** Remove `parent_id?: number` from the Case type interface once all UI references to `parentId` / `parent_id` are gone.

### Data Migration (Step B8)

```sql
-- Phase 1: Backfill InvestigationGroup for all existing FRAUD_AND_AML containers
INSERT INTO investigation_groups (alert_id, tenant_id, created_at)
SELECT a.alert_id, c.tenant_id, NOW()
FROM cases c
JOIN alerts a ON a.case_id = c.case_id
WHERE c.case_type = 'FRAUD_AND_AML' AND c.parent_id IS NULL
ON CONFLICT (alert_id) DO NOTHING;

-- Phase 2: Reclassify container Case rows so STATUS_84 can be retired
UPDATE cases
SET status = 'STATUS_83_CLOSED_INCONCLUSIVE'
WHERE case_type = 'FRAUD_AND_AML' AND parent_id IS NULL;

-- Validation (must be 0 before dropping the enum value)
SELECT COUNT(*) FROM cases WHERE status = 'STATUS_84_COMPLETED';
```

Risk: Phase 2 is irreversible on the container rows (they move from `STATUS_84` to `STATUS_83`). Run on staging with a full data snapshot. Validate Phase 1 before Phase 2.

---

## Side Effects and Risks

### Risks of Track A Alone

| Risk | Likelihood | Mitigation |
|---|---|---|
| Report filter `caseType=FRAUD_AND_AML` returns zero results | Expected | Container rows are now excluded. Update the controller's filter option or add a note in the API docs. |
| `avgResolutionTime` changes for historical data | Expected / intended | The metric was previously distorted; the new value is more accurate. |
| Closed totals appear lower after Track A | Expected / intended | Container `STATUS_84` closes are no longer counted. This is correct. |

### Risks of Track B

| Risk | Likelihood | Mitigation |
|---|---|---|
| Existing `parent_id` data becomes orphaned | Low | Phase 1 migration creates `InvestigationGroup` records before Phase 2 reclassifies containers. Children's `parent_id` values point to now-`STATUS_83` rows but this is harmless (they are closed and excluded from queries). |
| `case-creation-approval.service.ts` has undiscovered FRAUD_AND_AML branches | Low | Verified at lines 84, 190, 477 — review each branch during implementation. |
| Tests that assert parent propagation behaviour | Expected | Any test that explicitly checks parent case status changes after child state transitions will fail. Each failing test confirms a removed behaviour — update fixtures, do not restore the logic. |
| Combined SAR/STR closure coordination is lost | Accepted | The issue design explicitly defers parallel BPMN orchestration to a follow-on feature. Track B makes FRAUD and AML cases independently closeable, which is correct for the interim. |
| Reopening a child case after both are closed | Low | Post-Track B, reopening follows the standard `REOPENABLE_CASE_STATUSES` path with no parent to update. No regression; the irreconcilable state (parent `STATUS_84_COMPLETED` with an active child) is eliminated. |

### Security Observation

The permission bypass in `case.repository.ts` is confirmed: any in-tenant user can read any `FRAUD_AND_AML` container case without owning it or having a task assigned. This is an access control gap, not merely a code smell. Track B closes it by removing the container row. If Track B is delayed, consider adding explicit roles/tenant-admin logic to the bypass rather than leaving it as a broad type-based exception.

### Cross-Issue Dependency

Issue #214 Track A and issue #220 Track B both modify `report.service.ts`. The #214 Track A adds `LEAF_CASE_FILTER` to `buildCommonCaseFilters`; the #220 Track B renames priority enum values in the same workload bucketing section (~lines 470–491). Recommended sequencing: merge #214 Track A first (lower risk, no migration), then merge #220 Track B with the already-present `LEAF_CASE_FILTER`.

---

## Effort Estimate

| Track | Files | Effort |
|---|---|---|
| Track A | 1 (`report.service.ts`) | 2–4 hours including test |
| Track B — schema + migration | 1 schema file + migration SQL | 0.5 day |
| Track B — `triage.service.ts` + `case-creation.service.ts` | 2 files | 2 hours |
| Track B — remove propagation (4 services) | 7 locations across 4 files | 0.5 day |
| Track B — permission bypass removal | 1 file | 1 hour |
| Track B — closure gate removal | 1 file | 1 hour |
| Track B — frontend (`CloseCaseModal`, `CaseFilters`, `casesTable.utils.ts`, `ViewCaseModal`, `CaseModalsManager`) | 5 files | 0.5 day |
| Track B — data migration + validation | SQL + QA | 0.5 day |
| Track B — `STATUS_84` retirement (Step B7) | 3 backend + 3 frontend files | 0.5 day |
| **Track A total** | | **~4 hours** |
| **Track B total** | | **~3–4 days** |

---

## Acceptance Criteria (Verification Checklist)

### Track A
- [ ] A `FRAUD_AND_AML` alert triaged and both children closed contributes 2 to `closedCases` in reports, not 3
- [ ] The case-type breakdown in reports shows `FRAUD` and `AML` buckets but no `FRAUD_AND_AML` bucket
- [ ] `closedCases` count equals the sum of `STATUS_81 + STATUS_82 + STATUS_83 + STATUS_71 + STATUS_72` outcomes
- [ ] `avgResolutionTime` reflects only actual investigation time (child cases), not container close time

### Track B
- [ ] A `FRAUD_AND_AML` alert triage creates exactly 2 `Case` rows (`FRAUD` and `AML`), both with `parent_id = null`
- [ ] An `InvestigationGroup` row is created with the correct `alert_id` and `tenant_id`
- [ ] No `Case` row with `case_type = FRAUD_AND_AML` exists in the database after migration
- [ ] Assigning a task to the `FRAUD` child case does not change the status of the `AML` child case
- [ ] Suspending the `FRAUD` child case does not change the status of any other case
- [ ] Closing the `FRAUD` child case with `STATUS_82` does not trigger any status update on the `AML` case
- [ ] A user who is not the owner or task-assignee of a `FRAUD` case receives a 403 when attempting to access it
- [ ] The close modal for a `FRAUD` case shows `STATUS_81 / STATUS_82 / STATUS_83` outcome options only — no `STATUS_84_COMPLETED`
- [ ] `STATUS_84_COMPLETED` does not appear in the case filter dropdown
- [ ] Zero `Case` rows with `status = STATUS_84_COMPLETED` exist after data migration

---

## Recommended Sequencing

1. **Ship Track A first** — one PR, one file, no migration. Stops report data corruption immediately.
2. **Validate staging data** before Track B — confirm the `parent_id` relationship structure in the production database before running the data migration.
3. **Ship Track B as a single coordinated release** — schema migration, service changes, permission fix, frontend cleanup, and data migration must land together. Do not merge partial Track B steps; the removal of the permission bypass and the closure gate require the container row to already be absent from new data.
4. **Defer `STATUS_84` retirement (Step B7)** until Track B is in production and the data migration validates zero rows — this is an irreversible enum change.
