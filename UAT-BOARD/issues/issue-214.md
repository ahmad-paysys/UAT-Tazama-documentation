# Issue #214 — FRAUD_AND_AML "container" cases modelled as peer Case rows

**Link:** https://github.com/tazama-lf/case-management-system/issues/214
**Author:** Justus-at-Tazama
**Assignee:** sobia-rizwan1567
**Status:** Open · No comments
**Labels:** None
**Report Date:** 2026-06-29

---

## Executive Summary

A `FRAUD_AND_AML` alert creates **three `Case` rows** in the database: one "container" parent (`FRAUD_AND_AML` type) and two child investigations (`FRAUD` and `AML`). The container has no real investigator, no real lifecycle, and no real verdict — yet it is modelled as a full `Case` entity. To make this work, the codebase compensates with a synthetic terminal status (`STATUS_84_COMPLETED`), a permission bypass, and hand-coded parent↔child status propagation spread across **four backend services**. Additionally, because the report service has no awareness of container rows, every `FRAUD_AND_AML` alert is **triple-counted** in all reports, and the `STATUS_84` close never lands in any outcome slice — so closed totals are permanently irreconcilable.

This is a significant architectural defect with compounding side effects. It is not a UI bug or a missing feature — it is a modelling error at the domain layer that produces incorrect data in every report and requires bespoke workarounds in creation, permissions, lifecycle, closure, and the frontend.

---

## How the system works today (confirmed in code)

### 1. Three Case rows per FRAUD_AND_AML alert

In [`triage.service.ts`](case-management-system/backend/src/modules/triage/triage.service.ts), when `alert.alert_type === CaseType.FRAUD_AND_AML`, the code calls `createCaseWithInvestigationTask` twice in sequence — once for `CaseType.FRAUD` and once for `CaseType.AML` — after the container case has already been created:

```ts
// triage.service.ts ~L372-L393
if (alert.alert_type === CaseType.FRAUD_AND_AML) {
  await this.caseCreateService.createCaseWithInvestigationTask(
    CaseType.FRAUD, userId, tenantId, alert.case_id, priority, ...
  );
  await this.caseCreateService.createCaseWithInvestigationTask(
    CaseType.AML, userId, tenantId, alert.case_id, priority, ...
  );
}
```

The same pattern repeats at `triage.service.ts` ~L582 for the system-triage path.

In [`case-creation.service.ts`](case-management-system/backend/src/modules/case/services/case-creation.service.ts), `createCaseWithInvestigationTask` creates the child with `parentId: parentCaseId` wired into the Prisma create call. The Prisma schema ([`backend/prisma/schema.prisma`](case-management-system/backend/prisma/schema.prisma) L112) makes `parent_id Int?` on the `Case` model — nullable self-reference.

Result: **three `Case` rows** per `FRAUD_AND_AML` alert, where `parent_id` is set only on the two children.

---

### 2. Synthetic STATUS_84_COMPLETED

The BPMN process (`cms.bpmn20.xml`) only offers three valid closure outcomes for the `closeCase` user task:

```xml
<!-- cms.bpmn20.xml L201-L204 -->
<flowable:value id="STATUS_82_CLOSED_CONFIRMED" name="STATUS_82_CLOSED_CONFIRMED"/>
<flowable:value id="STATUS_81_CLOSED_REFUTED"   name="STATUS_81_CLOSED_REFUTED"/>
<flowable:value id="STATUS_83_CLOSED_INCONCLUSIVE" name="STATUS_83_CLOSED_INCONCLUSIVE"/>
```

There is no `STATUS_84` in the BPMN. Instead, the **frontend overrides** the `recommendedOutcome` field to `STATUS_84_COMPLETED` hardcoded in JavaScript when the case type is `FRAUD_AND_AML`:

```ts
// CloseCaseModal.tsx L100-L107
if (caseData?.type === 'FRAUD_AND_AML') {
  const combined = 'STATUS_84_COMPLETED';
  setFormData((prev) => ({
    ...prev,
    recommendedOutcome: combined as CloseCaseDto['recommendedOutcome'],
  }));
}
```

This is the **only code path** that produces `STATUS_84_COMPLETED`. It circumvents the BPMN closure form entirely.

