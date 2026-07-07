> **Prompt used for this review:**
>
> `read pull-requests.md and process this PR: https://github.com/tazama-lf/case-management-system/pull/233 in file CMS-PR-233-claude.md`
> `include this prompt at the top of the file. do not refer to any other md files for this task.`

# PR Review: CMS #233 — fix: Replace FRAUD_AND_AML container case with InvestigationGroup linkage

**Repo:** tazama-lf/case-management-system
**Branch:** `paysys/fraud_and_aml` → `dev`
**Author:** sobia-rizwan1567 (Sobia Rizwan)
**Date Reviewed:** 2026-07-07
**Label:** bug
**Size:** +2,709 / -2,212 lines across 71 files
**Commits:** multiple (latest reviewed: `d5902b5f`; head reviewed by CodeRabbit last: `0de99991`)
**State:** OPEN
**Existing approvals:** none (CodeRabbit — COMMENTED across 4 review passes)

---

## Table of Contents

- [Overview](#overview)
- [What Changed (Detailed)](#what-changed-detailed)
  - [1. Prisma schema + migrations — introduce `InvestigationGroup`, drop `STATUS_84_COMPLETED`](#1-prisma-schema--migrations)
  - [2. Backfill script — historical FRAUD_AND_AML container conversion](#2-backfill-script)
  - [3. `InvestigationGroupService` and module wiring](#3-investigationgroupservice-and-module-wiring)
  - [4. Alert repository — grouped alert/case resolution](#4-alert-repository)
  - [5. Case creation split — FRAUD + AML siblings via group](#5-case-creation-split)
  - [6. Case closure/reopening/task-lifecycle simplification](#6-case-closurereopeningtask-lifecycle-simplification)
  - [7. Case service — complete-case flow and grouped alert linking](#7-case-service--complete-case-flow-and-grouped-alert-linking)
  - [8. Triage service — FRAUD+AML split and rollback](#8-triage-service)
  - [9. Reporting, RBAC, constants cleanup](#9-reporting-rbac-constants-cleanup)
  - [10. Frontend — related-case flow, `groupId` migration, UI cleanups](#10-frontend)
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

This PR removes the FRAUD_AND_AML *container* case model that previously represented a synthetic parent case pointing at FRAUD and AML sub-cases through `Case.parent_id`. In its place it introduces an `InvestigationGroup` — a lightweight join record keyed by `alert_id` that binds sibling FRAUD and AML cases through `Case.group_id`, and it removes the associated `STATUS_84_COMPLETED` container status. The change touches roughly the entire vertical slice for FRAUD_AND_AML flows: Prisma schema, two migrations plus a backfill SQL script, alert/case repositories, case creation/closure/reopening/triage services, reporting, RBAC, and a broad set of frontend components that used to render the parent/sub-case relationship.

The motivation stated in the PR description is to eliminate a synthetic container case that was propagating status changes across siblings, bypassing permission checks, producing triple-counted reporting rows, and requiring container-specific UI. Coordination between the FRAUD and AML sibling cases now happens exclusively through the shared `InvestigationGroup` (and, on alerts, the `related_case_id`/`related_case_type` fields). The deployment note calls out an important non-atomic dependency: migrations, then the `backfill-investigation-groups.sql` transactional script, then code deployment; the enum-swap migration is not safe under concurrent writes and requires a maintenance window.

The PR is large but internally coherent — every change traces back to the same refactor. The one clearly unrelated concern bundled in is a set of task-lifecycle changes for `SAR/STR Filing` and reporting filter refinements, both of which the description does not explicitly discuss.

| File (grouped) | Nature of Change |
|------|-----------------|
| `backend/prisma/schema.prisma` | Adds `InvestigationGroup` model + `@relation`s; adds `Case.group_id` + `@@index([group_id])`; retires container-only fields. |
| `backend/prisma/migrations/20260701000001_add_investigation_group/*` | Creates `investigation_groups` table, unique index on `alert_id`, adds `cases.group_id` column. |
| `backend/prisma/migrations/20260702120000_remove_case_status_completed/*` | Enum swap that drops `STATUS_84_COMPLETED`. |
| `backend/prisma/migrations/20260702140000_add_investigation_group_relations/*` | Follow-up: adds FK constraints and `cases_group_id_idx`. |
| `backend/prisma/scripts/backfill-investigation-groups.sql` | Transactional backfill of existing FRAUD_AND_AML containers/subcases into groups. |
| `backend/src/modules/investigation-group/*` | New module: `InvestigationGroupService` — create/find/delete group records. |
| `backend/src/modules/repository/alert.repository.ts` | Adds `getGroupedCasesForAlert`; rewrites `getAlertByCaseId` to fall back through `investigationGroup` and to scope by `tenant_id`. |
| `backend/src/modules/case/services/case-creation.service.ts` | Creates FRAUD + AML siblings inside a single `createCaseWithInvestigationTask` path; uses `CANDIDATE_GROUPS.INVESTIGATIONS`. |
| `backend/src/modules/case/services/case-creation-approval.service.ts` | Splits the container case at approval time into two sibling investigations sharing a group. |
| `backend/src/modules/case/services/case-closure-approval.service.ts` | Guards against missing investigation task; removes container-status propagation. |
| `backend/src/modules/case/services/case-reopening.service.ts` | Drops parent-case propagation. |
| `backend/src/modules/case/case.service.ts` | Rewires `getAlertByCaseId` usage; consolidates alert `caseId` update on complete-case; adds AML sibling creation on supervisor approve. |
| `backend/src/modules/task/services/task-lifecycle.service.ts` | Gates both `tx.case.update` **and** `handleCaseStatusChanged` behind `updatedTask.name !== 'SAR/STR Filing'` on unassign. |
| `backend/src/modules/triage/triage.service.ts` | FRAUD+AML manual/AI triage now creates group + two cases in a transaction; captures `detachedAlert` from final `updateAlert`. |
| `backend/src/modules/report/report.service.ts` | Excludes containers, uses grouped cases in workload/ageing calculations. |
| `backend/src/modules/repository/case.repository.ts` | `findCaseWithPermissionCheck` + list queries now aware of `group_id`. |
| `backend/src/utils/rbac/permissionMatrix.json`, `backend/src/utils/enums/case-enum.ts`, `backend/src/constants/case.constants.ts` | Removes `STATUS_84_COMPLETED` + related permissions. |
| `frontend/src/features/cases/**` | Removes parent/sub-case props; renders `groupId` where relevant. `CloseCaseModal` supervisor path is now report-only. |
| `frontend/src/features/alerts/**` | Adds `related_case_id`/`related_case_type` on `AlertDetails`; `AlertsDetailModal` adds refresh + related-case invalidation. |
| `frontend/src/features/cases/components/view/CaseDetailsTab.tsx` | Renders group badge — hardcoded label `"FRAUD_AND_AML"`. |
| Test files (backend + frontend, ~20 files) | Rewritten to cover grouped-case flows; new `investigation-group.service.spec.ts`. |

[↑ Back to top](#pr-review-cms-233--fix-replace-fraud_and_aml-container-case-with-investigationgroup-linkage)

---

## What Changed (Detailed)

### 1. Prisma schema + migrations

`backend/prisma/schema.prisma` — new model and relation:

```prisma
model Case {
  ...
  group_id             Int?
  investigationGroup   InvestigationGroup?  @relation(fields: [group_id], references: [id])

  @@index([group_id])
  @@map("cases")
}

model Alert {
  ...
  case_id            Int?                @unique
  case               Case?               @relation(fields: [case_id], references: [case_id])
  investigationGroup InvestigationGroup?
  @@map("alerts")
}
```

Two migrations are shipped:

1. `20260701000001_add_investigation_group` — creates `investigation_groups` table (unique on `alert_id`) and adds `cases.group_id` column (nullable, no FK yet).
2. `20260702140000_add_investigation_group_relations` — formalises the relations, adds `cases_group_id_idx` and the FK constraint `cases_group_id_fkey ON DELETE SET NULL`.

The `STATUS_84_COMPLETED` enum value is removed in `20260702120000_remove_case_status_completed` via an enum swap. The migration is explicitly documented as unsafe under concurrent writes.

### 2. Backfill script

`backend/prisma/scripts/backfill-investigation-groups.sql` — a single `BEGIN … COMMIT` block that:
- Inserts one `investigation_groups` row per historical FRAUD_AND_AML container case (keyed on `alert_id`).
- Sets `cases.group_id` on sub-cases pointing to their originating container's newly created group id.
- Nulls out `alerts.case_id` for FRAUD_AND_AML alerts (link now flows through `investigation_groups`).
- Leaves `cases.parent_id` intact for the container row so the historical link isn't lost until a follow-on migration removes it.

Deployment order (from the PR description): **migrations → backfill script → code**. This is important — deploying the code before the backfill leaves grouped-case flows returning empty siblings for historical rows.

### 3. `InvestigationGroupService` and module wiring

`backend/src/modules/investigation-group/investigation-group.service.ts` — thin service backed by Prisma. Exposes:
- `createInvestigationGroup(alertId, tenantId, tx?)` — enforces the `alert_id` unique constraint.
- `getInvestigationGroupByAlertId(alertId, tenantId, tx?)`.
- `deleteInvestigationGroup(id, tx?)` — used in the AML rollback path in triage.

The module is imported by `case.module.ts` and `triage.module.ts`.

### 4. Alert repository

`backend/src/modules/repository/alert.repository.ts` — `getAlertByCaseId` now correctly scopes by `tenant_id` and falls back through the group:

```ts
async getAlertByCaseId(caseId: number, tenantId: string, tx?: Prisma.TransactionClient): Promise<number> {
  const client = tx ?? this.prisma;
  const caseRecord = await client.case.findFirst({
    where: { case_id: caseId, tenant_id: tenantId },
    select: {
      alert: { select: { alert_id: true } },
      investigationGroup: { select: { alert_id: true } },
    },
  });
  const alertId = caseRecord?.alert?.alert_id ?? caseRecord?.investigationGroup?.alert_id;
  if (alertId) return alertId;
  throw new NotFoundException(`Alert with Case ID ${caseId} not found`);
}
```

And a new `getGroupedCasesForAlert(alertId, tenantId, tx?)` returns every case in an alert's `InvestigationGroup`. This is the primary resolution path used everywhere the code previously relied on `parent_id → children`.

The previously flagged `getAlertIdsWithInvestigationGroup()` helper that materialised alert IDs into memory has been removed from the repository entirely.

### 5. Case creation split

`case-creation.service.ts::createCaseWithInvestigationTask` — creates a case together with an `INVESTIGATE_CASE` task, using `CANDIDATE_GROUPS.INVESTIGATIONS`:

```ts
const task = await this.taskService.createTask(
  {
    caseId: createdCase.case_id,
    status: TaskStatus.STATUS_01_UNASSIGNED,
    name: TASK_NAMES.INVESTIGATE_CASE,
    description: `Task to investigate: ${createdCase.case_id}`,
    candidateGroup: CANDIDATE_GROUPS.INVESTIGATIONS,
  },
  ...
);
```

`case-creation-approval.service.ts` handles the supervisor approval path: on FRAUD_AND_AML, it converts the pending case into a FRAUD investigation, creates an AML sibling, links both to a shared `InvestigationGroup`, and detaches the original alert's `case_id` (correctly passing `null` where earlier drafts passed `undefined`).

### 6. Case closure/reopening/task-lifecycle simplification

`case-closure-approval.service.ts` now guards the investigation-task lookup:

```ts
const investigationTasks = caseData.tasks
  .filter(t => t.name === TASK_NAMES.INVESTIGATE_CASE &&
              (t.status === TaskStatus.STATUS_20_IN_PROGRESS ||
               t.status === TaskStatus.STATUS_30_COMPLETED))
  .sort((a, b) => new Date(b.created_at).getTime() - new Date(a.created_at).getTime());
const investigationTask: Task | null = investigationTasks.at(0) ?? null;
if (!investigationTask) {
  throw new NotFoundException({ message: 'Investigation task not found or not in a closeable state', ... });
}
```

`task-lifecycle.service.ts::unassign` gates both the case-status write **and** the `handleCaseStatusChanged` call by the SAR/STR filing task name:

```ts
if (updatedTask.name !== 'SAR/STR Filing') {
  await tx.case.update({ ... status: STATUS_02_READY_FOR_ASSIGNMENT ... });
  await this.flowableService.handleCaseStatusChanged({ caseId: existingTask.case_id, ... });
}
```

`case-reopening.service.ts` drops parent-case propagation entirely.

### 7. Case service — complete-case flow and grouped alert linking

`case.service.ts::completeCaseCreation` now issues a single alert update whose `caseId` is either the current case id or `null` for the FRAUD_AND_AML supervisor path:

```ts
const alertId = await this.alertRepository.getAlertByCaseId(caseId, tenantId, prisma);
if (alertId) {
  await this.alertRepository.updateAlert(
    alertId,
    {
      priority_score: updateData.priorityScore,
      priority: updateData.priority,
      alertType: updateData.caseType,
      predictionOutcome: updateData.predictionOutcome,
      confidencePer: updateData.confidence,
      caseId: isFraudNAML && isSupervisor ? null : caseId,
    } as unknown as UpdateAlertDTO,
    prisma,
  );
}

let amlCase = null;
if (isFraudNAML && isSupervisor) {
  amlCase = await this.caseCreationService.createCaseWithInvestigationTask(
    CaseType.AML, userId, existingCase.tenant_id, updatedCase.priority,
    CaseCreationType.AUTOMATIC_SYSTEM, role, investigationGroup!.id, prisma,
  );
}
```

The prior double-write (`case_id` field first, then a follow-up nulling call for the FRAUD path) has been consolidated into one call using the correct `caseId` field.

### 8. Triage service

`triage.service.ts` — both manual and AI triage now build FRAUD+AML pairs through the group. The final `updateAlert` result is captured:

```ts
const detachedAlert = await this.alertService.updateAlert(
  alertId,
  userId,
  { caseId: null } as unknown as UpdateAlertDTO,
  tx,
);
return { alert: detachedAlert, completeNewCaseTask, existingCase };
```

The FRAUD+AML branch runs inside `alertRepository.transaction`, including `createInvestigationGroup` and both `createCaseWithInvestigationTask` calls. A dedicated rollback test (`should rollback FRAUD investigation on AML failure`) exercises the failure path.

### 9. Reporting, RBAC, constants cleanup

- `report.service.ts`: excludes container rows from workload/ageing (container rows no longer exist post-backfill; the filter also protects any residual rows). Report status filter no longer materialises alert ID lists.
- `permissionMatrix.json`: removes `STATUS_84_COMPLETED` entries.
- `case.constants.ts`, `case-enum.ts`: enum value and any references removed.

### 10. Frontend

- `AlertDetailsResponse.dto.ts` (backend) + `triage.types.ts` (frontend) add `related_case_id?: number | null` and `related_case_type?: string | null`.
- `AlertsDetailModal.tsx` — refreshes on prop change, invalidates both `['case', case_id]` and `['case', related_case_id]` on close, renders related-case status + navigation. Both backdrop and close-button handlers duplicate identical invalidation logic.
- `AlertsDashboard.tsx` — reloads alert on modal-triggered refresh.
- `CaseModalsManager.tsx`, `ViewCaseModal.tsx`, `CaseDetailsTab.tsx`, `CaseActionsPanel.tsx`, `LinkedItemsTab.tsx`, `TasksDetailsModal.tsx` — all sub-case/parent-case props deleted; `groupId` used where a linked case must be shown.
- `CaseDetailsTab.tsx` — group badge renders literal `"FRAUD_AND_AML"` whenever `row.groupId` is set.
- `CloseCaseModal.tsx` — the supervisor footer now offers only "Generate Investigation Report" (gated on `!reportApproved`); no direct supervisor submit button.
- `casesTable.utils.ts`, `caseService.ts`, `useCase.ts` — `parent_id`/`parentId` → `group_id`/`groupId`; `getSubCasesDetails` removed.

[↑ Back to top](#pr-review-cms-233--fix-replace-fraud_and_aml-container-case-with-investigationgroup-linkage)

---

## Code Quality Analysis

### Strengths

- **The refactor is coherent end-to-end.** The synthetic container case is replaced consistently from the DB through service layers and into the UI — no half-migrated call sites were found on spot checks.
- **Migrations are honest about their operational risk.** The enum-swap migration explicitly calls out the maintenance-window requirement; the follow-on relation migration adds the FK + index that the first migration deliberately deferred.
- **Backfill is transactional and idempotent-friendly.** The `BEGIN … COMMIT` block keeps historical data consistent; it does not depend on ordering with other migrations because the schema pieces exist first.
- **Tenant scoping was tightened.** `getAlertByCaseId` used to lean on `alert.findUnique` / `case.findUnique` without tenant scope; the new implementation filters by `tenant_id` in the single `findFirst`.
- **Multi-round CodeRabbit feedback was actively worked in.** Almost every Major CodeRabbit item from the initial pass was resolved in the follow-up commits (see [CodeRabbit Activity](#coderabbit-activity)).
- **Rollback test coverage exists for the two-step create path.** `triage.service.spec.ts` has an explicit "should rollback FRAUD investigation on AML failure" test — the exact scenario CodeRabbit flagged as risky.
- **The `investigationTask` null guard is present** with a proper `NotFoundException` in `case-closure-approval.service.ts`, plus a dedicated regression spec — this was CodeRabbit's Major-severity concern and it is now covered by both code and test.

### Issues and Observations

#### Issue 1 — `TaskLogTab` passes `onSwitchToCaseDetails` but neither declares nor forwards it

**Severity: Major (Bug, type safety)**

`frontend/src/features/cases/components/ViewCaseModal.tsx:189` still passes `onSwitchToCaseDetails={() => { setTab('details'); }}` to `TaskLogTab`, but `TaskLogTab`'s prop interface no longer includes it:

```ts
// frontend/src/features/cases/components/view/TaskLogTab.tsx:39-51
interface TaskLogTabProps {
  caseId: number;
  caseStatus?: string;
  onRefreshCases?: () => Promise<void>;
  alertId?: number;
  canManageSupervisorActions?: boolean;
  caseData?: any;
  onApproveCase?: (caseData: any) => void;
  onApproveCaseCreation?: (caseData: any) => void;
  onRejectCaseCreation?: (caseData: any) => void;
  onAbandonCase?: (caseData: any) => void;
  onAfterTaskReassign?: () => void;
}
```

`TasksDetailsModal.tsx` (`TaskDetailsModalProps` at lines 21-28) also no longer accepts this prop. Under `strict` TypeScript this is a hard type error at the `ViewCaseModal` call site; even if the build tolerates it, the callback is unreachable — clicking whatever previously triggered "switch to case details" now does nothing.

**Fix:** either drop the `onSwitchToCaseDetails` prop at the `ViewCaseModal.tsx:189` call site (if the "switch to details" affordance was intentionally removed with the sub-case UI), or re-add the prop to `TaskLogTabProps` and forward it into `TaskDetailsModal`. Given the rest of the sub-case flow was intentionally removed, dropping the prop is the likely intent.

#### Issue 2 — `getUserCasesByUserId` contains an empty conditional block

**Severity: Minor (Code Quality)**

`backend/src/modules/case/case.controller.ts:512-518`:

```ts
const { userId: requestingUserId, tenantId } = extractUserData(req);
if (requestingUserId !== targetUserId) {
  /* empty */
}
return await this.caseService.getUserCases(targetUserId, query, tenantId);
```

`@RequireSupervisorRole()` already gates the route; the conditional does nothing. Either remove the block and the `requestingUserId` destructuring (keep `tenantId`), or fill in the intended check (e.g. audit-log a supervisor viewing another user's queue).

#### Issue 3 — `CaseDetailsTab` group badge is a literal, not a data-driven value

**Severity: Minor (Maintainability)**

`frontend/src/features/cases/components/view/CaseDetailsTab.tsx:467-493` renders the string literal `"FRAUD_AND_AML"` whenever `row.groupId` is set. FRAUD_AND_AML is the only group type today, so this is not wrong — but it silently misrepresents any future group type. Either surface a `group_type` from the API or derive it from the sibling cases' `case_type` values.

#### Issue 4 — `AlertsDetailModal` duplicates close/invalidate logic between backdrop and button

**Severity: Minor (Maintainability)**

`AlertsDetailModal.tsx` — both the backdrop `onClick` (around line 500-510) and the close-button `onClick` (around line 520-530) repeat identical `queryClient.invalidateQueries` calls for `case_id` and `related_case_id`, followed by `onAlertUpdated?.()` and `onClose()`. Extract into a shared `handleModalClose` to avoid drift:

```tsx
const handleModalClose = () => {
  if (alert?.case_id) queryClient.invalidateQueries({ queryKey: ['case', alert.case_id] });
  if (alert?.related_case_id) queryClient.invalidateQueries({ queryKey: ['case', alert.related_case_id] });
  onAlertUpdated?.();
  onClose();
};
```

#### Issue 5 — `triage.types.ts` types `related_case_type` as `string`

**Severity: Informational (Type safety)**

`frontend/src/features/alerts/types/triage.types.ts:179-180`:

```ts
related_case_type?: string | null;
```

`CaseType` is already used for `case_type` on the same interface. Tightening to `CaseType | null` prevents drift from the backend enum.

#### Issue 6 — `CloseCaseModal` supervisor path has no non-report close action

**Severity: Informational (UX / Product decision)**

`frontend/src/features/cases/components/CloseCaseModal.tsx:221-245` — non-supervisors get a "Submit for Approval" button; supervisors get *only* a "Generate Investigation Report" button gated on `!reportApproved`. There is no supervisor equivalent of a direct close/submit. If this is intentional (the report generation modal itself performs the close on approval), it should be documented; if not, the supervisor closure path is missing.

#### Issue 7 — Enum-swap migration is not safe under concurrent writes

**Severity: Informational (Operational)**

`backend/prisma/migrations/20260702120000_remove_case_status_completed` — the PR description already flags this: it requires a maintenance window. The observation here is only that this constraint must reach whoever runs the deploy — it is easy to miss in a 71-file PR.

[↑ Back to top](#pr-review-cms-233--fix-replace-fraud_and_aml-container-case-with-investigationgroup-linkage)

---

## Security Assessment

| Concern | Assessment |
|---------|-----------|
| Tenant isolation on grouped case lookups | **Improved.** `getAlertByCaseId` and `getGroupedCasesForAlert` both filter `case`/`investigationGroup` by `tenant_id`. Was previously relying on a `findUnique` by primary key with no tenant check. |
| Auth/RBAC on new endpoints | No new endpoints. `permissionMatrix.json` narrows on the removal of `STATUS_84_COMPLETED`; no permission was widened. |
| Alert/case link integrity across `caseId` / `case_id` naming | **Fixed by this PR.** The prior mixed `case_id` / `caseId` write on complete-case has been consolidated into a single `updateAlert` using the correct `caseId` field. |
| Cross-tenant leakage in related-case navigation (new UI) | The frontend renders `related_case_id` returned by the backend; backend joins already tenant-scope the lookup. No cross-tenant surface introduced. |
| SQL injection / string interpolation | Backfill script uses parameterless `INSERT … SELECT` on schema tables — no user input. Prisma is used everywhere else. |
| Migration safety | Enum-swap is not concurrent-safe. Documented as requiring a maintenance window. This is an operational risk, not a code vulnerability. |

No new security vulnerabilities introduced by this PR; two pre-existing concerns (tenant scope on `getAlertByCaseId` and the mismatched `caseId`/`case_id` alert update) have been fixed.

[↑ Back to top](#pr-review-cms-233--fix-replace-fraud_and_aml-container-case-with-investigationgroup-linkage)

---

## Test Coverage

- **Backend unit coverage is comprehensive for the refactor.** New/updated specs cover: `alert.service`, `alert.statistics.service`, `case-closure-approval.service` (including the "missing investigation task regression" describe block at `case-closure-approval.service.spec.ts:807`), `case-creation-approval.service`, `case-creation.service`, `case-query.service`, `case-reopening.service`, `case.repository`, `case.service`, `flowable.service`, new `investigation-group.service.spec.ts`, `rbac.service`, `report.service`, `task-lifecycle.service`, `task.service`, `triage.service` (with an explicit AML-failure rollback test).
- **Frontend tests updated for every touched component.** `AlertsDetailModal`, `CaseModalsManager`, `CloseCaseModal`, `TasksDetailsModal`, `ViewCaseModal`, `CaseActionsPanel`, `CaseDetailsTab`, `LinkedItemsTab`, `caseService`, and `triage.types` all have updated or new tests.
- **Gap: no test asserts that `TaskLogTab` compiles against `ViewCaseModal`.** The `onSwitchToCaseDetails` prop mismatch (Issue 1) is not caught by any spec — this is the kind of thing `tsc --noEmit` in CI (or a component-level render test) would catch.
- **PR checklist**: Locally ✓, Dev env ✓, Husky ✓, Unit tests + docs ✓. "Not needed, changes very basic" is (correctly) left unchecked. No coverage screenshot is attached in the PR body — worth confirming CI coverage output is green before merge.

[↑ Back to top](#pr-review-cms-233--fix-replace-fraud_and_aml-container-case-with-investigationgroup-linkage)

---

## CodeRabbit Activity

CodeRabbit reviewed four commits across the PR's iterations. Nearly every Major item has been resolved in-place.

### Pass 1 — initial commit `f2e484c`

**Findings:** 10 actionable + 4 outside-diff + 8 nits

| Finding | Severity | Status |
|---------|----------|--------|
| Stale `alert` after final `caseId:null` update in `triage.service` | Major | ✅ Resolved (captured as `detachedAlert`) |
| Stale `onSwitchToCaseDetails` prop | Major | ❌ Not resolved (still passed from `ViewCaseModal.tsx:189`) — see [Issue 1](#issue-1--tasklogtab-passes-onswitchtocasedetails-but-neither-declares-nor-forwards-it) |
| SAR/STR Filing Flowable status update gating | Major | ✅ Resolved (both `tx.case.update` and `handleCaseStatusChanged` gated) |
| `case_id` vs `caseId` in alert update on complete-case | Major | ✅ Resolved (single write using `caseId`) |
| Non-transactional writes in FRAUD_AND_AML split | Major | ✅ Resolved (writes moved into transaction) |
| Duplicate Flowable emission on case creation | Major | ✅ Resolved (single `createCase` side effect) |
| `undefined` vs `null` for alert unlink | Major | ✅ Resolved (`null` used) |
| Non-atomic FRAUD+AML triage | Major | ✅ Resolved (transaction wrap + rollback test) |
| Hardcoded `'investigator'` candidate group | Major | ✅ Resolved (`CANDIDATE_GROUPS.INVESTIGATIONS`) |
| Investigation task null guard | Major | ✅ Resolved (explicit `NotFoundException`) |
| `cases.group_id` index | Nit | ✅ Resolved (added in follow-on migration `20260702140000`) |
| Prisma `@relation`s for group/alert | Nit | ✅ Resolved (added in schema + follow-on migration) |
| Empty conditional in `getUserCasesByUserId` | Nit | ❌ Not resolved — see [Issue 2](#issue-2--getusercasesbyuserid-contains-an-empty-conditional-block) |
| Group badge hardcoded to `"FRAUD_AND_AML"` | Nit | ❌ Not resolved — see [Issue 3](#issue-3--casedetailstab-group-badge-is-a-literal-not-a-data-driven-value) |
| Duplicate handler logic in `AlertsDetailModal` | Nit | ❌ Not resolved — see [Issue 4](#issue-4--alertsdetailmodal-duplicates-closeinvalidate-logic-between-backdrop-and-button) |
| `related_case_type` as `string` | Nit | ❌ Not resolved — see [Issue 5](#issue-5--triagetypests-types-related_case_type-as-string) |
| `AlertRepository.getAlertIdsWithInvestigationGroup` materialises IDs | Nit | ✅ Resolved (method removed) |
| Missing investigation task regression test | Nit | ✅ Resolved (spec at `:807`) |

### Pass 2 — `009e638`

**Findings:** 1 actionable + 1 nit — both addressed the tenant scoping on `getAlertByCaseId` (now scoped by `tenant_id`) and mock-helper duplication in triage spec.

### Pass 3 — `024bf6a`

**Findings:** 1 actionable — investigation-task type annotation (`Task | null`). ✅ Resolved (`investigationTasks.at(0) ?? null`).

### Pass 4 — `0de9999`

Comment-only pass, no new findings.

[↑ Back to top](#pr-review-cms-233--fix-replace-fraud_and_aml-container-case-with-investigationgroup-linkage)

---

## Summary and Verdict

**Verdict: Approve with minor cleanup requested**

This is a genuinely large refactor and it has been executed cleanly. The synthetic container-case model was a real source of correctness problems (parent status propagation, permission bypass, triple-counted reports) and the `InvestigationGroup` replacement is a straightforward, well-scoped alternative. Two important pre-existing correctness bugs (tenant-scoping on `getAlertByCaseId`, mismatched `case_id`/`caseId` write on complete-case) were fixed as part of this work. All of CodeRabbit's Major findings — including the transactional atomicity of the two-step FRAUD+AML case creation and the SAR/STR Filing Flowable gate — were addressed in follow-up commits, and the risky rollback path has a dedicated test.

One real bug remains (`onSwitchToCaseDetails` passed to a component that does not declare it) plus a small handful of minor cleanups. The bug is a strict-TypeScript compile error at the `ViewCaseModal.tsx:189` call site — under most CI configurations this will fail the build, but even if it slips through, the callback is unreachable. Operationally, the enum-swap migration requiring a maintenance window and the mandatory deploy order (migrations → backfill → code) must be understood by whoever runs the release.

### Blocking

1. **`TaskLogTab` prop mismatch (Issue 1)** — `ViewCaseModal.tsx:189` still passes `onSwitchToCaseDetails` to `TaskLogTab` which does not declare it (nor does `TaskDetailsModal`). Drop the prop at the call site, or reinstate it in `TaskLogTabProps` and forward it.

### Non-blocking but recommended

2. **Empty conditional in `getUserCasesByUserId` (Issue 2)** — remove or fill in.
3. **Hardcoded `"FRAUD_AND_AML"` group badge (Issue 3)** — data-drive from case row.
4. **Duplicate close/invalidate handlers in `AlertsDetailModal` (Issue 4)** — extract shared `handleModalClose`.
5. **`related_case_type` type tightening (Issue 5)** — `string | null` → `CaseType | null`.
6. **Confirm supervisor closure UX (Issue 6)** — verify the report-generation-only supervisor path is intentional; if so, note it in the PR description.
7. **Restate maintenance-window requirement in the release runbook (Issue 7)** — enum-swap migration + strict deploy order.

[↑ Back to top](#pr-review-cms-233--fix-replace-fraud_and_aml-container-case-with-investigationgroup-linkage)

---

## GitHub Review Comment

````markdown
**Approve with minor cleanup requested**

Large, coherent refactor — the `InvestigationGroup` model cleanly replaces the FRAUD_AND_AML container case, tenant-scoping on `getAlertByCaseId` and the mismatched `case_id`/`caseId` alert-update are fixed as pre-existing bugs, and every Major CodeRabbit finding was resolved in follow-ups (including the two-step FRAUD+AML transaction and SAR/STR Filing Flowable gate). One real bug remains, plus five minor cleanups. Please also make sure the release runbook captures the maintenance-window requirement for the enum-swap migration and the strict `migrations → backfill → code` deploy order.

---

### Blocking

**1. `TaskLogTab` prop mismatch — `onSwitchToCaseDetails` is passed but never declared**

`frontend/src/features/cases/components/ViewCaseModal.tsx:189` still passes:

```tsx
<TaskLogTab
  ...
  onSwitchToCaseDetails={() => { setTab('details'); }}
/>
```

But `TaskLogTabProps` (`frontend/src/features/cases/components/view/TaskLogTab.tsx:39-51`) no longer declares that prop, and `TaskDetailsModalProps` (`frontend/src/features/cases/components/TasksDetailsModal.tsx:21-28`) doesn't accept it either. Under strict TypeScript this is a hard type error at the call site; even if the build tolerates it, the callback is unreachable.

Since the sub-case navigation flow was intentionally removed, the simplest fix is to drop the prop at the call site:

```diff
 <TaskLogTab
   ...
-  onSwitchToCaseDetails={() => { setTab('details'); }}
 />
```

If the "switch to details" affordance should remain for tasks, re-add the prop to `TaskLogTabProps` and forward it into `TaskDetailsModal` there.

---

### Non-blocking (please address in this PR if possible)

**2. Empty conditional in `getUserCasesByUserId`**

`backend/src/modules/case/case.controller.ts:512-518`:

```ts
const { userId: requestingUserId, tenantId } = extractUserData(req);
if (requestingUserId !== targetUserId) {
  /* empty */
}
return await this.caseService.getUserCases(targetUserId, query, tenantId);
```

`@RequireSupervisorRole()` already gates this — drop the block and the unused `requestingUserId`:

```ts
const { tenantId } = extractUserData(req);
return await this.caseService.getUserCases(targetUserId, query, tenantId);
```

**3. `CaseDetailsTab` group badge is a hardcoded literal**

`frontend/src/features/cases/components/view/CaseDetailsTab.tsx:467-493` always renders the string `"FRAUD_AND_AML"` when `row.groupId` is set. Fine while it's the only group type, but derive it from the case row (or a new `group_type` field) before more types are introduced.

**4. Duplicate close/invalidate logic in `AlertsDetailModal`**

`frontend/src/features/alerts/components/AlertsDetailModal.tsx` — the backdrop `onClick` and the close-button `onClick` both repeat identical `queryClient.invalidateQueries` + `onAlertUpdated?.()` + `onClose()` logic. Extract:

```tsx
const handleModalClose = () => {
  if (alert?.case_id) queryClient.invalidateQueries({ queryKey: ['case', alert.case_id] });
  if (alert?.related_case_id) queryClient.invalidateQueries({ queryKey: ['case', alert.related_case_id] });
  onAlertUpdated?.();
  onClose();
};
```

Then use `onClick={handleModalClose}` on both.

**5. Tighten `related_case_type` to `CaseType | null`**

`frontend/src/features/alerts/types/triage.types.ts:179-180`:

```diff
-  related_case_type?: string | null;
+  related_case_type?: CaseType | null;
```

`case_type: CaseType` is already used on the same interface — keeps types consistent and prevents drift from the backend enum.

**6. Confirm supervisor closure UX in `CloseCaseModal`**

`frontend/src/features/cases/components/CloseCaseModal.tsx:221-245` — supervisors get only "Generate Investigation Report" (gated on `!reportApproved`), no direct close/submit button. If that's intentional (i.e. the report-generation modal performs the close on approval), please note it in the PR description; if not, the supervisor close action is missing.
````

[↑ Back to top](#pr-review-cms-233--fix-replace-fraud_and_aml-container-case-with-investigationgroup-linkage)
