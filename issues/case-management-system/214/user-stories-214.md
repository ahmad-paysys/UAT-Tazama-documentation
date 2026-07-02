# Issue #214 — User Stories: Fix FRAUD_AND_AML Container Cases Modelled as Peer Case Rows

**Assigned developers:** 2 developers (Dev A and Dev B)  
**Tracks:** Track A (reporting fix) → Track B (full re-model)  
**Merge order:** Track A ships first, independently. Track B follows. `STATUS_84` retirement is a separate step after Track B is in production and validated.

**Developer split:**  
- **Dev A:** US-214-01 (Track A), US-214-03 (triage & schema), US-214-05 (backend removal — permissions & closure)  
- **Dev B:** US-214-02 (data migration prep), US-214-04 (backend removal — status propagation & lifecycle), US-214-06 (frontend cleanup)

Stories within the same track that touch different files can be worked in parallel between Dev A and Dev B.

---

## US-214-01 — Fix Report Triple-Counting of FRAUD_AND_AML Alerts

**Title:** Fix reports so a single FRAUD_AND_AML alert counts as 2 cases, not 3

**Body:**  
Every FRAUD_AND_AML alert currently creates three database rows: a container parent (typed `FRAUD_AND_AML`) and two child rows (typed `FRAUD` and `AML`). All report queries count all three rows, so one alert appears as three cases in every report. Outcome totals become irreconcilable because the container's synthetic terminal status `STATUS_84_COMPLETED` sits outside the normal outcome breakdown.

The fix is a single constant added to `report.service.ts` that filters out container rows from all report queries. No schema change. No other files touched. Safe to ship immediately.

**Scope (Track A — one file):**  
- File: `backend/src/modules/report/report.service.ts`
- Add static constant: `private static readonly LEAF_CASE_FILTER = { NOT: { case_type: CaseType.FRAUD_AND_AML } }`
- Spread into `buildCommonCaseFilters()` — this propagates the filter to all report queries automatically
- Manually patch any direct queries that bypass the helper (two known locations)
- Remove `STATUS_84_COMPLETED` from the private `CLOSED_STATUSES` array and `STATUS_DISTRIBUTION_MAP` used for reports only (not from canonical constants — those remain until Track B)

**Acceptance Criteria:**
- [ ] A single FRAUD_AND_AML alert contributes exactly 2 to `totalCases` (the FRAUD child + the AML child)
- [ ] The case-type breakdown shows `FRAUD` and `AML` buckets; no `FRAUD_AND_AML` bucket appears in reports
- [ ] `closedCases` = sum of actual outcome statuses (STATUS_81 + STATUS_82 + STATUS_83 + STATUS_71 + STATUS_72) only
- [ ] `avgResolutionTime` is no longer distorted by instant container-row closes
- [ ] Investigator workload report does not attribute the ownerless container to any investigator
- [ ] All existing report tests pass; new test added for FRAUD_AND_AML alert scenario

**Testing:**
- Seed a staging database with one FRAUD_AND_AML alert (3 rows); run the report endpoint; verify `totalCases = 2`
- Verify the case-type breakdown has `FRAUD: 1, AML: 1` and no `FRAUD_AND_AML` entry
- Compare report totals before and after the change on a production-like dataset

---

## US-214-02 — Create `InvestigationGroup` Table and Data Migration

**Title:** Create the `InvestigationGroup` table and backfill existing FRAUD_AND_AML data

**Body:**  
Track B replaces the container `Case` row with a lightweight `InvestigationGroup` record that serves as a pure grouping mechanism. It has no status, no owner, no lifecycle — it just links an alert to its two child cases. This story adds the table and prepares the data migration.

This story can be worked in parallel with US-214-03. It does not change any application logic — it only creates the schema and migration scripts.

**Scope (Track B — schema only):**  
- Add `InvestigationGroup` model to `schema.prisma`:
  ```
  id          Int      @id @default(autoincrement())
  alert_id    String   @unique
  tenant_id   String
  created_at  DateTime @default(now())
  ```
  No status. No owner. No foreign key to Case.
- Write a Prisma migration file
- Write a data backfill script:
  - For every existing `Case` row with `case_type = FRAUD_AND_AML`: create an `InvestigationGroup` with the same `alert_id` and `tenant_id`
  - Then reclassify the container `Case` row to `status = STATUS_83_CLOSED_INCONCLUSIVE` (not deleted — kept for audit history)
- Write a validation query: `SELECT count(*) FROM cases WHERE case_type = 'FRAUD_AND_AML' AND status != 'STATUS_83_CLOSED_INCONCLUSIVE'` should return 0 after backfill