`STATUS_84_COMPLETED` is then treated as a closed status in [`case.constants.ts`](case-management-system/backend/src/constants/case.constants.ts):
- Included in `CLOSED_CASE_STATUSES` (L20)
- Included in `CASE_CLOSURE_OUTCOMES` (L10)
- Included in `INACTIVE_CASE_STATUSES` (L43)
- **Excluded from `REOPENABLE_CASE_STATUSES`** — a container can never be reopened even if a child is

The frontend also has a dedicated style for it in [`casesTable.utils.ts`](case-management-system/frontend/src/features/cases/components/casesTable.utils.ts) L47:
```ts
STATUS_84_COMPLETED: 'bg-green-50 text-green-700',
```

And it is selectable in `CaseFilters.tsx` status dropdown as "Completed" (L68).

---

### 3. No BPMN coordination — exclusive gateway, not parallel

The `isFraudNAMLGateway` in the BPMN is an **exclusive gateway** (`exclusiveGateway`):

```xml
<!-- cms.bpmn20.xml L28, L198-L206 -->
<exclusiveGateway id="isFraudNAMLGateway" name="Is Fraud and AML Case?"/>
...
<sequenceFlow id="flowIsFraudNAML" sourceRef="isFraudNAMLGateway" targetRef="investigateCase">
    <conditionExpression>${isFraudNAML == true}</conditionExpression>
</sequenceFlow>
<sequenceFlow id="flowNotFraudNAML" sourceRef="isFraudNAMLGateway" targetRef="completeCaseCreation">
    <conditionExpression>${isFraudNAML == false}</conditionExpression>
</sequenceFlow>
```

When `isFraudNAML == true`, the BPMN routes to the **same `investigateCase` user task** as a normal single case. There is **no parallel gateway, no call activity, no sub-process, and no multi-instance** anywhere in the file. The entire coordination ("wait for both FRAUD and AML to close, then close the container") is implemented in application code outside Flowable.

---

### 4. Hand-coded parent↔child status propagation across four services

Every significant state transition in a child case checks `if (updatedCase.parent_id)` and manually mirrors the status to the parent. This exists in four separate services:

#### `task.service.ts` — `promoteParentCaseToInProgress` (L303–L345)
Called when a task is assigned and a child case moves to `STATUS_20_IN_PROGRESS`. Checks the sibling's status; if both children are in-progress (or one is closed), promotes the parent to `STATUS_20_IN_PROGRESS`.

#### `task-lifecycle.service.ts` — three locations (L65, L168, L273)
- **L65** (task assignment): if both children reach `STATUS_10_ASSIGNED`, mirrors `STATUS_10_ASSIGNED` to parent
- **L168**: additional task-assignment propagation path
- **L273**: propagation on further task state changes

#### `case.service.ts` (suspend/resume — L90, L229)
On **suspension**: if both children are suspended (`STATUS_21_SUSPENDED`), suspends the parent too. On **resume** (L229): mirrors sibling state back to parent.

```ts
// case.service.ts ~L90-L103
if (updatedCase.parent_id) {
  const subCase = await prisma.case.findFirst({
    where: { parent_id: updatedCase.parent_id, NOT: { case_id: updatedCase.case_id } }
  });
  if (updatedCase.status === CaseStatus.STATUS_21_SUSPENDED && subCase?.status === CaseStatus.STATUS_21_SUSPENDED) {
    await prisma.case.update({ where: { case_id: updatedCase.parent_id }, data: { status: CaseStatus.STATUS_21_SUSPENDED } });
  }
}
```

#### `case-reopening.service.ts` (L68, L244)
When a child is reopened, mirrors the parent back to `STATUS_20_IN_PROGRESS`. A child can be reopened (its status is in `REOPENABLE_CASE_STATUSES`), but the parent (`STATUS_84`) cannot — so a reopened child creates an **irreconcilable state** where the parent is "Completed" but a child is active again.

---

### 5. Permission bypass for FRAUD_AND_AML container

In [`case.repository.ts`](case-management-system/backend/src/modules/repository/case.repository.ts) `findCaseWithPermissionCheck` (L185–L196), `FRAUD_AND_AML` cases are in their own OR-branch that **skips all owner/assignee checks**:

