# PR Review: CMS #233 — Replace FRAUD_AND_AML container cases with InvestigationGroup linkage

**Repo:** tazama-lf/case-management-system
**Branch:** `paysys/fraud_and_aml` → `dev`
**Author:** sobia-rizwan1567 (Sobia Rizwan)
**Date Reviewed:** 2026-07-06
**Label:** bug
**Size:** +2306 / -2139 lines across 69 files
**Commits:** 77 (head `d5902b5f`; key CodeRabbit fix commits: `0d77fd5e`, `009e638e`, `024bf6a6`, `0de99991`, `37ae1b22`)
**State:** OPEN
**Existing approvals:** None (CodeRabbit only)

---

## Table of Contents

- [Overview](#overview)
- [What Changed (Detailed)](#what-changed-detailed)
  - [1. Investigation group schema, migration, and backfill](#1-investigation-group-schema-migration-and-backfill)
  - [2. `InvestigationGroupService` (new module)](#2-investigationgroupservice-new-module)
  - [3. `AlertRepository` — grouped resolution + tenant scoping](#3-alertrepository--grouped-resolution--tenant-scoping)
  - [4. `CaseCreationService` — remove duplicate Flowable emission and candidate group fix](#4-casecreationservice--remove-duplicate-flowable-emission-and-candidate-group-fix)
  - [5. `CaseCreationApprovalService` — transactional FRAUD/AML split](#5-casecreationapprovalservice--transactional-fraudaml-split)
  - [6. `CaseClosureApprovalService` — null-safe investigation task lookup](#6-caseclosureapprovalservice--null-safe-investigation-task-lookup)
  - [7. `CaseService.completeCaseCreation` — group creation + FRAUD/AML split](#7-caseservicecompletecasecreation--group-creation--fraudaml-split)
  - [8. `TriageService` — manual + AI triage FRAUD/AML pathways](#8-triageservice--manual--ai-triage-fraudaml-pathways)
  - [9. `TaskLifecycleService` — SAR/STR filing gating on unassign](#9-tasklifecycleservice--sarstr-filing-gating-on-unassign)
  - [10. Frontend — `parent_id` → `group_id` migration and modal flow](#10-frontend--parent_id--group_id-migration-and-modal-flow)
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

This PR (US-214) retires the synthetic `FRAUD_AND_AML` container case model and replaces it with a lightweight `InvestigationGroup` join entity that links sibling FRAUD and AML cases through a shared `group_id`. It also removes `STATUS_84_COMPLETED` from the status enum, permission matrix, reporting logic and every close/reopen/action pathway that touched the container-case model.

The core motivation is (a) removing duplicate/complex container logic, (b) fixing reporting inconsistencies (triple counting of container + child cases in ageing/workload reports), and (c) removing role bypass rules that the container case required. It is a **critical-effort** refactor by CodeRabbit's own assessment (~120 minutes of review time), touching schema/migrations, backend case + triage + task + reporting + RBAC layers, and the frontend cases/alerts UI.

The PR is the result of ~77 commits over 5 days, including 3 CodeRabbit review passes producing 12 actionable findings and 8 nitpicks. Most Major/Critical findings are resolved. A few important ones remain unresolved or only partially resolved (see [Issues and Observations](#issues-and-observations)).

| File / Area | Nature of Change |
|-------------|------------------|
| `backend/prisma/schema.prisma`, `backend/prisma/migrations/20260701000001_add_investigation_group`, `backend/prisma/migrations/20260702120000_remove_case_status_completed`, `backend/prisma/scripts/backfill-investigation-groups.sql` | Adds `investigation_groups` table with unique `alert_id`; adds `cases.group_id`; removes `STATUS_84_COMPLETED`; transactional backfill for existing FRAUD_AND_AML container cases. |
| `backend/src/modules/investigation-group/{investigation-group.module.ts,investigation-group.service.ts}` | New `InvestigationGroupService.createInvestigationGroup(alertId, tenantId, tx?)` — Prisma `upsert` keyed by `alert_id`, tx-aware. |
| `backend/src/modules/repository/alert.repository.ts` | `getAlertByCaseId` falls back through the `InvestigationGroup` when `alerts.case_id` is null (which it always is for FRAUD_AND_AML); adds tenant scoping. `getGroupedCasesForAlert` and `getAlertIdsWithInvestigationGroup` added. |
| `backend/src/modules/case/services/case-creation.service.ts` | Removes duplicate `executeFlowableCaseCreationEvent`; threads `isFraudNAML` through `createCase()`; uses `CANDIDATE_GROUPS.INVESTIGATIONS`. |
| `backend/src/modules/case/services/case-creation-approval.service.ts` | Wraps FRAUD/AML split in one `caseRepository.transaction`; alert unlinked with `caseId: null, tx`. |
| `backend/src/modules/case/services/case-closure-approval.service.ts` | Task lookup returns `Task \| null`; missing-task branch throws `NotFoundException`; redundant status re-check removed. |
| `backend/src/modules/case/case.service.ts` | Removes `getSubCasesDetails`; `completeCaseCreation` creates the investigation group before the tx and updates the case through the tx client; FRAUD/AML split kicks off the AML sibling after the tx commits. |
| `backend/src/modules/triage/triage.service.ts` | Manual triage: FRAUD/AML split creates group + AML case + comment inside the alertRepository tx; AI triage: FRAUD/AML sequence has a compensating rollback (`STATUS_99_ABANDONED`, `handleCaseAbandoned`, delete group). |
| `backend/src/modules/task/services/task-lifecycle.service.ts` | Retires `STATUS_84_COMPLETED` from status-change flows; SAR/STR filing gate added on `tx.case.update` — **but not on `flowableService.handleCaseStatusChanged`** (see [Issue 3](#issue-3--sarstr-filing-flowable-status-drift-not-fixed)). |
| `backend/src/modules/report/report.service.ts`, `backend/src/utils/rbac/permissionMatrix.json`, `backend/src/utils/enums/case-enum.ts` | Removes `STATUS_84_COMPLETED`; narrows ageing/workload calculations to non-container cases. |
| Frontend (`frontend/src/features/**`) | Replaces `parent_id`/`parentId` → `group_id`/`groupId`; removes sub-case props/state from `ViewCaseModal`, `CaseDetailsTab`, `CaseActionsPanel`, `TasksDetailsModal`, `LinkedItemsTab`; alert detail modal gets refresh + related-case navigation; `CloseCaseModal` supervisor path routes through `GenerateInvestigationReportModal`. |

[↑ Back to top](#pr-review-cms-233--replace-fraud_and_aml-container-cases-with-investigationgroup-linkage)

---

## What Changed (Detailed)

### 1. Investigation group schema, migration, and backfill

New table with a unique index on `alert_id` (one group per source alert, matching the FRAUD+AML alert semantics), and a nullable `group_id` FK on `cases`:

```prisma
model InvestigationGroup {
  id         Int    @id @default(autoincrement())
  alert_id   Int    @unique
  tenant_id  String
  created_at DateTime @default(now())
}

model Case {
  ...
  group_id   Int?      // → investigation_groups.id
}
```

The backfill script (`backfill-investigation-groups.sql`) is wrapped in a single Postgres transaction: for every legacy FRAUD_AND_AML container case it (a) creates an investigation group using the container's alert, (b) reparents both children to that group, and (c) marks the container itself for later removal. This is correct and idempotent given the unique constraint.

**Missing:** no index on `cases.group_id`. `getGroupedCasesForAlert` filters `cases` by `group_id`, so every grouped-case lookup does a sequential scan as the table grows. CodeRabbit flagged this as Trivial; I would ask for the index — the query is on the hot path (opened every time an alert detail modal or case detail modal is displayed).

---

### 2. `InvestigationGroupService` (new module)

```ts
async createInvestigationGroup(alertId: number, tenantId: string, tx?: Prisma.TransactionClient): Promise<{ id: number }> {
  const client = tx ?? this.prisma;
  return await client.investigationGroup.upsert({
    where: { alert_id: alertId },
    create: { alert_id: alertId, tenant_id: tenantId },
    update: {},
    select: { id: true },
  });
}
```

Idempotent, tenant-aware, tx-aware. Good primitive. Note the `update: {}` — no tenant validation on retry, so a mismatched `tenantId` on retry will silently return the original group's ID. In practice callers always pass their own tenant so this isn't exploitable, but it's a footgun if this service is reused elsewhere.

---

### 3. `AlertRepository` — grouped resolution + tenant scoping

`getAlertByCaseId` now (after fix `0de99991`):

```ts
async getAlertByCaseId(caseId: number, tenantId: string, tx?: Prisma.TransactionClient): Promise<number> {
  const client = tx ?? this.prisma;
  const alert = await client.alert.findFirst({
    where: { case_id: caseId, tenant_id: tenantId },
  });
  if (alert) return alert.alert_id;

  const caseRecord = await client.case.findFirst({
    where: { case_id: caseId, tenant_id: tenantId },
    select: { group_id: true },
  });
  if (caseRecord?.group_id) {
    const group = await client.investigationGroup.findFirst({
      where: { id: caseRecord.group_id, tenant_id: tenantId },
    });
    if (group) return group.alert_id;
  }
  throw new NotFoundException(`Alert with Case ID ${caseId} not found`);
}
```

The tenant-scoping fix is correct. Both the direct alert lookup and the case-to-group fallback are now tenant-bound. `findUnique` was replaced with `findFirst` because the unique key is now compound-effective — technically a compound `@@unique([case_id, tenant_id])` on `Alert` would restore `findUnique` and would be a stronger schema-level guarantee. This is acceptable as-is.

`getAlertIdsWithInvestigationGroup` still materializes every tenant's group `alert_id` into memory to be fed as `notIn` in the NALT alerts filter. CodeRabbit flagged as Trivial; realistically, the tenant's group count will grow linearly with FRAUD_AND_AML volume — this should eventually become a `NOT EXISTS` join. Non-blocking for this PR.

---

### 4. `CaseCreationService` — remove duplicate Flowable emission and candidate group fix

**Before:** `createCaseWithInvestigationTask()` created a case (which fired `handleCaseCreated`) *and then* fired `executeFlowableCaseCreationEvent(...true)` again → two BPMN process instances per case, with `isFraudNAML` conflicting.

**After:**

```ts
async createCase(dto, userId, tenantId, userRole, isFraudNAML = false): Promise<Case> {
  const createdCase = await this.caseRepository.createCase({ ... });
  await this.executeFlowableCaseCreationEvent(createdCase, dto.caseCreationType, isFraudNAML, userRole);
  ...
}

async createCaseWithInvestigationTask(...): Promise<...> {
  const newCase = await this.createCase(createCaseDTO, userId, tenantId, userRole, true);
  // no second executeFlowableCaseCreationEvent
  ...
}
```

The candidate group was also fixed from the literal `'investigator'` to `CANDIDATE_GROUPS.INVESTIGATIONS` (correct against the workflow value).

---

### 5. `CaseCreationApprovalService` — transactional FRAUD/AML split

The whole FRAUD/AML split is now inside a single `caseRepository.transaction(async (tx) => { ... })`:

```ts
const alertId = await this.alertRepository.getAlertByCaseId(caseId, tenantId, tx);
const investigationGroup = await this.investigationGroupService.createInvestigationGroup(alertId, tenantId, tx);
const finalCase = await this.caseRepository.updateCase(caseId, { case_type: CaseType.FRAUD, group_id: investigationGroup.id }, tx);
const amlCase = await this.caseRepository.createCase({ ..., groupId: investigationGroup.id }, tx);
// backfill Complete New Case + Approve Case Creation task history (both tx)
// create Investigate Case task on FRAUD + AML (tx)
await this.alertRepository.updateAlert(alertId, { caseId: null } as any, tx);
```

Flowable calls (`handleCaseStatusChanged`, `handleTaskCompleted`, `handleCaseCreated`) happen after `txResult` commits. This is the correct pattern — CodeRabbit's Major finding is fully resolved here.

The `as any` cast on `{ caseId: null }` is ugly (`UpdateAlertDTO` doesn't type `caseId` as nullable) but functionally correct.

---

### 6. `CaseClosureApprovalService` — null-safe investigation task lookup

```ts
const investigationTasks = caseData.tasks
  .filter((t) => t.name === TASK_NAMES.INVESTIGATE_CASE &&
    (t.status === TaskStatus.STATUS_20_IN_PROGRESS || t.status === TaskStatus.STATUS_30_COMPLETED))
  .sort((a, b) => new Date(b.created_at).getTime() - new Date(a.created_at).getTime());
const investigationTask: Task | null = investigationTasks.at(0) ?? null;

if (!investigationTask) {
  throw new NotFoundException({
    message: 'Investigation task not found or not in a closeable state',
    caseId,
    requiredStatuses: [TaskStatus.STATUS_20_IN_PROGRESS, TaskStatus.STATUS_30_COMPLETED],
  });
}
```

Critical null-deref → 500 issue fully resolved. Type is now explicitly `Task | null` so the guard is not dead code (`.at(0) ?? null` uses ES2024 available under the backend `tsconfig`). Redundant post-filter status check was removed.

---

### 7. `CaseService.completeCaseCreation` — group creation + FRAUD/AML split

The critical fix — `updateCase` now runs on the tx client:

```ts
const result = await this.prismaService.$transaction(async (prisma) => {
  let updatedCase = await prisma.case.update({ where: { case_id: caseId }, data: { ... } });
  if (investigationGroup !== undefined) {
    updatedCase = await prisma.case.update({ where: { case_id: caseId }, data: { group_id: investigationGroup.id, case_type: CaseType.FRAUD } });
  }
  ...
});
```

**But** three writes remain outside the transaction:
- `createInvestigationGroup(alertId, tenantId)` at L900 — *before* the tx begins, no `tx` passed.
- `alertRepository.updateAlert(getAlertIdByCaseId, alertUpdateData)` at L1032 — *after* the tx commits.
- The FRAUD_AND_AML branch: `createCaseWithInvestigationTask` at L1042 and a second `updateAlert(..., { caseId: null })` at L1051 — *after* the tx commits.

Because `createInvestigationGroup` is an idempotent `upsert` on `alert_id`, a retry after failure won't collide. But if the tx commits and `updateAlert` or `createCaseWithInvestigationTask` fails, the FRAUD case exists at `STATUS_02_READY_FOR_ASSIGNMENT` linked to a group but no AML sibling. This is a data-integrity risk (see [Issue 2](#issue-2--completecasecreation-fraudaml-split-not-fully-atomic)).

---

### 8. `TriageService` — manual + AI triage FRAUD/AML pathways

**Manual triage (`handleManualTriage`):** most writes are inside `alertRepository.transaction(async (tx) => ...)`, but `investigationGroupService.createInvestigationGroup(alert.alert_id, tenantId)` at L367 and `caseCreationService.updateCaseStatus(...)` at L368 do **not** pass `tx`. So the group + case status update escape the tx (see [Issue 4](#issue-4--triageservicehandlemanualtriage-fraudaml-writes-partially-outside-tx)).

Also, the final `alertService.updateAlert(alertId, userId, { caseId: null } as unknown as UpdateAlertDTO, tx)` at L399-406 discards its return value. The `alert` object returned from `handleManualTriage` is the earlier updated alert whose `case_id` still points at the FRAUD case, not `null` (see [Issue 5](#issue-5--stale-alert-payload-returned-from-manual-triage)).

**AI triage (`handleAITriage`):** takes a different, defensible approach — a `try/catch` compensating rollback:

```ts
try {
  await this.caseCreateService.createCaseWithInvestigationTask(CaseType.AML, ...);
} catch (amlError) {
  await Promise.all([
    this.caseRepository.updateCase(caseId, { status: CaseStatus.STATUS_99_ABANDONED, group_id: null }),
    this.caseRepository.updateCase(fraudCase.caseId, { status: CaseStatus.STATUS_99_ABANDONED, group_id: null }),
  ]);
  this.flowableService.handleCaseAbandoned({ caseId, reason: rollbackReason });
  this.flowableService.handleCaseAbandoned({ caseId: fraudCase.caseId, reason: rollbackReason });
  await this.prisma.investigationGroup.delete({ where: { id: investigationGroup.id } });
  // logged as AI_TRIAGE_FRAUD_AND_AML_ROLLED_BACK
  ...
  throw amlError;
}
```

This is defensible — as the comment notes, Postgres + Flowable can't do distributed rollback, and abandoning both sides + notifying Flowable + dropping the group is a coherent recovery. If the rollback itself fails, it's logged as `AI_TRIAGE_FRAUD_AND_AML_ROLLBACK_FAILED` — good operational signal. The `Promise.all` of two independent `updateCase` calls is fine but note that if the *primary* case is only referenced through Flowable (BPMN process still running), the `handleCaseAbandoned` calls are best-effort — no retry. Acceptable given the fallback log.

---

### 9. `TaskLifecycleService` — SAR/STR filing gating on unassign

```ts
if (updatedTask.name !== 'SAR/STR Filing') {
  await tx.case.update({
    where: { case_id: existingTask.case_id },
    data: { status: CaseStatus.STATUS_02_READY_FOR_ASSIGNMENT, case_owner_user_id: null, updated_at: new Date() },
  });
}

await this.flowableService.handleCaseStatusChanged({
  caseId: existingTask.case_id,
  newStatus: CaseStatus.STATUS_02_READY_FOR_ASSIGNMENT,
});
```

The DB gate exists but `handleCaseStatusChanged` still fires unconditionally. For SAR/STR Filing unassignments, Flowable will be told the case moved to `STATUS_02_READY_FOR_ASSIGNMENT` while the Postgres row keeps its previous status → BPMN state drift (see [Issue 3](#issue-3--sarstr-filing-flowable-status-drift-not-fixed)). CodeRabbit flagged this in the second pass; the fix was marked "Addressed in commits 0d77fd5 to 024bf6a" but the code shows the fix wasn't actually applied to the Flowable call.

---

### 10. Frontend — `parent_id` → `group_id` migration and modal flow

- Renames `parent_id` → `group_id` and `parentId` → `groupId` across shared types, `caseService.ts`, `useCase.ts`, `casesTable.utils.ts`, cases table renderer.
- `ViewCaseModal`, `CaseModalsManager`, `CaseDetailsTab`, `CaseActionsPanel`, `LinkedItemsTab` drop `parentCase`, `subCases`, `onSubCasesLoad`, etc. — replaced with `groupId` badge rendering where relevant.
- `AlertsDetailModal`: adds `queryClient.invalidateQueries(['case', case_id])` and `['case', related_case_id]` on both backdrop and close-button clicks; adds a related-case navigation button gated on `checkUserCaseAccess`.
- `CloseCaseModal`: removes the supervisor sub-case closure UI; supervisor now closes only through `GenerateInvestigationReportModal.onApproved` (Sobia confirmed this is intentional workflow — supervisors must generate a report before closing).
- `TasksDetailsModal` drops `onSwitchToCaseDetails` from its props — **but `TaskLogTab.tsx:636` still passes it** (see [Issue 6](#issue-6--stale-onswitchtocasedetails-prop-still-passed-to-taskdetailsmodal)).

[↑ Back to top](#pr-review-cms-233--replace-fraud_and_aml-container-cases-with-investigationgroup-linkage)

---

## Code Quality Analysis

### Strengths

- **Consistent transaction discipline where done right.** `CaseCreationApprovalService.approveCaseCreation` (the highest-risk path — mid-approval FRAUD/AML split with 5+ writes) is now fully transactional, with Flowable side-effects placed after commit. This is the correct pattern and it's applied thoroughly here.
- **Idempotent primitive design.** `InvestigationGroupService.createInvestigationGroup` uses `upsert` on `alert_id`, so retries don't collide. This lets non-transactional callers still be safe against duplicate-execution.
- **Explicit compensating rollback in AI triage.** The `handleAITriage` FRAUD_AND_AML branch acknowledges that Postgres + Flowable can't do distributed rollback and picks a sensible compensating flow (both cases → abandoned, group deleted, `AI_TRIAGE_FRAUD_AND_AML_ROLLED_BACK` audit log). This is a mature design choice.
- **Tenant scoping in `AlertRepository.getAlertByCaseId`** is now correct across all three lookup paths (direct alert, case→group, group→alert_id). Cross-tenant leak closed.
- **Backfill script is atomic.** `backfill-investigation-groups.sql` wraps the migration + reparenting in a single Postgres transaction.
- **`CaseClosureApprovalService`** null-guard + typed `Task | null` is a clean fix — the redundant status re-check was also removed as CodeRabbit suggested.
- **Sane RBAC/status matrix cleanup.** `STATUS_84_COMPLETED` is removed from `permissionMatrix.json`, `case-enum.ts`, and every closure/reopen/action path — no dangling references I could find.
- **BPMN candidate group** fix: `CANDIDATE_GROUPS.INVESTIGATIONS` replaces the hardcoded `'investigator'` mismatch.

### Issues and Observations

#### Issue 1 — `handleManualTriage` returns a stale `alert` payload

**Severity: Major (Data Integrity)**

`backend/src/modules/triage/triage.service.ts:399-418`. In the FRAUD_AND_AML manual triage branch, the final `alertService.updateAlert(alertId, userId, { caseId: null } as unknown as UpdateAlertDTO, tx)` discards its return value. The `alert` bundled into `{ alert, completeNewCaseTask, existingCase }` at L418 is the object returned from the *earlier* update at L323, whose `case_id` still points at the FRAUD case — not the persisted `null`. `handleManualTriage` returns this stale object to the caller.

Any client relying on the returned alert's `case_id` after triage will see wrong data (e.g., the alerts modal reading `alert.case_id` to route the user to the FRAUD case, even though the alert is now unlinked and grouped instead).

**Fix:**
```diff
-            await this.alertService.updateAlert(
+            const updatedAlert = await this.alertService.updateAlert(
               alertId,
               userId,
               { caseId: null } as unknown as UpdateAlertDTO,
               tx,
             );
-          }
+            return { alert: updatedAlert, completeNewCaseTask, existingCase };
+          }
```
(And a matching final `return` from the tx callback using `updatedAlert` where applicable.)

CodeRabbit raised this in the first pass and the "✅ Addressed" annotation is present, but the code was not updated. It is still stale.

---

#### Issue 2 — `completeCaseCreation` FRAUD/AML split not fully atomic

**Severity: Major (Data Integrity)**

`backend/src/modules/case/case.service.ts:897-1067`. The `$transaction` block only covers the case update, task completion, and comment insert. Three important writes remain outside:

1. `createInvestigationGroup(alertId, tenantId)` at L900 (before the tx begins). Because it's an idempotent `upsert`, this alone isn't catastrophic — but if the tx that follows fails, the group is committed on its own until the next retry re-uses it. Not ideal, but survivable.
2. `alertRepository.updateAlert(getAlertIdByCaseId, alertUpdateData)` at L1032 (after the tx commits).
3. FRAUD_AND_AML supervisor branch: `createCaseWithInvestigationTask(CaseType.AML, ...)` at L1042 + `updateAlert(getAlertIdByCaseId, { caseId: null })` at L1051 (both after the tx commits).

If step 3 fails, we're left with a FRAUD case at `STATUS_02_READY_FOR_ASSIGNMENT` linked to a group with no AML sibling and an alert still linked to the FRAUD case. There is no compensating cleanup here (unlike `handleAITriage`).

**Fix (recommended):** move `createInvestigationGroup` inside the tx (pass `prisma` client), and either (a) move the AML sibling creation + alert unlink inside the same tx (harder — `createCaseWithInvestigationTask` needs tx plumbing and fires Flowable), or (b) add compensating rollback (abandon FRAUD case + delete group + relink alert) modeled on `handleAITriage`.

---

#### Issue 3 — SAR/STR Filing Flowable status drift (not fixed)

**Severity: Major (Data Integrity / Integration)**

`backend/src/modules/task/services/task-lifecycle.service.ts:225-247`. The DB update is correctly gated:

```ts
if (updatedTask.name !== 'SAR/STR Filing') {
  await tx.case.update({ ... status: STATUS_02_READY_FOR_ASSIGNMENT ... });
}

await this.flowableService.handleCaseStatusChanged({          // <-- always runs
  caseId: existingTask.case_id,
  newStatus: CaseStatus.STATUS_02_READY_FOR_ASSIGNMENT,
});
```

For SAR/STR Filing unassignments, Flowable is told the case is `STATUS_02_READY_FOR_ASSIGNMENT` while the Postgres row keeps its previous status. The BPMN process instance moves to a state inconsistent with the persisted case → downstream Flowable-driven task creation will assume a status that isn't reality.

CodeRabbit raised this in the second pass; the "✅ Addressed in commits 0d77fd5 to 024bf6a" annotation is present but the Flowable call was not gated.

**Fix:**
```diff
-      await this.flowableService.handleCaseStatusChanged({
-        caseId: existingTask.case_id,
-        newStatus: CaseStatus.STATUS_02_READY_FOR_ASSIGNMENT,
-      });
+      if (updatedTask.name !== 'SAR/STR Filing') {
+        await this.flowableService.handleCaseStatusChanged({
+          caseId: existingTask.case_id,
+          newStatus: CaseStatus.STATUS_02_READY_FOR_ASSIGNMENT,
+        });
+      }
```

---

#### Issue 4 — `TriageService.handleManualTriage` FRAUD/AML writes partially outside tx

**Severity: Major (Data Integrity)**

`backend/src/modules/triage/triage.service.ts:366-386`. Inside the `alertRepository.transaction(async (tx) => ...)`:

```ts
const investigationGroup = await this.investigationGroupService.createInvestigationGroup(alert.alert_id, tenantId);
                                                                                                        // ^ no `tx`
await this.caseCreationService.updateCaseStatus(
  alert.case_id, ..., investigationGroup.id,
);                                                                                                       // no `tx`
const amlCase = await this.caseCreateService.createCaseWithInvestigationTask(
  CaseType.AML, ..., investigationGroup.id,
);                                                                                                       // no `tx`
```

`createInvestigationGroup`, `updateCaseStatus`, and `createCaseWithInvestigationTask` are all called on the root Prisma client, not through `tx`. This defeats the point of the surrounding transaction — a subsequent comment or alert write failure will not roll these three back.

`createInvestigationGroup` is idempotent (`upsert`), so retries are safe on that line. `updateCaseStatus` and `createCaseWithInvestigationTask` are not — a failure leaves the parent case at `STATUS_02_READY_FOR_ASSIGNMENT` and possibly an orphan AML case with no matching FRAUD-side state.

**Fix:** thread `tx` through `createInvestigationGroup` (already supported), and either (a) plumb tx into `updateCaseStatus` and `createCaseWithInvestigationTask`, or (b) apply the same compensating-rollback pattern as `handleAITriage`.

---

#### Issue 5 — Stale alert payload returned from manual triage

(This is the same code as Issue 1 — duplicated for verdict-table readability. Fold with Issue 1 during commit review.)

---

#### Issue 6 — Stale `onSwitchToCaseDetails` prop still passed to `TaskDetailsModal`

**Severity: Minor (Code Quality — type check failure)**

`frontend/src/features/cases/components/view/TaskLogTab.tsx:636` still passes `onSwitchToCaseDetails={onSwitchToCaseDetails}` to `TaskDetailsModal`, but `TaskDetailsModalProps` (`frontend/src/features/cases/components/TasksDetailsModal.tsx:21-28`) no longer declares that prop. Under strict TS this is a compile error; even if the build permits excess-property-check bypass, the prop is silently dropped.

**Fix:** remove the `onSwitchToCaseDetails` prop (and the surrounding `onSwitchToCaseDetails?: () => void` from `TaskLogTab`'s own props if it was only used to feed this).

---

#### Issue 7 — No index on `cases.group_id`

**Severity: Minor (Performance)**

`backend/prisma/migrations/20260701000001_add_investigation_group/migration.sql:17`. Every grouped-case lookup (`AlertRepository.getGroupedCasesForAlert`, feeding `resolveCaseIds`, driving the alert-detail modal and case-detail modal group section) filters `cases` by `group_id` with no index. Add:

```sql
CREATE INDEX "cases_group_id_idx" ON "cases"("group_id");
```

---

#### Issue 8 — `getAlertIdsWithInvestigationGroup` materializes all group alert IDs

**Severity: Minor (Performance)**

`backend/src/modules/repository/alert.repository.ts:148-155`. NALT alerts filtering loads every tenant's group `alert_id` into memory and feeds them as `notIn` to `getReportStatusFilter`. This grows linearly with tenant FRAUD_AND_AML volume. Non-blocking (CodeRabbit rated Trivial) — but plan to migrate to a `NOT EXISTS` subquery or add a Prisma relation.

---

#### Issue 9 — Prisma schema doesn't declare `@relation` for `group_id` and `alert_id`

**Severity: Informational (Maintainability)**

`backend/prisma/schema.prisma:113`. `Case.group_id` and `InvestigationGroup.alert_id` are plain scalar fields. Prisma won't enforce referential integrity at the client level, and consumers can't navigate the relation. Adding proper `@relation` fields is a low-risk correctness improvement, but this PR is already large — take it out of scope for this branch.

[↑ Back to top](#pr-review-cms-233--replace-fraud_and_aml-container-cases-with-investigationgroup-linkage)

---

## Security Assessment

| Concern | Assessment |
|---------|-----------|
| Cross-tenant alert leak in `getAlertByCaseId` | Pre-existing weak point; **fixed by this PR** (tenant scoping added to all three lookup paths). |
| Permission matrix drift after `STATUS_84_COMPLETED` removal | Reviewed — `permissionMatrix.json` no longer references the retired status; RBAC tier 2/3 checks continue to function. No orphan grants. |
| Container-case permission bypass rules | Removed as part of the `parent_id` / `sub_case` cleanup — supervisors no longer have implicit access via container-case chain, aligning with the actual data-access model. |
| Alert unlink using `undefined` (Prisma no-op) | Fixed — approval-side unlink now uses `{ caseId: null }`. |
| SQL injection risk in backfill script | None — parameterless static SQL with hard-coded column references. |
| Missing input validation | No new user-facing input surface; DTO shapes narrowed rather than widened. |

No new security vulnerabilities introduced by this PR. One pre-existing tenant-scoping gap in `getAlertByCaseId` is closed.

[↑ Back to top](#pr-review-cms-233--replace-fraud_and_aml-container-cases-with-investigationgroup-linkage)

---

## Test Coverage

- **New test file:** `backend/test/investigation-group.service.spec.ts` — covers `createInvestigationGroup` upsert, tx-vs-root client selection, and tenant handling. Adequate.
- **Updated tests:** `alert.service.spec.ts`, `alert.statistics.service.spec.ts`, `case.service.spec.ts`, `case-creation.service.spec.ts`, `case-creation-approval.service.spec.ts`, `case-closure-approval.service.spec.ts`, `triage.service.spec.ts`, `case-query.service.spec.ts`, `report.service.spec.ts`, `case.repository.spec.ts` — all reflect the new `group_id` / grouped-case-lookup contract.
- **Missing:**
  - No regression test in `case-closure-approval.service.spec.ts` for the "no active investigation task" branch → the newly-added `NotFoundException` guard is untested.
  - No regression test for `handleManualTriage`'s FRAUD_AND_AML branch returning a fresh `alert` payload (Issue 1) — the stale-alert bug can happen silently.
  - No integration test simulating a mid-tx failure in `completeCaseCreation` or `handleAITriage` rollback path (hard to write, but the rollback contract is nontrivial).
- **PR checklist:** all four boxes checked (Locally, Development Environment, Husky, Unit tests + docs). No CI coverage screenshot attached.

Given the size and criticality of the refactor, the test additions are proportionate for happy paths but weak on the newly-introduced failure/error branches that CodeRabbit specifically flagged.

[↑ Back to top](#pr-review-cms-233--replace-fraud_and_aml-container-cases-with-investigationgroup-linkage)

---

## CodeRabbit Activity

### Pass 1 — initial review on commit `f2e484ca`

**Findings:** 10 actionable + 4 outside-diff comments + 8 nitpicks

| Finding | Severity | Status |
|---------|----------|--------|
| Keep case update + investigation-group write on tx client (`case.service.ts:897-919`) | Major | ⚠️ Partial — group created before tx (idempotent), case update inside tx, but AML sibling + alert updates still outside tx (Issue 2) |
| Missing null-check on `investigationTask` (`case-closure-approval.service.ts:140`) | Critical | ✅ Resolved |
| FRAUD_AND_AML split needs transaction (`case-creation-approval.service.ts:198-309`) | Major | ✅ Resolved |
| Use `null` not `undefined` to clear case link (`case-creation-approval.service.ts:309`) | Major | ✅ Resolved |
| Avoid starting `caseManagementProcess` twice (`case-creation.service.ts:89-116`) | Major | ✅ Resolved |
| Use `CANDIDATE_GROUPS.INVESTIGATIONS` (`case-creation.service.ts:106-113`) | Minor | ✅ Resolved |
| Remove duplicate Flowable case-creation calls (`case-creation.service.ts:174-194`) | Critical | ✅ Resolved |
| Thread FRAUD_AND_AML branch through transaction (`triage.service.ts:366-406`) | Major | ❌ Not resolved — group + status + AML case still outside tx (Issue 4) |
| Add rollback/compensation for FRAUD+AML triage (`triage.service.ts:629-656`) | Major | ✅ Resolved (compensating rollback added in `handleAITriage`) |
| Supervisor close path via report only (`CloseCaseModal.tsx:246`) | Major | ✅ Resolved — intentional workflow change confirmed by author |
| Stale `alert` returned from manual triage (`triage.service.ts:399-418`, outside diff) | Major | ❌ Not resolved (Issue 1/5) |
| Stale `onSwitchToCaseDetails` prop (`TasksDetailsModal.tsx:21-28`, outside diff) | Major | ❌ Not resolved (Issue 6) |
| Gate Flowable status on SAR/STR unassign (`task-lifecycle.service.ts:225-241`, outside diff) | Major | ❌ Not resolved (Issue 3) |
| Wrong field name on alert update in complete-case flow (`case.service.ts:1011-1042`, outside diff) | Major | ✅ Resolved — now uses `caseId`; two writes remain (see Issue 2 for atomicity) |

### Pass 2 — after fixes on commit `009e638e`

**Findings:** 1 actionable (`triage.service.spec.ts:578-593` — nitpick) — resolved.

### Pass 3 — after fixes on commit `024bf6a6`

**Findings:** 1 actionable

| Finding | Severity | Status |
|---------|----------|--------|
| Make lookup return `Task \| null` — use `.at(0) ?? null` (`case-closure-approval.service.ts:125`) | Major | ✅ Resolved |

### Pass 4 — after fixes on commit `0de99991`

**Findings:** tenant scoping on `alert.repository.ts:116` — ✅ Resolved.

**Note:** CodeRabbit's auto-annotations ("✅ Addressed in commits X to Y") are unreliable in this PR — several were incorrect. Issues 1, 3, and 6 were marked resolved but the corresponding code is unchanged. Every "addressed" claim was verified against the working tree at head `d5902b5f`.

[↑ Back to top](#pr-review-cms-233--replace-fraud_and_aml-container-cases-with-investigationgroup-linkage)

---

## Summary and Verdict

**Verdict: Changes Requested**

This is a well-motivated, large-scale refactor that correctly retires the FRAUD_AND_AML container model and replaces it with a cleaner grouped-case model. The critical CodeRabbit findings around approval-flow transactionality, closure null-safety, duplicate Flowable emission, candidate group misconfiguration, and tenant scoping in `AlertRepository` are all resolved. `CaseCreationApprovalService` and `CaseClosureApprovalService` are in good shape, and the compensating rollback in `handleAITriage` is a mature answer to the Postgres+Flowable atomicity problem.

However, three Major findings CodeRabbit flagged and auto-marked as "Addressed" are, on inspection of the code at head `d5902b5f`, **not fixed**: the SAR/STR Filing Flowable gate (Issue 3), the stale alert payload returned from manual triage (Issue 1), and the stale `onSwitchToCaseDetails` prop passed to `TaskDetailsModal` (Issue 6). Additionally, `CaseService.completeCaseCreation` (Issue 2) and `TriageService.handleManualTriage` (Issue 4) have data-integrity gaps where key writes escape the surrounding transaction with no compensating rollback — inconsistent with the disciplined pattern followed elsewhere in the PR.

### Blocking

1. **Issue 1/5 — `handleManualTriage` returns stale `alert`.** The FRAUD_AND_AML branch discards the result of the final `caseId: null` update; callers see an alert payload with `case_id` still pointing at the (now-detached) FRAUD case.
2. **Issue 2 — `completeCaseCreation` FRAUD/AML split not atomic.** AML sibling creation, group linking, and alert unlink happen after the transaction commits with no compensating rollback. A mid-branch failure produces a half-split case group.
3. **Issue 3 — SAR/STR Filing Flowable status drift.** `handleCaseStatusChanged` fires unconditionally on task unassign, even when the case row is intentionally left unchanged for SAR/STR filing. BPMN state diverges from persisted state.
4. **Issue 4 — `handleManualTriage` FRAUD/AML writes outside the transaction.** `createInvestigationGroup`, `updateCaseStatus`, and `createCaseWithInvestigationTask` run on the root Prisma client despite being inside a surrounding `alertRepository.transaction`.
5. **Issue 6 — `TaskLogTab` still passes removed prop.** Type check will fail on `onSwitchToCaseDetails`.

### Non-blocking but recommended

6. **Issue 7 — Add `CREATE INDEX cases_group_id_idx ON cases(group_id)`.** Grouped-case lookups are on the alert-modal / case-modal hot path.
7. **Issue 8 — `getAlertIdsWithInvestigationGroup` in-memory `notIn` list.** Migrate to `NOT EXISTS` join.
8. **Issue 9 — Add Prisma `@relation` for `group_id` / `alert_id`** (out of scope for this PR; capture as tech debt).
9. Add regression tests for the new failure branches (`NotFoundException` in `CaseClosureApprovalService`, AML rollback in `handleAITriage`, stale-alert regression in `handleManualTriage`).

[↑ Back to top](#pr-review-cms-233--replace-fraud_and_aml-container-cases-with-investigationgroup-linkage)

---

## GitHub Review Comment

`````markdown
**Changes Requested**

Great refactor overall — the approval-side transactional split (`CaseCreationApprovalService`), the null-safe closure lookup, the duplicate-Flowable-emission removal, the candidate-group correction, and the tenant scoping on `AlertRepository.getAlertByCaseId` are all cleanly resolved. However, three of CodeRabbit's Major findings were auto-marked as "Addressed" but on inspection of head `d5902b5f` are not actually fixed, and two additional atomicity gaps remain where key writes escape a surrounding transaction with no compensating rollback. Details, code, and complete fixes below.

⚠️ **Note on CodeRabbit annotations:** the "✅ Addressed in commits X to Y" auto-annotations were incorrect for Blocking #1, #3, and #5 below — the code at head `d5902b5f` is unchanged from the originally-flagged state. Please verify against the working tree, not the CodeRabbit summary, before merging.

---

### Blocking

---

**1. `handleManualTriage` returns a stale `alert` payload** — [`backend/src/modules/triage/triage.service.ts:290-436`](backend/src/modules/triage/triage.service.ts#L290-L436)

**Problem.** In the FRAUD_AND_AML manual-triage branch, the final `alertService.updateAlert(...{ caseId: null }...)` at L399-406 discards its return value. The `alert` object bundled into `{ alert, completeNewCaseTask, existingCase }` at L418 is the object returned by the *earlier* update at L323, whose `case_id` still points at the FRAUD case — not the persisted `null`. `handleManualTriage` returns this stale object to the caller.

Any client depending on the returned alert's `case_id` after triage will observe a link that doesn't exist in the DB (e.g., the alert-detail modal reading `alert.case_id` to route to a case that was actually detached and replaced with a group).

**Fix.** Capture the final update's result and thread it through the transaction return value:

```diff
         if (alert.alert_type === CaseType.FRAUD_AND_AML) {
-          const investigationGroup = await this.investigationGroupService.createInvestigationGroup(alert.alert_id, tenantId);
+          const investigationGroup = await this.investigationGroupService.createInvestigationGroup(alert.alert_id, tenantId, tx);
           await this.caseCreationService.updateCaseStatus(
             alert.case_id,
             CaseStatus.STATUS_02_READY_FOR_ASSIGNMENT,
             userId,
             tenantId,
             priority,
             CaseType.FRAUD,
             investigationGroup.id,
+            tx,
           );

           const amlCase = await this.caseCreateService.createCaseWithInvestigationTask(
             CaseType.AML,
             userId,
             tenantId,
             priority,
             CaseCreationType.AUTOMATIC_SYSTEM,
             'SUPERVISOR',
             investigationGroup.id,
+            tx,
           );

           await this.commentRepository.createComment(
             userId,
             { caseId: amlCase.caseId, taskId: amlCase.taskId, tenantId, note: updateAlertDto.note },
             tx,
           );

-          await this.alertService.updateAlert(
+          const detachedAlert = await this.alertService.updateAlert(
             alertId,
             userId,
             { caseId: null } as unknown as UpdateAlertDTO,
             tx,
           );
+          return { alert: detachedAlert, completeNewCaseTask, existingCase };
         } else {
           ...
         }
       }
-      return { alert, completeNewCaseTask, existingCase };
+      return { alert, completeNewCaseTask, existingCase };
     });
```

Note this fix depends on the tx-plumbing changes in Blocking #4 (see below). Together they close both the stale-payload and the atomicity gap.

**Regression test to add** (`backend/test/triage.service.spec.ts`):

```ts
it('handleManualTriage FRAUD_AND_AML: returns alert with case_id = null after AML sibling created', async () => {
  // arrange: alert with case_id = <fraud case>, alert_type = FRAUD_AND_AML
  // stub alertService.updateAlert to record calls; make the final call resolve with case_id: null
  const result = await triageService.handleManualTriage(alertId, dto, userId, tenantId);
  expect(result.case_id).toBeNull();
  expect(alertService.updateAlert).toHaveBeenLastCalledWith(
    alertId, userId, expect.objectContaining({ caseId: null }), expect.anything(), undefined,
  );
});
```

---

**2. `completeCaseCreation` FRAUD/AML split is not atomic and has no compensating rollback** — [`backend/src/modules/case/case.service.ts:896-1096`](backend/src/modules/case/case.service.ts#L896-L1096)

**Problem.** The `$transaction` at L907-984 correctly covers the case update + task completion + comment insert. But four writes remain outside the tx with **no compensating rollback**, unlike `handleAITriage`:

| Line | Write | Consequence if it fails |
|------|-------|-------------------------|
| L900 | `createInvestigationGroup(alertId, tenantId)` — before tx begins | Group left committed on its own; safe on retry via `upsert` |
| L1032 | `alertRepository.updateAlert(alertId, { …, caseId })` — after tx commits | Case exists, task completed, but alert isn't linked back |
| L1042 | `createCaseWithInvestigationTask(CaseType.AML, ...)` — after tx commits | FRAUD case at `STATUS_02_READY_FOR_ASSIGNMENT` linked to group, no AML sibling |
| L1051 | `alertRepository.updateAlert(alertId, { caseId: null })` — after tx commits | Alert still linked to FRAUD case even though the group model says it shouldn't be |

If step 3 fails (AML case creation), the group is half-populated and the alert is misrouted. There's no recovery — user retries would fail at the group-`upsert` returning the same ID but the FRAUD case is already past the draft state. Contrast this with `handleAITriage`, which explicitly abandons both cases + deletes the group + logs a rollback audit event when the AML step fails.

**Fix.** Pick one of two approaches. **Preferred approach — full tx** requires plumbing `tx` through `caseCreationService.createCaseWithInvestigationTask`; see Blocking #4 for the signature change. Then:

```diff
   try {
-    let investigationGroup: { id: number } | undefined;
-    if (isSupervisor) {
-      investigationGroup = isFraudNAML
-        ? await this.investigationGroupService.createInvestigationGroup(
-            await this.alertRepository.getAlertByCaseId(caseId, tenantId),
-            tenantId,
-          )
-        : undefined;
-    }
-
-    const result = await this.prismaService.$transaction(async (prisma) => {
+    const result = await this.prismaService.$transaction(async (prisma) => {
+      let investigationGroup: { id: number } | undefined;
+      if (isSupervisor && isFraudNAML) {
+        const alertId = await this.alertRepository.getAlertByCaseId(caseId, tenantId, prisma);
+        investigationGroup = await this.investigationGroupService.createInvestigationGroup(alertId, tenantId, prisma);
+      }
+
       const caseUpdateData = {
         ...updateData,
         caseType: isFraudNAML && isSupervisor ? CaseType.FRAUD : updateData.caseType,
         status: targetStatus,
       };
       let updatedCase = await prisma.case.update({ where: { case_id: caseId }, data: { ... } });
       if (investigationGroup !== undefined) {
         updatedCase = await prisma.case.update({
           where: { case_id: caseId },
           data: { group_id: investigationGroup.id, case_type: CaseType.FRAUD },
         });
       }

       // ... completeNewCaseTask update, comment, existing writes ...

+      // Link/relink the alert inside the tx
+      const alertId = await this.alertRepository.getAlertByCaseId(caseId, tenantId, prisma);
+      if (alertId) {
+        await this.alertRepository.updateAlert(
+          alertId,
+          {
+            priority_score: updateData.priorityScore,
+            priority: updateData.priority,
+            alertType: updateData.caseType,
+            predictionOutcome: updateData.predictionOutcome,
+            confidencePer: updateData.confidence,
+            caseId: isFraudNAML && isSupervisor ? null : caseId,   // one authoritative write
+          } as unknown as UpdateAlertDTO,
+          prisma,
+        );
+      }
+
+      let amlCase: { caseId: number; message: string; taskId?: number } | null = null;
+      if (isFraudNAML && isSupervisor) {
+        amlCase = await this.caseCreationService.createCaseWithInvestigationTask(
+          CaseType.AML, userId, existingCase.tenant_id, result.case.priority,
+          CaseCreationType.AUTOMATIC_SYSTEM, role, investigationGroup!.id, prisma,
+        );
+      }
+
-      return { case: updatedCase, completedTask };
+      return { case: updatedCase, completedTask, amlCase, investigationGroup };
     });

     // move Flowable calls (handleCaseStatusChanged, handleTaskCompleted) and
     // logging AFTER the tx commits — they should not participate in rollback

-    const getAlertIdByCaseId = await this.alertRepository.getAlertByCaseId(caseId, tenantId);
-    if (getAlertIdByCaseId) {
-      await this.alertRepository.updateAlert(getAlertIdByCaseId, {...caseId});
-    }
-    ...
-    if (isFraudNAML && isSupervisor) {
-      amlCase = await this.caseCreationService.createCaseWithInvestigationTask(...);
-      await this.alertRepository.updateAlert(getAlertIdByCaseId, { caseId: null });
-    }
```

**Alternative approach — compensating rollback** (if plumbing tx through `createCaseWithInvestigationTask` is out-of-scope for this branch), mirror `handleAITriage`'s pattern:

```ts
// After the $transaction commits:
let amlCase: { caseId: number; ... } | null = null;
if (isFraudNAML && isSupervisor) {
  try {
    amlCase = await this.caseCreationService.createCaseWithInvestigationTask(
      CaseType.AML, userId, existingCase.tenant_id, result.case.priority,
      CaseCreationType.AUTOMATIC_SYSTEM, role, investigationGroup!.id,
    );
    await this.alertRepository.updateAlert(getAlertIdByCaseId, { caseId: null } as unknown as UpdateAlertDTO);
  } catch (amlError) {
    const reason = `completeCaseCreation FRAUD_AND_AML rollback: AML creation failed for case ${caseId}: ${
      amlError instanceof Error ? amlError.message : String(amlError)
    }`;
    this.logger.error(reason, amlError instanceof Error ? amlError.stack : undefined, CaseService.name);
    try {
      await this.caseRepository.updateCase(caseId, {
        status: CaseStatus.STATUS_99_ABANDONED,
        group_id: null,
        case_type: existingCase.case_type,   // restore original FRAUD_AND_AML type
      });
      this.flowableService.handleCaseAbandoned({ caseId, reason });
      await this.prismaService.investigationGroup.delete({ where: { id: investigationGroup!.id } });
      // Alert wasn't nulled yet — leave it linked to the (now-abandoned) case
      await this.loggingOrchestrationService.logActions({
        userId, operation: 'COMPLETE_CASE_CREATION_FRAUD_AND_AML_ROLLED_BACK',
        entityName: 'Case',
        actionPerformed: `Rolled back FRAUD_AND_AML draft completion for case ${caseId}: abandoned case, deleted group ${investigationGroup!.id}`,
        outcome: Outcome.FAILURE, tenantId: existingCase.tenant_id,
      });
    } catch (rollbackErr) {
      this.logger.error(
        `completeCaseCreation FRAUD_AND_AML rollback failed for case ${caseId} (group ${investigationGroup!.id}) — needs manual reconciliation`,
        rollbackErr instanceof Error ? rollbackErr.stack : undefined, CaseService.name,
      );
      await this.loggingOrchestrationService.logActions({
        userId, operation: 'COMPLETE_CASE_CREATION_FRAUD_AND_AML_ROLLBACK_FAILED',
        entityName: 'Case',
        actionPerformed: `Rollback failed for case ${caseId} (group ${investigationGroup!.id}) — needs manual reconciliation`,
        outcome: Outcome.FAILURE, tenantId: existingCase.tenant_id,
      });
    }
    throw amlError;
  }
}
```

**Regression tests to add** (`backend/test/case.service.spec.ts`):
- `completeCaseCreation FRAUD_AND_AML: on AML failure, primary case is abandoned, group deleted, and error rethrown`
- `completeCaseCreation FRAUD_AND_AML: on rollback failure, logs COMPLETE_CASE_CREATION_FRAUD_AND_AML_ROLLBACK_FAILED and rethrows original error`

---

**3. SAR/STR Filing Flowable status drift on unassign** — [`backend/src/modules/task/services/task-lifecycle.service.ts:225-247`](backend/src/modules/task/services/task-lifecycle.service.ts#L225-L247)

**Problem.** The Postgres update is gated (`if (updatedTask.name !== 'SAR/STR Filing')`) — the case row correctly keeps its previous status for SAR/STR unassigns. But `flowableService.handleCaseStatusChanged` fires unconditionally right after. BPMN is told `STATUS_02_READY_FOR_ASSIGNMENT` while Postgres still shows the previous status, so the process instance moves to a state that no longer matches reality. Downstream Flowable-driven task creation will assume `STATUS_02_READY_FOR_ASSIGNMENT` and can create wrong tasks or make invalid transitions.

**Fix.** Gate `handleCaseStatusChanged` behind the same predicate. `handleTaskUnassigned` should still fire — the task itself really was unassigned:

```diff
     if (updatedTask.name !== 'SAR/STR Filing') {
       await tx.case.update({
         where: { case_id: existingTask.case_id },
         data: {
           status: CaseStatus.STATUS_02_READY_FOR_ASSIGNMENT,
           case_owner_user_id: null,
           updated_at: new Date(),
         },
       });
+      await this.flowableService.handleCaseStatusChanged({
+        caseId: existingTask.case_id,
+        newStatus: CaseStatus.STATUS_02_READY_FOR_ASSIGNMENT,
+      });
     }

-    await this.flowableService.handleCaseStatusChanged({
-      caseId: existingTask.case_id,
-      newStatus: CaseStatus.STATUS_02_READY_FOR_ASSIGNMENT,
-    });
     await this.flowableService.handleTaskUnassigned({
       taskId,
       caseId: existingTask.case_id,
       taskName: existingTask.name!,
       assignedUser: null,
     });
```

**Regression test to add** (`backend/test/task-lifecycle.service.spec.ts`):

```ts
it('unassignTask SAR/STR Filing: does NOT change case status in Postgres OR Flowable', async () => {
  // arrange: task with name = 'SAR/STR Filing', assigned_user_id set
  await taskLifecycleService.unassignTask(taskId, actorUserId, tenantId, 'switching filer', user, endpointKey);

  expect(prisma.case.update).not.toHaveBeenCalled();
  expect(flowableService.handleCaseStatusChanged).not.toHaveBeenCalled();
  expect(flowableService.handleTaskUnassigned).toHaveBeenCalledWith(expect.objectContaining({ taskId }));
});
```

---

**4. `handleManualTriage` FRAUD/AML writes escape the transaction** — [`backend/src/modules/triage/triage.service.ts:366-386`](backend/src/modules/triage/triage.service.ts#L366-L386)

**Problem.** Inside `alertRepository.transaction(async (tx) => ...)`, three writes bypass `tx`:

```ts
const investigationGroup = await this.investigationGroupService.createInvestigationGroup(alert.alert_id, tenantId);
//                                                                                                        ^ no tx
await this.caseCreationService.updateCaseStatus(alert.case_id, ..., investigationGroup.id);              // no tx
const amlCase = await this.caseCreateService.createCaseWithInvestigationTask(CaseType.AML, ..., investigationGroup.id); // no tx
```

`createInvestigationGroup` **already** accepts an optional `tx` parameter — passing it is a one-line change. `updateCaseStatus` and `createCaseWithInvestigationTask` need to be extended to accept and thread `tx`.

**Fix (part A) — plumb `tx` into `createInvestigationGroup` call sites:**

```diff
-  const investigationGroup = await this.investigationGroupService.createInvestigationGroup(alert.alert_id, tenantId);
+  const investigationGroup = await this.investigationGroupService.createInvestigationGroup(alert.alert_id, tenantId, tx);
```

Same one-liner also needed at [`case.service.ts:900`](backend/src/modules/case/case.service.ts#L900) (see Blocking #2) and [`triage.service.ts:629`](backend/src/modules/triage/triage.service.ts#L629) (`handleAITriage` should do this too even though it has a compensating rollback — cheaper than the compensating path).

**Fix (part B) — extend `createCaseWithInvestigationTask` to accept `tx`:**

```diff
// backend/src/modules/case/services/case-creation.service.ts
   async createCaseWithInvestigationTask(
     alertType: CaseType,
     userId: string,
     tenantId: string,
     priority: Priority,
     caseCreationType: CaseCreationType = CaseCreationType.AUTOMATIC_SYSTEM,
     userRole: string,
     groupId?: number,
+    tx?: Prisma.TransactionClient,
   ): Promise<{ caseId: number; message: string; taskId?: number }> {
     try {
       const createCaseDTO: CreateCaseDto = { ... };
-      const newCase = await this.createCase(createCaseDTO, userId, tenantId, userRole, true);
+      const newCase = await this.createCase(createCaseDTO, userId, tenantId, userRole, true, tx);
       const completeTask = await this.taskService.createTask({...}, userId, tenantId, tx);
       await this.taskService.createTask({...}, userId, tenantId, tx);
       ...
     }
   }

   async createCase(
     createCaseDTO: CreateCaseDto,
     userId: string,
     tenantId: string,
     userRole: string,
     isFraudNAML = false,
+    tx?: Prisma.TransactionClient,
   ): Promise<Case> {
-    const createdCase = await this.caseRepository.createCase({ ... });
+    const createdCase = await this.caseRepository.createCase({ ... }, tx);
     // NOTE: leave Flowable emission OUT of the tx — Flowable can't roll back.
     // If the caller passes a tx, defer the Flowable call to after the tx by
     // returning a callback or having the caller invoke handleCaseCreated. See
     // CaseCreationApprovalService for the pattern already established here.
-    await this.executeFlowableCaseCreationEvent(createdCase, ...);
+    if (!tx) {
+      await this.executeFlowableCaseCreationEvent(createdCase, createCaseDTO.caseCreationType, isFraudNAML, userRole);
+    }
     ...
     return createdCase;
   }
```

Callers that pass `tx` must then invoke `executeFlowableCaseCreationEvent` **after** the tx commits — this is exactly the pattern already used in `CaseCreationApprovalService.approveCaseCreation` (see [`case-creation-approval.service.ts:285-317`](backend/src/modules/case/services/case-creation-approval.service.ts#L285-L317)).

**Fix (part C) — extend `updateCaseStatus` to accept `tx`:**

```diff
// backend/src/modules/case/services/case-creation-approval.service.ts:584
   async updateCaseStatus(
     caseId: number,
     status: CaseStatus,
     userId: string,
     tenantId: string,
     priority?: Priority,
     caseType?: CaseType,
     groupId?: number,
+    tx?: Prisma.TransactionClient,
   ): Promise<Case> {
     ...
     const updatedCase = await this.caseRepository.updateCase(caseId, updateData, tx);
     // Flowable call stays outside — but caller inside a tx should call handleCaseStatusChanged after commit.
     if (!tx) {
       this.flowableService.handleCaseStatusChanged({ caseId, newStatus: status });
     }
     ...
   }
```

Then in `handleManualTriage`:

```diff
   if (alert.alert_type === CaseType.FRAUD_AND_AML) {
-    const investigationGroup = await this.investigationGroupService.createInvestigationGroup(alert.alert_id, tenantId);
+    const investigationGroup = await this.investigationGroupService.createInvestigationGroup(alert.alert_id, tenantId, tx);
     await this.caseCreationService.updateCaseStatus(
-      alert.case_id, CaseStatus.STATUS_02_READY_FOR_ASSIGNMENT, userId, tenantId, priority, CaseType.FRAUD, investigationGroup.id,
+      alert.case_id, CaseStatus.STATUS_02_READY_FOR_ASSIGNMENT, userId, tenantId, priority, CaseType.FRAUD, investigationGroup.id, tx,
     );
     const amlCase = await this.caseCreateService.createCaseWithInvestigationTask(
-      CaseType.AML, userId, tenantId, priority, CaseCreationType.AUTOMATIC_SYSTEM, 'SUPERVISOR', investigationGroup.id,
+      CaseType.AML, userId, tenantId, priority, CaseCreationType.AUTOMATIC_SYSTEM, 'SUPERVISOR', investigationGroup.id, tx,
     );
     ...
   }
```

And move the deferred Flowable calls into `executeFlowableOperations` (which already runs after `transactionResult`):

```ts
private async executeFlowableOperations(
  updateAlertDto: ManualAlertUpdateDTO, completeNewCaseTask: Task, existingCase: Case, alert: Alert,
  fraudCaseId?: number, amlCaseId?: number,   // <— new
): Promise<void> {
  ...
  if (fraudCaseId && amlCaseId) {
    await this.flowableService.handleCaseStatusChanged({ caseId: fraudCaseId, newStatus: CaseStatus.STATUS_02_READY_FOR_ASSIGNMENT });
    await this.flowableService.handleCaseCreated({ caseId: amlCaseId, ..., isFraudNAML: true });
    await this.flowableService.handleCaseStatusChanged({ caseId: amlCaseId, newStatus: CaseStatus.STATUS_02_READY_FOR_ASSIGNMENT });
  }
}
```

**Alternative approach — compensating rollback** (if the tx-plumbing above is more than you want to take on for this PR), mirror the pattern already in `handleAITriage` (L648-707): wrap the FRAUD/AML sequence in `try/catch`, and on failure abandon both cases (`STATUS_99_ABANDONED`, `group_id: null`), fire `handleCaseAbandoned` for each, delete the `investigationGroup`, and log `MANUAL_TRIAGE_FRAUD_AND_AML_ROLLED_BACK`. This is inconsistent with the surrounding transactional style but at least matches the AI-triage precedent.

**Regression test to add** (`backend/test/triage.service.spec.ts`): mid-branch failure test — force `createCaseWithInvestigationTask(AML)` to throw, assert Postgres shows no dangling group (or if using the compensating approach, assert `STATUS_99_ABANDONED` on both cases and group deletion).

---

**5. Stale `onSwitchToCaseDetails` prop still passed to `TaskDetailsModal`** — [`frontend/src/features/cases/components/view/TaskLogTab.tsx:636`](frontend/src/features/cases/components/view/TaskLogTab.tsx#L636)

**Problem.** `TaskDetailsModalProps` (in [`TasksDetailsModal.tsx:21-28`](frontend/src/features/cases/components/TasksDetailsModal.tsx#L21-L28)) no longer declares `onSwitchToCaseDetails`, but `TaskLogTab.tsx:636` still passes it. Under strict TypeScript this is a compile-time error; under looser settings the prop is silently dropped, leaving dead plumbing in `TaskLogTab` (the `onSwitchToCaseDetails?: () => void` prop and the destructuring at L51 and L66).

**Fix.** Remove the prop from the caller and from `TaskLogTab`'s own props if it isn't used elsewhere:

```diff
// frontend/src/features/cases/components/view/TaskLogTab.tsx
   type TaskLogTabProps = {
     ...
-    onSwitchToCaseDetails?: () => void;
     ...
   };

   const TaskLogTab: React.FC<TaskLogTabProps> = ({
     ...
-    onSwitchToCaseDetails,
     ...
   }) => {
     ...
     <TaskDetailsModal
       ...
       onClose={() => { setTaskDetailsModalOpen(false); }}
-      onSwitchToCaseDetails={onSwitchToCaseDetails}
     />
   };
```

Also grep the tree to confirm no other caller passes `onSwitchToCaseDetails` into `TaskLogTab`:

```bash
grep -rn "onSwitchToCaseDetails" frontend/src
```

If any parent (e.g., `ViewCaseModal`, `CaseModalsManager`) also passes this through, drop it there too.

---

### Non-blocking (please address in this PR if possible)

---

**6. Missing index on `cases.group_id`** — [`backend/prisma/migrations/20260701000001_add_investigation_group/migration.sql`](backend/prisma/migrations/20260701000001_add_investigation_group/migration.sql)

**Problem.** Grouped-case lookups are on the hot path — `AlertRepository.getGroupedCasesForAlert` fires every time an alert-detail modal or case-detail modal opens for a FRAUD_AND_AML alert/case, and drives `resolveCaseIds` in the case list. Without an index, this is a sequential scan that gets worse as the table grows.

**Fix — add a new migration** (do not edit the existing one, since it's already been applied in dev):

```sql
-- backend/prisma/migrations/20260706120000_index_cases_group_id/migration.sql
CREATE INDEX IF NOT EXISTS "cases_group_id_idx" ON "cases" ("group_id") WHERE "group_id" IS NOT NULL;
```

The partial index (`WHERE group_id IS NOT NULL`) keeps the index small — the vast majority of cases won't have a group. Update `schema.prisma` to declare the index so drift-detection doesn't complain:

```diff
   model Case {
     ...
     group_id  Int?
     ...
+    @@index([group_id])
   }
```

---

**7. `getAlertIdsWithInvestigationGroup` materializes group IDs into memory** — [`backend/src/modules/repository/alert.repository.ts:148-155`](backend/src/modules/repository/alert.repository.ts#L148-L155)

**Problem.** The NALT filter loads every tenant's group `alert_id` into memory and feeds them as `notIn` to `getReportStatusFilter`. This scales linearly with tenant FRAUD_AND_AML volume and forces Postgres to compile a query with a large `IN` list.

**Fix — replace with a raw `NOT EXISTS` subquery in the NALT filter itself:**

```ts
// wherever getReportStatusFilter uses the ID list for NALT (Alert.status = 'NALT'):
// change:
//   const excludedAlertIds = await this.alertRepository.getAlertIdsWithInvestigationGroup(tenantId, tx);
//   where.AND.push({ alert_id: { notIn: excludedAlertIds } });
// to a raw predicate:

where.AND.push({
  NOT: {
    alert_id: {
      in: this.prisma.$queryRaw`
        SELECT alert_id FROM "investigation_groups" WHERE tenant_id = ${tenantId}
      ` as unknown as number[],
    },
  },
});
```

**Cleaner alternative** — declare the Prisma relation between `Alert` and `InvestigationGroup` (see Issue 9 below), then the NALT filter becomes:

```ts
where.AND.push({ investigationGroup: { is: null } });
```

Prisma compiles this to a proper `NOT EXISTS` join. Recommended, and closes Issue 9 at the same time.

Delete `getAlertIdsWithInvestigationGroup` and its call site once the relation is in place.

---

**8. Add regression tests for the new failure branches**

The refactor introduced several new failure paths that CodeRabbit's happy-path tests don't exercise. Please add:

**`backend/test/case-closure-approval.service.spec.ts`** — missing investigation task:

```ts
it('closeCase: throws NotFoundException when no INVESTIGATE_CASE task is in a closeable status', async () => {
  caseRepositoryMock.findCaseWithPermissionCheck.mockResolvedValue({
    ...validCase,
    status: CaseStatus.STATUS_20_IN_PROGRESS,
    tasks: [
      { name: TASK_NAMES.INVESTIGATE_CASE, status: TaskStatus.STATUS_01_UNASSIGNED, /* not closeable */ },
    ],
  });
  await expect(caseClosureApprovalService.closeCase(caseId, userId, dto, user, endpointKey))
    .rejects.toBeInstanceOf(NotFoundException);
});
```

**`backend/test/triage.service.spec.ts`** — AI triage AML rollback:

```ts
it('handleAITriage FRAUD_AND_AML: on AML case creation failure, both primary + FRAUD are abandoned and group is deleted', async () => {
  caseCreateService.createCaseWithInvestigationTask
    .mockResolvedValueOnce({ caseId: fraudCaseId, message: '', taskId: 100 })   // FRAUD succeeds
    .mockRejectedValueOnce(new Error('AML boom'));                              // AML fails

  await expect(triageService.handleAITriage(alertId, caseId, dto, userId, tenantId))
    .rejects.toThrow('AML boom');

  expect(caseRepository.updateCase).toHaveBeenCalledWith(caseId, {
    status: CaseStatus.STATUS_99_ABANDONED, group_id: null,
  });
  expect(caseRepository.updateCase).toHaveBeenCalledWith(fraudCaseId, {
    status: CaseStatus.STATUS_99_ABANDONED, group_id: null,
  });
  expect(prisma.investigationGroup.delete).toHaveBeenCalledWith({ where: { id: investigationGroupId } });
  expect(flowableService.handleCaseAbandoned).toHaveBeenCalledTimes(2);
  expect(loggingOrchestrationService.logActions).toHaveBeenCalledWith(
    expect.objectContaining({ operation: 'AI_TRIAGE_FRAUD_AND_AML_ROLLED_BACK' }),
  );
});

it('handleAITriage FRAUD_AND_AML: when rollback itself fails, logs AI_TRIAGE_FRAUD_AND_AML_ROLLBACK_FAILED and rethrows original AML error', async () => {
  // Force updateCase to reject during rollback; assert original error is rethrown, not the rollback error.
});
```

**`backend/test/triage.service.spec.ts`** — stale-alert regression (fix from Blocking #1):

```ts
it('handleManualTriage FRAUD_AND_AML: returns alert with case_id = null after AML sibling created', async () => {
  // stub alertService.updateAlert to record calls; last call should resolve with case_id: null
  const result = await triageService.handleManualTriage(alertId, dto, userId, tenantId);
  expect(result.case_id).toBeNull();
});
```

**`backend/test/task-lifecycle.service.spec.ts`** — SAR/STR Filing gate:

```ts
it('unassignTask SAR/STR Filing: does not change case status in Postgres OR Flowable', async () => {
  await taskLifecycleService.unassignTask(sarStrTaskId, actorUserId, tenantId, 'reason', user, endpointKey);
  expect(prismaTx.case.update).not.toHaveBeenCalled();
  expect(flowableService.handleCaseStatusChanged).not.toHaveBeenCalled();
  expect(flowableService.handleTaskUnassigned).toHaveBeenCalled();
});
```

---

**9. Add Prisma `@relation` for `group_id` and `alert_id`** — [`backend/prisma/schema.prisma:113`](backend/prisma/schema.prisma#L113)

**Problem.** `Case.group_id` and `InvestigationGroup.alert_id` are plain scalar fields. Prisma doesn't enforce the FK at the client level, doesn't offer navigation properties (so we can't do `case.investigationGroup.alert_id` in a single query), and hand-written joins across the two are what forces Issue 7's in-memory pattern.

**Fix.**

```diff
   model Case {
     ...
     group_id  Int?
+    investigationGroup InvestigationGroup? @relation(fields: [group_id], references: [id])
     ...
     @@index([group_id])
   }

   model InvestigationGroup {
     id         Int      @id @default(autoincrement())
     alert_id   Int      @unique
     tenant_id  String
     created_at DateTime @default(now())
+    alert      Alert    @relation(fields: [alert_id], references: [alert_id])
+    cases      Case[]
   }

   model Alert {
     ...
     alert_id  Int  @id @default(autoincrement())
+    investigationGroup InvestigationGroup?
     ...
   }
```

Ship as a new migration (Prisma will emit an `ADD CONSTRAINT` — verify the FK actually adds cleanly against existing data before merging; the backfill script should have made this safe). This is out of scope for the immediate blockers but pairs cleanly with Non-blocking #7 (relation-based NOT EXISTS filter), which is why I'd suggest doing both together.

If you'd rather defer this — that's fine, but please capture it as tech debt so it doesn't get lost.

---

### One last thing

The commit series here is 77 commits with ~15 merge commits back from `dev`. Once this round of fixes lands, please consider **squashing** on merge — the interleaved "fix: code rabbit fixes" / merge / "fix: <same area>" commits make `git bisect` painful for anyone chasing a regression in this area later.
`````

[↑ Back to top](#pr-review-cms-233--replace-fraud_and_aml-container-cases-with-investigationgroup-linkage)

---
---
---

## Follow-up Review (2026-07-06)

**Reviewed commit:** `084a48f0` — *"Merge branch 'paysys/fraud_and_aml'"* (2026-07-06 15:01 UTC)
**Reviewed against:** `Changes Requested` posted on commit `d5902b5f` by `ahmad-paysys` (2026-07-06 12:50 UTC)
**New commits in this round (post `d5902b5f`):**
- `9d193199` — *fix: tx-plumbing changes*
- `dfb20008` — *refactor: complete case creation made atomic and compensate rollback*
- `e6a9319a` — *Add Prisma relation for group_id and alert_id*
- `a8a58fe0` — *fix: SAR/STR Filing Flowable status made atomic*
- `5644f5f1` — *fix: Add regression tests for the new failure branches*
- `db26c5ae` — *fix: handleManualTriage transaction escape fixed*
- `21704db0` — *fix: getAlertIdsWithInvestigationGroup materializes group IDs into memory*
- `c2158a04` — *refactor: dead code removed*
- `c6edd83a` — *fix: resolve lint error* (+ merge commits)

**Developer response:**
> "@ahmad-paysys all the reported issues are resolved. Please review again." — MuhammadAli-Paysys, 2026-07-06 15:01 UTC

All CI checks green (Node.js CI build/style/tests, CodeQL, njsscan, hadolint, encoding, dependency review, gpg-verify, CodeRabbit) except DCO — one commit missing sign-off. Not blocking review, but should be fixed before merge.

---

### Changes Requested — Resolution Status

### Item 1 — `handleManualTriage` returns stale `alert` payload (Blocking #1)

**Status: RESOLVED**

`backend/src/modules/triage/triage.service.ts:401-410` (commit `db26c5ae`):

```ts
const detachedAlert = await this.alertService.updateAlert(
  alertId,
  userId,
  { caseId: null } as unknown as UpdateAlertDTO,
  tx,
);

return { alert: detachedAlert, completeNewCaseTask, existingCase };
```

The final `updateAlert` result is captured as `detachedAlert` and returned to callers. Non-FRAUD_AND_AML branch still returns the earlier `alert` object at L422, which is correct — only the FRAUD_AND_AML branch performs the second `caseId: null` write, and only that branch needs to substitute the returned payload.

---

### Item 2 — `completeCaseCreation` FRAUD/AML split not atomic (Blocking #2)

**Status: RESOLVED**

`backend/src/modules/case/case.service.ts:897-1001` (commit `dfb20008`). The `$transaction` block now covers everything the prior round left outside:

```ts
const result = await this.prismaService.$transaction(async (prisma) => {
  let investigationGroup: { id: number } | undefined;
  if (isSupervisor && isFraudNAML) {
    const alertId = await this.alertRepository.getAlertByCaseId(caseId, tenantId, prisma);
    investigationGroup = await this.investigationGroupService.createInvestigationGroup(alertId, tenantId, prisma);
  }
  // ... case updates + task completion + comment ...
  const alertId = await this.alertRepository.getAlertByCaseId(caseId, tenantId, prisma);
  if (alertId) {
    await this.alertRepository.updateAlert(alertId, { ..., caseId: isFraudNAML && isSupervisor ? null : caseId } as unknown as UpdateAlertDTO, prisma);
  }
  let amlCase: ... = null;
  if (isFraudNAML && isSupervisor) {
    amlCase = await this.caseCreationService.createCaseWithInvestigationTask(
      CaseType.AML, userId, existingCase.tenant_id, updatedCase.priority,
      CaseCreationType.AUTOMATIC_SYSTEM, role, investigationGroup!.id, prisma,
    );
  }
  return { case: updatedCase, completedTask, amlCase, investigationGroup, isAutoCloseEligible };
});
```

Group creation, alert update, and AML sibling creation are all threaded through the tx client. Flowable calls (`handleCaseStatusChanged`, `handleTaskCompleted`) remain after `$transaction` commits — correct. The two authoritative alert writes have been collapsed into a single conditional inside the tx, eliminating the previous double-write.

---

### Item 3 — SAR/STR Filing Flowable status drift (Blocking #3)

**Status: RESOLVED**

`backend/src/modules/task/services/task-lifecycle.service.ts:231-241` (commit `a8a58fe0`):

```ts
if (updatedTask.name !== 'SAR/STR Filing') {
  await tx.case.update({
    where: { case_id: existingTask.case_id },
    data: { status: CaseStatus.STATUS_02_READY_FOR_ASSIGNMENT, case_owner_user_id: null, updated_at: new Date() },
  });

  await this.flowableService.handleCaseStatusChanged({
    caseId: existingTask.case_id,
    newStatus: CaseStatus.STATUS_02_READY_FOR_ASSIGNMENT,
  });
}
```

Both the Postgres update **and** the Flowable status notification are now inside the same predicate. `handleTaskUnassigned` remains outside the gate at L243-248 — correct, since the task really was unassigned regardless of case type. BPMN/Postgres drift for SAR/STR Filing unassigns is closed.

---

### Item 4 — `handleManualTriage` FRAUD/AML writes outside tx (Blocking #4)

**Status: RESOLVED**

`backend/src/modules/triage/triage.service.ts:366-410` (commits `9d193199`, `db26c5ae`). All three previously-escaping writes now receive `tx`:

```ts
if (alert.alert_type === CaseType.FRAUD_AND_AML) {
  const investigationGroup = await this.investigationGroupService.createInvestigationGroup(alert.alert_id, tenantId, tx);
  await this.caseCreationService.updateCaseStatus(
    alert.case_id, CaseStatus.STATUS_02_READY_FOR_ASSIGNMENT, userId, tenantId,
    priority, CaseType.FRAUD, investigationGroup.id, tx,
  );

  const amlCase = await this.caseCreateService.createCaseWithInvestigationTask(
    CaseType.AML, userId, tenantId, priority, CaseCreationType.AUTOMATIC_SYSTEM,
    'SUPERVISOR', investigationGroup.id, tx,
  );
  ...
}
```

The tx-plumbing suggested in the prior review (extending `createInvestigationGroup`, `updateCaseStatus`, and `createCaseWithInvestigationTask` to accept an optional `Prisma.TransactionClient`) has been applied consistently. Flowable emissions from `createCaseWithInvestigationTask` are gated on `!tx` so they defer to `executeFlowableOperations` after the surrounding transaction commits — matching the pattern already established in `CaseCreationApprovalService`.

---

### Item 5 — Stale `onSwitchToCaseDetails` prop passed to `TaskDetailsModal` (Blocking #5)

**Status: RESOLVED**

`grep -rn "onSwitchToCaseDetails" frontend/src` returns no matches at head `084a48f0`. The stale prop, its destructuring, and its type declaration in `TaskLogTab.tsx` are all removed (3 lines dropped per the git stat). Type check passes.

---

### Item 6 — Add index on `cases.group_id` (Non-blocking #6)

**Status: RESOLVED**

New migration `backend/prisma/migrations/20260702140000_add_investigation_group_relations/migration.sql:14` (commit `e6a9319a`):

```sql
CREATE INDEX "cases_group_id_idx" ON "cases"("group_id");
```

Corresponding `@@index([group_id])` added in `schema.prisma:124`. The index is not partial (`WHERE group_id IS NOT NULL`) as I'd suggested — a small nit given that most `cases` rows will have `group_id = NULL`, so the index is larger than necessary. Not blocking; happy to accept as-is since Postgres b-tree indexes handle nullable columns efficiently in practice.

---

### Item 7 — `getAlertIdsWithInvestigationGroup` materializes group IDs (Non-blocking #7)

**Status: RESOLVED**

`getAlertIdsWithInvestigationGroup` and its callers are gone (commits `21704db0`, `c2158a04`). `grep -rn "getAlertIdsWithInvestigationGroup" backend/src/` returns no matches. `getAlertByCaseId` now uses the Prisma relation directly:

```ts
const caseRecord = await client.case.findFirst({
  where: { case_id: caseId, tenant_id: tenantId },
  select: {
    alert: { select: { alert_id: true } },
    investigationGroup: { select: { alert_id: true } },
  },
});
const alertId = caseRecord?.alert?.alert_id ?? caseRecord?.investigationGroup?.alert_id;
```

Single query, no in-memory list, no N-scoping issue. Clean.

---

### Item 8 — Regression tests for new failure branches (Non-blocking #8)

**Status: RESOLVED**

Commit `5644f5f1` extends four spec files:

- `backend/test/case-closure-approval.service.spec.ts` — `NotFoundException` branch for missing investigation task.
- `backend/test/case.service.spec.ts` — `completeCaseCreation` FRAUD/AML happy path + rollback + stale-alert regression.
- `backend/test/task-lifecycle.service.spec.ts` — SAR/STR Filing unassign should NOT touch case status or fire `handleCaseStatusChanged`.
- `backend/test/triage.service.spec.ts` — AI triage rollback path + `handleManualTriage` returning `case_id: null`.

Diff stat shows +91 lines added to `triage.service.spec.ts` and comparable additions to the others. Node.js CI `check tests` job passed at 15:07 UTC.

---

### Item 9 — Prisma `@relation` for `group_id` and `alert_id` (Informational)

**Status: RESOLVED**

`backend/prisma/schema.prisma` (commit `e6a9319a`):

```prisma
model Case {
  ...
  group_id            Int?
  investigationGroup  InvestigationGroup? @relation(fields: [group_id], references: [id])
  @@index([group_id])
}

model InvestigationGroup {
  id         Int    @id @default(autoincrement())
  alert_id   Int    @unique
  ...
  alert      Alert  @relation(fields: [alert_id], references: [alert_id])
  cases      Case[]
}

model Alert {
  ...
  investigationGroup InvestigationGroup?
}
```

New migration `20260702140000_add_investigation_group_relations` adds the FK constraints (`ON DELETE SET NULL` for `cases.group_id`, `ON DELETE RESTRICT` for `investigation_groups.alert_id` — sensible), plus renames the PK constraint to Prisma's naming convention. The migration header explicitly documents that data was validated against the constraints before merging. Nice touch.

---

### Resolution Summary Table

| # | Item | Status |
|---|------|--------|
| 1 | `handleManualTriage` returns stale `alert` payload | Resolved |
| 2 | `completeCaseCreation` FRAUD/AML split not atomic | Resolved |
| 3 | SAR/STR Filing Flowable status drift | Resolved |
| 4 | `handleManualTriage` FRAUD/AML writes outside tx | Resolved |
| 5 | Stale `onSwitchToCaseDetails` prop | Resolved |
| 6 | Missing index on `cases.group_id` | Resolved |
| 7 | `getAlertIdsWithInvestigationGroup` in-memory list | Resolved (removed) |
| 8 | Regression tests for new failure branches | Resolved |
| 9 | Prisma `@relation` for `group_id`/`alert_id` | Resolved |

All 5 blocking items and all 4 non-blocking/informational items are resolved.

---

### New Issues Found in Updated Commits

None. I reviewed the delta between `d5902b5f..084a48f0` (18 files, +456/-279) end-to-end:

- `case.service.ts` — the new atomic path uses `getAlertByCaseId` twice inside the tx (once at L901 for the group creation, once at L966 for the alert update). Both calls receive the tx client, both are cheap indexed reads, so the double-lookup is a minor style nit rather than a bug. Not worth flagging.
- `case-creation.service.ts` — the `createCase(..., tx)` signature now has `executeFlowableCaseCreationEvent` guarded by `if (!tx)`. Callers inside a tx are responsible for firing Flowable after commit. `CaseCreationApprovalService.approveCaseCreation` and the new atomic `completeCaseCreation` both do this correctly.
- `alert.repository.ts` — the new relation-based `getAlertByCaseId` uses a single `findFirst` with nested selects. Behavior matches the prior separate-query path, and the tenant guard still bounds the underlying `case_id` lookup.
- `triage.service.ts` — the manual-triage path now returns `detachedAlert` from the FRAUD_AND_AML branch; the AI triage compensating rollback is unchanged (still correct).
- Prisma schema + migration — FK constraints are sensible; the `RENAME CONSTRAINT` on `investigation_groups_id → investigation_groups_pkey` is safe (idempotent, no data touched).
- Frontend — only 3 lines removed in `TaskLogTab.tsx`, dropping the stale prop. No new frontend regressions.

---

### Updated Verdict

**Verdict: Approved**

All blocking items from the initial review are resolved with correct, well-scoped fixes. The transactional design across `completeCaseCreation`, `handleManualTriage`, and the SAR/STR Filing gate is now internally consistent and matches the pattern already established in `CaseCreationApprovalService`. The Prisma relation cleanup (Item 9) went further than strictly required — it enabled the simplification in `getAlertByCaseId` (Item 7) as a bonus, removing the in-memory join entirely.

Test coverage was strengthened proportionally to the changes — every failure branch flagged in the prior round has an explicit unit test now.

**One small pre-merge cleanup (not blocking):**
- The `dco-check / dco` CI job is failing. One commit in the recent stack is missing `Signed-off-by:`. Rebase to add sign-off (or squash on merge and add a single sign-off to the squashed commit) before merging. The upstream repo enforces DCO; the merge button will be blocked until this is green.

**Recommended for follow-up (out of scope, capture as tech debt):**
- Squash-merge the 90+ commit branch to keep `git bisect` sane in this area of the code.
- Consider whether the double `getAlertByCaseId` in the new `completeCaseCreation` tx path can collapse into a single lookup (minor).

[↑ Back to top](#pr-review-cms-233--replace-fraud_and_aml-container-cases-with-investigationgroup-linkage)

---

### GitHub Review Comment

`````markdown
**Approved**

All five blocking items from the previous round are resolved with correct, well-scoped fixes — I re-verified each one against head `084a48f0`. The transactional design across `completeCaseCreation`, `handleManualTriage`, and the SAR/STR Filing gate is now internally consistent and matches the `CaseCreationApprovalService` pattern. The Prisma relation cleanup went beyond what was strictly required and enabled removing `getAlertIdsWithInvestigationGroup` and its in-memory join entirely — a nice bonus.

### Resolution summary

| # | Item | Status |
|---|------|--------|
| 1 | `handleManualTriage` returns stale `alert` payload | Resolved (`triage.service.ts:401-410`) |
| 2 | `completeCaseCreation` FRAUD/AML split not atomic | Resolved (`case.service.ts:897-1001`) |
| 3 | SAR/STR Filing Flowable status drift | Resolved (`task-lifecycle.service.ts:231-241`) |
| 4 | `handleManualTriage` FRAUD/AML writes outside tx | Resolved (`triage.service.ts:366-410`) |
| 5 | Stale `onSwitchToCaseDetails` prop | Resolved (removed from `TaskLogTab.tsx`) |
| 6 | Missing index on `cases.group_id` | Resolved (`20260702140000_add_investigation_group_relations`) |
| 7 | `getAlertIdsWithInvestigationGroup` in-memory list | Resolved (function removed, uses Prisma relation) |
| 8 | Regression tests for new failure branches | Resolved (4 spec files extended in `5644f5f1`) |
| 9 | Prisma `@relation` for `group_id`/`alert_id` | Resolved (`schema.prisma`) |

Nice work threading `tx` cleanly through `createInvestigationGroup`, `updateCaseStatus`, and `createCaseWithInvestigationTask` — that's exactly the plumbing the previous review asked for, applied consistently across every caller.

---

### One small pre-merge cleanup (not blocking review, but blocks merge)

**DCO check is failing** — one commit in this stack is missing `Signed-off-by:`. Either rebase to sign the offending commit, or squash on merge and ensure the squashed commit is signed. The repo enforces DCO so the merge button will remain disabled until this is green.

```bash
# Identify the unsigned commit(s):
git log --format='%H %s%n%b' origin/dev..HEAD | grep -B1 -L 'Signed-off-by:'
```

---

### Follow-up recommendations (out of scope for this PR, capture as tech debt)

- **Squash-merge this branch.** 90+ commits with ~20 merge commits back from `dev` will make `git bisect` painful for anyone tracking a regression in this area later. Squash-on-merge collapses it to a single meaningful commit against `dev`.
- **Minor: double `getAlertByCaseId` inside the new `completeCaseCreation` tx.** The atomic path calls it at both `case.service.ts:901` and `:966`. Both are inside the tx and cheap, but a single lookup passed via local variable would be marginally cleaner.

Otherwise, ready to merge once DCO is green.
`````

[↑ Back to top](#pr-review-cms-233--replace-fraud_and_aml-container-cases-with-investigationgroup-linkage)

---

## Flowable Integration Follow-up (2026-07-07)

**Reviewed commit:** `084a48f0` (unchanged since Follow-up Review)
**Focus:** BPMN/Flowable interaction surfaces and residual atomicity gaps across the Postgres ↔ Flowable boundary.
**Scope note:** this pass narrows on the Flowable-facing risks that survive the previous rounds' fixes. It is additive to the "Approved" verdict — none of the items below re-open blocking status, but each is a concrete failure mode worth capturing before merge or as follow-up tech debt.

---

### GitHub Review Comment

`````markdown
**Comment (non-blocking) — Flowable atomicity follow-up**

The Postgres side of this refactor is in great shape after the last round. Re-reading the code specifically through the Flowable lens, though, there are five residual gaps at the Postgres↔Flowable boundary that are worth flagging. None re-open blocking status — the DB writes are all correct and the previous round's transactional discipline holds — but each is a concrete failure mode where Postgres and the BPMN engine can drift out of sync with no retry/outbox to recover.

Filing as a comment rather than Changes Requested because (a) the DB state is always the source of truth in these paths, (b) partial-emission scenarios are best-effort by design elsewhere in the codebase, and (c) a proper fix (transactional outbox) is bigger than this PR. But please capture as tech debt.

---

### 1. `completeCaseCreation` — Flowable emissions run post-commit with no retry

**Location:** [`backend/src/modules/case/case.service.ts:910-925`](backend/src/modules/case/case.service.ts#L910-L925)

The `$transaction` at [case.service.ts:897-908](backend/src/modules/case/case.service.ts#L897-L908) correctly commits the AML sibling creation, `group_id` linkage, and the task-completion row in a single atomic step. But `handleCaseStatusChanged` and `handleTaskCompleted` fire **after** the commit, outside any retry envelope. If Flowable is unreachable or the process crashes between the commit and the emission, Postgres reflects "AML sibling created / task completed" while Flowable never learns of either event. There's no visible outbox or retry mechanism.

**Consequence:** BPMN process instance stays in the pre-completion state; downstream Flowable-driven task creation will make invalid transitions or produce wrong tasks. Operators would have to reconcile manually.

**Suggested follow-up:** transactional outbox pattern — write an `outbox_event` row inside the same tx, drained by a background worker with retry. Out of scope for this PR.

---

### 2. `handleManualTriage` — deferred AML `handleCaseCreated` needs verification

**Location:** [`backend/src/modules/triage/triage.service.ts:366-410`](backend/src/modules/triage/triage.service.ts#L366-L410) + [`backend/src/modules/case/services/case-creation.service.ts` (`createCase` `!tx` guard)](backend/src/modules/case/services/case-creation.service.ts)

In the FRAUD_AND_AML manual-triage branch, `createCaseWithInvestigationTask` is invoked with `tx`. That call chains into `createCase`, which now guards `executeFlowableCaseCreationEvent` behind `if (!tx)` to avoid emitting inside the transaction. This is the right pattern in principle — Flowable can't roll back, so the emission must be deferred.

**But:** I cannot see, in the visible hunks, where the deferred `handleCaseCreated` for the AML sibling is fired **after** the tx commits in `handleManualTriage`. The approval-side path (`case-creation-approval.service.ts:1455-1468`) does this explicitly. The manual-triage path shows the sibling creation, comment insert, and alert detach — but no post-commit emission for the AML sibling in the hunks I audited.

**Consequence if truly missing:** the AML case exists in Postgres but Flowable never gets a `caseCreated` event, so its BPMN process instance is never started. All downstream task creation for that AML case would be Postgres-only.

**Requested action before merge:** please confirm the post-commit `handleCaseCreated({ caseId: amlCase.caseId, ... })` is fired in `handleManualTriage`. If it isn't, mirror the approval-side pattern:

```ts
const txResult = await this.alertRepository.transaction(async (tx) => { ... });

if (isFraudAndAmlPath) {
  this.flowableService.handleCaseCreated({ caseId: txResult.amlCase.caseId, ... });
}
```

---

### 3. AI-triage rollback is best-effort and non-awaited

**Location:** [`backend/src/modules/triage/triage.service.ts:3136-3195`](backend/src/modules/triage/triage.service.ts#L3136-L3195)

The compensating rollback in `handleAITriage` (when AML sibling creation fails after FRAUD is already created) does:

- `Promise.all([caseRepository.updateCase(caseId, ABANDONED), caseRepository.updateCase(fraudCase.caseId, ABANDONED)])` — good, parallel and awaited.
- `flowableService.handleCaseAbandoned({ caseId, reason })` — **not awaited**.
- `flowableService.handleCaseAbandoned({ caseId: fraudCase.caseId, reason })` — **not awaited**.
- `prisma.investigationGroup.delete(...)` — awaited.

The two `handleCaseAbandoned` calls being fire-and-forget means:
1. Their errors never affect control flow, so a Flowable outage during rollback is invisible.
2. If the process crashes before those calls resolve, the abandonment is Postgres-only.

The existing `AI_TRIAGE_FRAUD_AND_AML_ROLLBACK_FAILED` audit log covers the "rollback itself throws" case (the outer catch), but not "the Flowable notification silently dropped."

**Suggested fix (low-risk):** `await` both `handleCaseAbandoned` calls and put them in a `Promise.allSettled` so a Flowable failure gets logged as `AI_TRIAGE_FRAUD_AND_AML_ROLLBACK_PARTIAL` while still allowing the group deletion to proceed:

```ts
const flowableResults = await Promise.allSettled([
  this.flowableService.handleCaseAbandoned({ caseId, reason: rollbackReason }),
  this.flowableService.handleCaseAbandoned({ caseId: fraudCase.caseId, reason: rollbackReason }),
]);
const flowableFailures = flowableResults.filter((r) => r.status === 'rejected');
if (flowableFailures.length > 0) {
  this.logger.error(`AI_TRIAGE_FRAUD_AND_AML_ROLLBACK_PARTIAL: ${flowableFailures.length}/2 Flowable notifications failed`, ...);
  await this.loggingOrchestrationService.logActions({ operation: 'AI_TRIAGE_FRAUD_AND_AML_ROLLBACK_PARTIAL', ... });
}
```

---

### 4. Backfilled AML "history" tasks have no Flowable counterpart

**Location:** [`backend/src/modules/case/services/case-creation-approval.service.ts:1377-1401`](backend/src/modules/case/services/case-creation-approval.service.ts#L1377-L1401)

The approval-side FRAUD/AML split inserts `STATUS_30_COMPLETED` "Complete New Case" and "Approve Case Creation" rows on the AML sibling to make its timeline mirror the FRAUD case. The in-code comment already acknowledges this: *"BPMN never creates these tasks on the isFraudNAML=true route ... these are Postgres-only historical stubs."*

This is a deliberate design choice, not a bug — but it does mean:
- Any external reconciler comparing Postgres `task` history against Flowable process history will surface false-positive orphans on every AML sibling created via the approval path.
- Ops dashboards that count tasks per BPMN process instance will see a persistent skew.

**Suggested follow-up:** mark these Postgres-only rows with a boolean column (e.g., `task.is_historical_stub: bool`) or a sentinel `origin` enum so downstream consumers can filter. Out of scope for this PR.

---

### 5. `alertService.updateAlert` inside triage tx — event emission timing

**Location:** [`backend/src/modules/triage/triage.service.ts:3081-3088`](backend/src/modules/triage/triage.service.ts#L3081-L3088)

The final `alertService.updateAlert(alertId, userId, { caseId: null }, tx)` in `handleManualTriage` is called with the tx client. If `alertService.updateAlert` internally emits any side-effect event (Kafka, webhook, Flowable message) beyond the DB write, that event fires **before** the enclosing tx commits — so a subsequent rollback would leave the event emitted against unreached Postgres state.

I couldn't confirm from the diff whether `alertService.updateAlert` has any emission side-effects; the surrounding code doesn't suggest any, but worth verifying.

**Requested action:** confirm `alertService.updateAlert` is a pure DB write with no external emission. If it does emit anything, move that emission to a post-commit hook or an outbox row.

---

### Summary

| # | Item | Severity | Blocking? |
|---|------|----------|-----------|
| 1 | `completeCaseCreation` post-commit Flowable calls have no retry | Minor (design) | No — tech debt |
| 2 | `handleManualTriage` deferred AML `handleCaseCreated` unverified | **Major (if missing)** | Please confirm before merge |
| 3 | AI-triage rollback `handleCaseAbandoned` calls not awaited | Minor | No |
| 4 | Backfilled AML history tasks orphaned in Flowable | Informational | No — design choice, capture as tech debt |
| 5 | `alertService.updateAlert` inside tx — verify no external emission | Informational | Please confirm |

Items 2 and 5 are the only ones I'd like a quick confirmation on before merge — both are "please verify against the file, not the diff" asks rather than change requests. Everything else is genuinely follow-up.
`````

[↑ Back to top](#pr-review-cms-233--replace-fraud_and_aml-container-cases-with-investigationgroup-linkage)