**Acceptance Criteria:**
- [ ] `InvestigationGroup` table exists in the schema with correct columns
- [ ] Backfill creates one `InvestigationGroup` row per existing FRAUD_AND_AML container case
- [ ] After backfill, all container `Case` rows have `status = STATUS_83_CLOSED_INCONCLUSIVE`
- [ ] Validation query returns 0
- [ ] Migration is reversible (down migration documented even if not automated)
- [ ] No application code changes in this story — schema and data only

**Testing:**
- Run migration + backfill on a copy of production data; verify counts match (one `InvestigationGroup` per FRAUD_AND_AML alert)
- Run the validation query; confirm it returns 0
- Verify existing child `Case` rows (`FRAUD`, `AML` typed) are untouched

---

## US-214-03 — Stop Creating the Container Case Row at Triage

**Title:** Update triage service to create an `InvestigationGroup` instead of a container `Case` row for FRAUD_AND_AML alerts

**Body:**  
When a FRAUD_AND_AML alert arrives, `triage.service.ts` currently creates three `Case` rows. After this change it will create two `Case` rows (FRAUD child, AML child) and one `InvestigationGroup` record. The container `Case` row is no longer created.

This is the central change of Track B. It must be merged after US-214-02 (the `InvestigationGroup` table must exist first).

**Scope (Track B — triage service):**  
- File: `backend/src/services/triage.service.ts` (lines 372 and 612 are the two creation branches)
- Replace `createContainerCase(FRAUD_AND_AML, ...)` with `createInvestigationGroup(alert_id, tenant_id)`
- Call `createCaseWithInvestigationTask(FRAUD, userId, tenantId, parentCaseId: null, ...)` — no parent ID
- Call `createCaseWithInvestigationTask(AML, userId, tenantId, parentCaseId: null, ...)` — no parent ID
- File: `backend/src/services/case-creation.service.ts`
  - Change `parentCaseId` parameter type from `number` to `number | null`
- Both child cases now have `parent_id = null`

**Acceptance Criteria:**
- [ ] A new FRAUD_AND_AML alert creates exactly 2 `Case` rows (typed `FRAUD` and `AML`) and 1 `InvestigationGroup` row
- [ ] Neither child `Case` row has a `parent_id` set
- [ ] No `Case` row with `case_type = FRAUD_AND_AML` is created by the new code path
- [ ] Both child cases are created with a real investigator owner (not null)
- [ ] Existing alerts (pre-migration) are not affected — their data was handled by US-214-02

**Testing:**
- Submit a FRAUD_AND_AML alert in staging; verify exactly 2 Case rows and 1 InvestigationGroup row in the DB
- Verify both Case rows have `parent_id = null`
- Verify both Case rows follow the normal investigation BPMN path (no special handling required)

---

## US-214-04 — Remove All Seven Status Propagation Blocks

**Title:** Delete the seven status propagation blocks that keep the container case status in sync with its children

**Body:**  
Because the container case has no real lifecycle, 7 places in the codebase manually propagate child case status changes up to the parent. These blocks are: `promoteParentCaseToInProgress` in `task.service.ts`, three propagation blocks in `task-lifecycle.service.ts`, and two each in `case.service.ts` and `case-reopening.service.ts`. After the container row is removed (US-214-03), these blocks have no purpose and will error if left in place.

This story can be worked in parallel with US-214-05 since they touch different files.

**Scope (Track B — propagation removal):**  
- File: `backend/src/services/task.service.ts` (lines 303–353)
  - Delete `promoteParentCaseToInProgress` method entirely (no callers will remain)
  - Remove all calls to this method
- File: `backend/src/services/task-lifecycle.service.ts` (lines 65–81, 168–184, 273–292)
  - Delete all three `if (updatedCase.parent_id)` blocks
- File: `backend/src/services/case.service.ts` (lines 90–107, 229–246)
  - Delete both parent-status propagation blocks from suspend and resume handlers
- File: `backend/src/services/case-reopening.service.ts` (lines 68–73, 244–249)
  - Delete both parent-status propagation blocks from the reopen handler

**Acceptance Criteria:**
- [ ] No `if (updatedCase.parent_id)` blocks exist anywhere in the codebase after this change
- [ ] `promoteParentCaseToInProgress` method is deleted
- [ ] Closing a FRAUD child case does NOT change the status of the AML child case (and vice versa)
- [ ] Suspending a case does NOT affect any sibling case
- [ ] All task lifecycle tests pass
- [ ] TypeScript compilation succeeds with no errors