```ts
// case.repository.ts ~L185-L196
whereCondition.OR = [
  {
    case_type: { in: ['FRAUD_AND_AML'] },   // ← any in-tenant user can access this
  },
  {
    AND: [
      { case_type: { notIn: ['FRAUD_AND_AML'] } },
      { OR: [
        { case_owner_user_id: userId },
        { tasks: { some: { assigned_user_id: userId, ... } } },
      ]},
    ],
  },
];
```

The container has `case_owner_user_id: null` (set on creation in `case-creation.service.ts`) — so without this bypass, no one could access it. The bypass is required because the container's design demands it.

---

### 6. Closure guard: only supervisors, only when both children are closed

In [`case-closure-approval.service.ts`](case-management-system/backend/src/modules/case/services/case-closure-approval.service.ts) L112–L145:

```ts
const isFraudAndAmlCase = caseData.case_type === 'FRAUD_AND_AML';
if (isFraudAndAmlCase) {
  if (role === 'CMS_SUPERVISOR') {
    const subCases = await this.prismaService.case.findMany({
      where: { parent_id: caseId, tenant_id: caseData.tenant_id }
    });
    const areSubCasesClosable = subCases.every((c) => CLOSED_CASE_STATUSES.includes(c.status));
    if (!areSubCasesClosable) throw new ConflictException('Sub Case is not in closable state');
  } else {
    throw new BadRequestException('Only a Supervisor can close FRAUD_AND_AML Case');
  }
}
```

This is sound as a guard — but the requirement for it is itself the smell. A correctly-modelled coordinator would not need a special gate here.

---

### 7. Reporting triple-count and outcome reconciliation break

The [`report.service.ts`](case-management-system/backend/src/modules/report/report.service.ts) has **no `parent_id IS NULL` or `parent_id IS NOT NULL` predicate anywhere** in its Prisma queries. Confirmed by grep — `parent_id` appears 0 times in `report.service.ts`.

This means:

- **Total case count**: 1 alert → 3 rows counted → **300% inflation** for every FRAUD_AND_AML case
- **Case type breakdown**: all three buckets (`FRAUD`, `AML`, `FRAUD_AND_AML`) are lit simultaneously for a single alert
- **Outcome reconciliation**: closed totals will never match outcome breakdown because `STATUS_84` is in `CLOSED_STATUSES` (L30) but is mapped to `'closed'` in `STATUS_DISTRIBUTION_MAP` (L49) without having a corresponding outcome bucket. The three outcomes `STATUS_81/82/83` map to `closed`, but their denominator is polluted by the container's `STATUS_84` close.
- **`report.controller.ts`** offers `FRAUD_AND_AML` as a `caseType` filter (L203) — this will return the container rows, not just the actual investigations.

The `STATUS_84_COMPLETED` appears in:
- `CLOSED_CASE_STATUSES` ✓
- `STATUS_DISTRIBUTION_MAP` as `'closed'` ✓
- `INACTIVE_CASE_STATUSES` ✓
- **Not in** `REOPENABLE_CASE_STATUSES` — intentional but creates divergence if children reopen

---

## Full inventory of affected files

| File | What's affected |
|---|---|
| [`backend/prisma/schema.prisma`](case-management-system/backend/prisma/schema.prisma) L112 | `parent_id Int?` self-reference on `Case` |
| [`backend/src/modules/bpmn/cms.bpmn20.xml`](case-management-system/backend/src/modules/bpmn/cms.bpmn20.xml) L23, L28, L192–L206 | Exclusive gateway for FRAUD_AND_AML routing (no parallel gateway) |
| [`backend/src/modules/triage/triage.service.ts`](case-management-system/backend/src/modules/triage/triage.service.ts) L372, L582 | Two child case creation calls for FRAUD_AND_AML |
| [`backend/src/modules/case/services/case-creation.service.ts`](case-management-system/backend/src/modules/case/services/case-creation.service.ts) L71–L106 | `createCaseWithInvestigationTask` sets `parentId` |
| [`backend/src/modules/case/services/case-creation-approval.service.ts`](case-management-system/backend/src/modules/case/services/case-creation-approval.service.ts) L84, L190, L477 | FRAUD_AND_AML branching in approval flow |
| [`backend/src/modules/task/task.service.ts`](case-management-system/backend/src/modules/task/task.service.ts) L303–L345 | `promoteParentCaseToInProgress` — status propagation |
| [`backend/src/modules/task/services/task-lifecycle.service.ts`](case-management-system/backend/src/modules/task/services/task-lifecycle.service.ts) L65, L168, L273 | Three separate parent status propagation blocks |
| [`backend/src/modules/case/case.service.ts`](case-management-system/backend/src/modules/case/case.service.ts) L90, L229 | Suspend/resume parent propagation |
| [`backend/src/modules/case/services/case-reopening.service.ts`](case-management-system/backend/src/modules/case/services/case-reopening.service.ts) L68, L244 | Reopen propagates parent back to in-progress |
| [`backend/src/modules/case/services/case-closure-approval.service.ts`](case-management-system/backend/src/modules/case/services/case-closure-approval.service.ts) L112–L145 | Container-only closure guard (supervisor + both children closed) |
| [`backend/src/modules/repository/case.repository.ts`](case-management-system/backend/src/modules/repository/case.repository.ts) L185–L196 | Permission bypass for `FRAUD_AND_AML` case type |
| [`backend/src/modules/report/report.service.ts`](case-management-system/backend/src/modules/report/report.service.ts) L30, L49, L119 | `STATUS_84` in closed/distribution maps; no `parent_id` filter anywhere |
| [`backend/src/modules/report/report.controller.ts`](case-management-system/backend/src/modules/report/report.controller.ts) L203 | `FRAUD_AND_AML` as a caseType filter option |
| [`backend/src/utils/enums/case-enum.ts`](case-management-system/backend/src/utils/enums/case-enum.ts) L5 | `COMPLETE = 'STATUS_84_COMPLETED'` |
| [`backend/src/constants/case.constants.ts`](case-management-system/backend/src/constants/case.constants.ts) L10, L20, L43 | `STATUS_84` in `CLOSURE_OUTCOMES`, `CLOSED_STATUSES`, `INACTIVE_STATUSES` |
| [`frontend/src/features/cases/components/CloseCaseModal.tsx`](case-management-system/frontend/src/features/cases/components/CloseCaseModal.tsx) L100–L107 | Only producer of `STATUS_84_COMPLETED` — hardcoded JS override |
| [`frontend/src/features/cases/components/CaseFilters.tsx`](case-management-system/frontend/src/features/cases/components/CaseFilters.tsx) L68, L119, L138 | `STATUS_84` in filter dropdown and closed-case logic |
| [`frontend/src/features/cases/components/casesTable.utils.ts`](case-management-system/frontend/src/features/cases/components/casesTable.utils.ts) L47, L57, L143 | `STATUS_84` badge colour, `FRAUD_AND_AML` badge, `parentId` mapping |
| [`frontend/src/features/cases/components/ViewCaseModal.tsx`](case-management-system/frontend/src/features/cases/components/ViewCaseModal.tsx) L112, L200 | FRAUD_AND_AML branch to load sub-cases |
| [`frontend/src/features/cases/components/CaseModalsManager.tsx`](case-management-system/frontend/src/features/cases/components/CaseModalsManager.tsx) L565, L686 | FRAUD_AND_AML case type mapping |
| [`frontend/src/features/alerts/types/triage.types.ts`](case-management-system/frontend/src/features/alerts/types/triage.types.ts) L23, L133, L193, L247 | `FRAUD_AND_AML` enum, `parent_id` on Case type |

---

## Side-effect map

```
FRAUD_AND_AML alert
       │
       ├─► Container Case row (parent_id = null, case_type = FRAUD_AND_AML)
       │       ├─ case_owner_user_id = null  → permission bypass required
       │       ├─ STATUS_84_COMPLETED        → non-BPMN status, hardcoded in frontend
       │       ├─ Non-reopenable             → diverges if a child reopens
       │       ├─ Counted in all reports     → triple-count, outcome irreconcilable
       │       └─ Supervisor-only close      → requires both children closed first
       │
       ├─► FRAUD Case row (parent_id = container.case_id)
       │       ├─ Real owner, real lifecycle, STATUS_81/82/83
       │       └─ Every status change propagates to parent (4 services)
       │
       └─► AML Case row (parent_id = container.case_id)
               ├─ Real owner, real lifecycle, STATUS_81/82/83
               └─ Every status change propagates to parent (4 services)
```

---