**Testing:**
- Close the FRAUD child of a FRAUD_AND_AML alert; verify the AML child is unchanged
- Suspend a case; verify no side effects on related cases
- Reopen a case; verify no propagation to other cases

---

## US-214-05 — Remove Permission Bypass and Closure Gate for Container Cases

**Title:** Remove the FRAUD_AND_AML permission bypass and the supervisor-only closure gate

**Body:**  
Because the container case has no owner, `case.repository.ts` contains a special OR-branch that grants any in-tenant user access to `FRAUD_AND_AML` typed cases, bypassing standard owner/assignee permission checks. Separately, `case-closure-approval.service.ts` has a special gate that prevents closing either child independently — both must be closed before a supervisor can close the container. Both of these workarounds become dead code once the container row is gone.

This story can be worked in parallel with US-214-04.

**Scope (Track B — permission and closure):**  
- File: `backend/src/repositories/case.repository.ts` (lines 185–196)
  - Remove the `OR: [{ case_type: { in: ['FRAUD_AND_AML'] } }, ...]` branch
  - Only standard owner/assignee permission checks remain
- File: `backend/src/services/case-closure-approval.service.ts` (lines 112–146)
  - Delete `isFraudAndAmlCase` check and the supervisor-only closure requirement
  - All cases now follow the single standard closure path

**Acceptance Criteria:**
- [ ] A user without ownership or assignment of a case receives a 403 when accessing it — regardless of `case_type`
- [ ] The `FRAUD_AND_AML` type no longer receives special permission treatment
- [ ] Investigators can close FRAUD and AML child cases independently, without waiting for a sibling
- [ ] No `isFraudAndAmlCase` logic exists anywhere in the codebase
- [ ] Permission tests pass: unauthorized access returns 403

**Testing:**
- Log in as a user not assigned to a FRAUD or AML case; attempt to access it; verify 403
- Close a FRAUD child case independently (without the AML child being closed); verify it succeeds
- Verify standard closure flow works end-to-end for both child case types

---

## US-214-06 — Clean Up Frontend for Removed Container Case Logic

**Title:** Remove all FRAUD_AND_AML container-specific UI code from the frontend

**Body:**  
The frontend has several components with hardcoded special handling for the container case: `CloseCaseModal.tsx` has a hardcoded `STATUS_84_COMPLETED` override, `CaseFilters.tsx` lists "Completed" as a status option, `casesTable.utils.ts` has a badge colour for `STATUS_84`, and `ViewCaseModal.tsx` / `CaseModalsManager.tsx` have FRAUD_AND_AML branches. All of these become dead code once the container row is removed.

**Scope (Track B — frontend cleanup):**  
- File: `frontend/src/components/CloseCaseModal.tsx`
  - Delete the `if (caseData?.type === 'FRAUD_AND_AML')` block that sets `STATUS_84_COMPLETED`
  - Delete the sub-cases display section
  - Delete the supervisor-only close button for container cases
  - Close modal now shows standard `STATUS_81/82/83` options only
- File: `frontend/src/components/CaseFilters.tsx`
  - Remove "Completed" (`STATUS_84`) from the status filter dropdown
- File: `frontend/src/utils/casesTable.utils.ts`
  - Remove `STATUS_84_COMPLETED` badge colour entry
- File: `frontend/src/components/ViewCaseModal.tsx`
  - Remove any FRAUD_AND_AML-specific branches
- File: `frontend/src/components/CaseModalsManager.tsx`
  - Simplify or remove FRAUD_AND_AML modal routing branches
- File: `frontend/src/types/triage.types.ts`
  - Remove `parent_id` from the `Case` interface

**Acceptance Criteria:**
- [ ] Close modal shows only `STATUS_81`, `STATUS_82`, `STATUS_83` outcome options — no `STATUS_84`
- [ ] Status filter dropdown has no "Completed" option
- [ ] No `STATUS_84` badge colour or label appears anywhere in the UI
- [ ] No `parent_id` field on the frontend `Case` type
- [ ] No TypeScript errors in the frontend build
- [ ] All frontend component tests pass with updated fixtures (~30–40 test fixtures updated)

**Testing:**
- Open the close case modal for a FRAUD and for an AML case; verify standard outcome options only
- Check the case filter dropdown; verify no STATUS_84 option
- Verify no console errors or TypeScript errors in the build
- Verify the case list renders correctly for FRAUD and AML typed cases

---

## US-214-07 — Retire `STATUS_84_COMPLETED` from All Canonical Constants