## Effort Assessment

**🔴 Large — 5–8 days across both repos**

This is not a localised bug. It is a modelling error whose workarounds are load-bearing across the entire case lifecycle. The work falls into two independent tracks:

### Track A — Immediate reporting mitigation (1–2 days, shippable independently)

Add `parent_id: null` filters to all report queries in `report.service.ts`. This stops the triple-count and removes container rows from all totals without touching anything else. This does **not** fix the underlying model, but it stops the data corruption in reports immediately.

Changes:
- `report.service.ts` — add `parent_id: null` to every `prismaService.case.findMany` / `count` where clause
- `report.controller.ts` — optionally remove or comment out `FRAUD_AND_AML` as a filter option (it selects only the container row, not investigations)
- No schema change, no migration, no frontend change

### Track B — Full re-model (5–7 days, must land after the BPMN orchestration is built)

Per the recommended solution in the issue:

1. **BPMN** — replace the `isFraudNAMLGateway` exclusive gateway with a parallel fork/join using two `<callActivity>` elements, each starting a child case process. Block on a parallel join until both children reach terminal states.

2. **Schema** — add an `investigation_group` table (thin, non-case grouping row: `id`, `alert_id`, `tenant_id` — no `status`, no `owner`). Add a Prisma migration. Retain `parent_id` initially as a soft link but stop relying on it for status propagation.

3. **Creation** (`triage.service.ts`, `case-creation.service.ts`, `case-creation-approval.service.ts`) — stop creating the container `FRAUD_AND_AML` `Case` row. Create an `investigation_group` record instead. Create only the two child cases.

4. **Status propagation** — remove the `if (updatedCase.parent_id)` blocks from all four services: `task.service.ts`, `task-lifecycle.service.ts`, `case.service.ts`, `case-reopening.service.ts`. Child case states become fully independent.

5. **Permissions** — remove the `FRAUD_AND_AML` bypass OR-branch from `case.repository.ts` `findCaseWithPermissionCheck`. No container row = nothing to bypass.

6. **Closure** — remove the `isFraudAndAmlCase` branch from `case-closure-approval.service.ts`. Combined SAR/STR closure becomes a coordination task in the Flowable orchestration process.

7. **`STATUS_84_COMPLETED`** — once Track B is live: remove from `CASE_CLOSURE_OUTCOMES`, `CLOSED_CASE_STATUSES`, `INACTIVE_CASE_STATUSES` in `case.constants.ts`; remove the enum value from `case-enum.ts` and `schema.prisma`; write a Prisma migration; remove from `CloseCaseModal.tsx`, `CaseFilters.tsx`, `casesTable.utils.ts`.

8. **Frontend** — update `ViewCaseModal.tsx`, `CaseModalsManager.tsx`, and related components to use `investigation_group` for grouping display rather than `parent_id`.

---

## Acceptance Criteria (from issue)

- [ ] A `FRAUD_AND_AML` alert produces exactly two `Case` rows (FRAUD + AML), each independently owned
- [ ] No `Case` row represents the container; FRAUD/AML pairing tracked via a non-case coordinator
- [ ] `STATUS_84_COMPLETED` is no longer written by any code path and is removed from the enum (with migration)
- [ ] Parent↔child status-propagation blocks removed from all four services
- [ ] Owner/assignee permission check no longer special-cases `FRAUD_AND_AML`
- [ ] Reports count only leaf investigations; a single combined alert is never triple-counted
- [ ] Closed totals reconcile with the outcome breakdown (81/82/83 only)
- [ ] Combined closure / SAR-STR is handled as a coordination step, not a parent-case status

---

## Recommended Sequencing

1. **Ship Track A first** (reporting `parent_id: null` filter) — stops data corruption in reports immediately, zero risk
2. **Build BPMN orchestration** — parallel call-activity process, confirm Flowable join behaviour in test environment
3. **Ship Track B** behind a feature flag or on a dedicated branch — removes all the workaround code
4. **Retire STATUS_84** — only safe after Track B is in production; needs a migration to handle any existing rows

> **Do not retire `STATUS_84` before Track B ships** — it is currently load-bearing for container closure. Any data migration must also handle existing container `Case` rows (set them to `STATUS_83_CLOSED_INCONCLUSIVE` or archive them before the enum is dropped).