**Title:** Remove `STATUS_84_COMPLETED` from canonical constants, schema, and database after Track B is validated in production

**Body:**  
`STATUS_84_COMPLETED` is a synthetic terminal status that only exists because container cases had no real lifecycle. Once Track B is in production and the data validation confirms zero Case rows with `STATUS_84`, this value can be permanently retired from all canonical definitions.

**This story must NOT be started until:**
1. Track B is deployed to production
2. The validation query `SELECT count(*) FROM cases WHERE status = 'STATUS_84_COMPLETED'` returns 0

**Scope (post-Track-B cleanup):**  
- File: `backend/src/constants/case.constants.ts` — remove `STATUS_84_COMPLETED`
- File: `backend/src/enums/case-enum.ts` — remove `STATUS_84` value
- File: `backend/prisma/schema.prisma` — remove `STATUS_84_COMPLETED` from the `CaseStatus` enum
- Write and run a Prisma migration to drop the enum value from PostgreSQL
- Confirm no application code references `STATUS_84` after removal (TypeScript compiler will catch this)

**Acceptance Criteria:**
- [ ] Validation query confirms 0 rows with `STATUS_84_COMPLETED` before any code change
- [ ] `STATUS_84_COMPLETED` does not exist in `case.constants.ts`, `case-enum.ts`, or `schema.prisma`
- [ ] Database migration removes the enum value cleanly
- [ ] TypeScript compilation passes with zero errors after removal
- [ ] No frontend component references `STATUS_84` in any form

**Testing:**
- Run validation query before starting; confirm 0 rows
- After migration: `SELECT enum_range(NULL::case_status)` in PostgreSQL should not include `STATUS_84_COMPLETED`
- Full frontend build with no TypeScript errors
- Smoke test: create a FRAUD_AND_AML alert, close both children — system behaves correctly with no STATUS_84 references

---

## US-214-08 — End-to-End Integration Testing for Issue #214

**Title:** Full regression and integration test pass for Issue #214 after all stories are merged

**Body:**  
This story is to be completed after all other US-214 stories (through US-214-07) are merged and deployed to the staging environment. It validates the complete system behaviour end-to-end, covering both Track A and Track B changes across backend, frontend, and data.

**Scope:**  
This is a testing-only story — no code changes.

**Test scenarios to cover:**

*Reporting (Track A):*
- Submit one FRAUD_AND_AML alert; verify `totalCases` in reports = 2 (not 3)
- Verify case-type breakdown shows only `FRAUD` and `AML` buckets in reports
- Verify `closedCases` only counts actual investigation outcomes

*New case creation (Track B):*
- Submit a FRAUD_AND_AML alert; verify exactly 2 Case rows + 1 InvestigationGroup row in the database
- Verify both Case rows have `parent_id = null`
- Verify both Case rows have a real `case_owner_user_id` (not null)
- Verify the InvestigationGroup row has the correct `alert_id` and `tenant_id`

*Independent lifecycle (Track B):*
- Assign a FRAUD child case and update its status; verify the AML child case is completely unaffected
- Close the FRAUD child case; verify the AML child case is still open and independently actionable
- Suspend the AML child case; verify no side effects on FRAUD child

*Permissions (Track B):*
- Attempt to access a FRAUD child case as a user without ownership or assignment; verify 403
- Attempt to access an AML child case as an unrelated user; verify 403

*Closure (Track B):*
- Close the FRAUD child using standard STATUS_81/82/83 options; verify it succeeds independently
- Verify no STATUS_84 modal option appears anywhere in the close case workflow

*Frontend (Track B):*
- Verify no STATUS_84 badge, filter option, or modal reference exists anywhere in the UI
- Verify no `parent_id` field appears on Case objects in network responses

*Data integrity (Track B):*
- Run `SELECT count(*) FROM cases WHERE case_type = 'FRAUD_AND_AML'` — should return 0 for new cases post-migration
- Run `SELECT count(*) FROM cases WHERE status = 'STATUS_84_COMPLETED'` — should return 0 after US-214-07

**Acceptance Criteria:**
- [ ] All test scenarios above pass on staging
- [ ] Reports reconcile correctly: `totalCases = closedCases + openCases`
- [ ] No STATUS_84 in database, code, or UI
- [ ] No FRAUD_AND_AML container rows in the database (post-migration)
- [ ] Full frontend build with zero TypeScript errors
- [ ] No regression in single-type case (FRAUD-only or AML-only alert) workflow
- [ ] Performance: triage of a FRAUD_AND_AML alert completes in under 5 seconds
