## Complete Fix — Exact Code Changes

This section gives every file change required for both tracks. Track A (reporting fix) can merge independently without touching anything else. Track B is ordered so each step compiles; it must land as a single coordinated release because it removes load-bearing workaround code throughout the lifecycle.

---

### Track A — Fix Reporting Triple-Count (PR 1 of 2)

**Scope:** `report.service.ts` only. Zero schema changes, zero frontend changes.

The fix is a single helper that adds `parent_id: null` to every Prisma `where` clause in the service, ensuring only leaf investigation cases are counted. Container rows (`FRAUD_AND_AML` parent cases) have `parent_id = null` themselves but `case_type = FRAUD_AND_AML` — we need to exclude them by their role as parent, not by type. The reliable signal is: a container has `parent_id = null` AND has children. We can't easily express the "has children" predicate without a subquery, but we *can* filter by `case_type != FRAUD_AND_AML` to exclude containers (containers are the only rows with that type; the two child investigations are typed `FRAUD` and `AML`). This is simpler and correct.

#### `backend/src/modules/report/report.service.ts`

**Change 1 — Add a helper constant at the top of the class (after the existing static constants, ~line 51):**

```ts
// Excludes FRAUD_AND_AML container rows from all counts.
// Container rows have case_type = FRAUD_AND_AML; their child investigations
// have case_type = FRAUD or AML and are the actual countable work items.
private static readonly LEAF_CASE_FILTER = {
  NOT: { case_type: CaseType.FRAUD_AND_AML },
} as const;
```

**Change 2 — Apply `LEAF_CASE_FILTER` to every `prisma.case` query in `buildCommonCaseFilters`:**

```ts
// BEFORE
private buildCommonCaseFilters(filters?: {
  caseType?: string;
  priority?: string;
  investigator?: string;
  tenantId?: string;
}): Record<string, any> {
  const where: Record<string, any> = {};
  if (filters?.caseType) where.case_type = filters.caseType;
  if (filters?.priority) where.priority = filters.priority;
  if (filters?.investigator) where.case_owner_user_id = filters.investigator;
  if (filters?.tenantId) where.tenant_id = filters.tenantId;
  return where;
}

// AFTER
private buildCommonCaseFilters(filters?: {
  caseType?: string;
  priority?: string;
  investigator?: string;
  tenantId?: string;
}): Record<string, any> {
  const where: Record<string, any> = { ...ReportsService.LEAF_CASE_FILTER };
  if (filters?.caseType) where.case_type = filters.caseType;
  if (filters?.priority) where.priority = filters.priority;
  if (filters?.investigator) where.case_owner_user_id = filters.investigator;
  if (filters?.tenantId) where.tenant_id = filters.tenantId;
  return where;
}
```

Because every report query calls `buildCommonCaseFilters` (directly or through `applyInvestigatorScope`), this one change propagates to all counts, groupBys, findManys, and trend queries throughout the file.

**Change 3 — Patch the two direct queries that do not go through `buildCommonCaseFilters`:**

Grep confirms two additional direct queries in `getInvestigatorWorkload` (~line 559) and `computeStatusDetails` (~line 315). Apply the filter to each:

```ts
// getInvestigatorWorkload — line 559 area
// BEFORE
where: {
  created_at: { gte: startDate, lte: endDate },
  case_owner_user_id: { not: null },
  tenant_id: tenantId,
},

// AFTER
where: {
  ...ReportsService.LEAF_CASE_FILTER,
  created_at: { gte: startDate, lte: endDate },
  case_owner_user_id: { not: null },
  tenant_id: tenantId,
},
```

```ts
// computeStatusDetails — line 315 area — inside the map callback
// BEFORE
where: { ...whereClause, status },

// AFTER
where: { ...ReportsService.LEAF_CASE_FILTER, ...whereClause, status },
```

**Change 4 — `STATUS_84_COMPLETED` in `STATUS_DISTRIBUTION_MAP` (line 49):**

`STATUS_84` is a container-only status. After applying the leaf filter it will never appear in any query result. Remove it from the distribution map to make the contract explicit:

```ts
// Remove this line from STATUS_DISTRIBUTION_MAP:
[CaseStatus.STATUS_84_COMPLETED]: 'closed',
```

Also remove from `CLOSED_STATUSES` (line 30) — the leaf filter makes it unreachable, and its presence here is what caused it to pollute closed totals:

```ts
// BEFORE
private static readonly CLOSED_STATUSES: CaseStatus[] = [
  CaseStatus.STATUS_71_AUTOCLOSED_CONFIRMED,
  CaseStatus.STATUS_72_AUTOCLOSED_REFUTED,
  CaseStatus.STATUS_81_CLOSED_REFUTED,
  CaseStatus.STATUS_82_CLOSED_CONFIRMED,
  CaseStatus.STATUS_83_CLOSED_INCONCLUSIVE,
  CaseStatus.STATUS_84_COMPLETED,   // ← remove
];

// AFTER
private static readonly CLOSED_STATUSES: CaseStatus[] = [
  CaseStatus.STATUS_71_AUTOCLOSED_CONFIRMED,
  CaseStatus.STATUS_72_AUTOCLOSED_REFUTED,
  CaseStatus.STATUS_81_CLOSED_REFUTED,
  CaseStatus.STATUS_82_CLOSED_CONFIRMED,
  CaseStatus.STATUS_83_CLOSED_INCONCLUSIVE,
];
```

Note: `STATUS_84` is still in `case.constants.ts` `CLOSED_CASE_STATUSES` for the lifecycle code — do not change that file in Track A. Only the report service's private static list changes here.

**Track A impact:**
- `totalCases` no longer triple-counts FRAUD_AND_AML alerts
- `caseTypes` breakdown no longer shows a `FRAUD_AND_AML` bucket inflating totals
- `closedCases` no longer includes container `STATUS_84` closes
- `avgResolutionTime` no longer distorted by container rows that close near-instantly
- Investigator workload report no longer attributes the container case to the case creator

---

### Track B — Full Re-Model (PR 2 of 2)

Track B removes every workaround layer. It must ship as a single coordinated release; individual steps do not compile until all are in place.

---

#### Step B1 — Schema: Add `investigation_group` Table, Keep `parent_id` Temporarily

**File: `backend/prisma/schema.prisma`**

Add the new lightweight grouping table after the `ReferenceId` model:

```prisma
model InvestigationGroup {
  id         Int      @id @default(autoincrement())
  alert_id   Int      @unique
  tenant_id  String
  created_at DateTime @default(now()) @db.Timestamp(6)

  @@map("investigation_groups")
}
```

The `Case` model's `parent_id Int?` field (line 112) is **kept in place** for Track B — it is still needed to identify existing container rows during the migration. It will be removed in a follow-on cleanup migration once all containers are retired from production data.

No change to `CaseType` enum — `FRAUD_AND_AML` stays as a valid alert type (alerts still arrive as that type); it just no longer produces a `Case` row of that type.

Migration command:
```bash
cd backend
npx prisma migrate dev --name "add_investigation_group"
```

---

#### Step B2 — Stop Creating the Container Case (`triage.service.ts`)

Two code paths create child cases — manual triage (~L372) and auto-triage (~L582). In both, the container case already exists by the time the `FRAUD_AND_AML` branch runs. The fix: create an `InvestigationGroup` record instead of relying on the container case, and keep creating the two child cases (no change to child creation).

**File: `backend/src/modules/triage/triage.service.ts`**

Add `InvestigationGroupRepository` injection (or inline Prisma call — shown inline for minimal blast radius):

```ts
// Manual triage path — L372 area
// BEFORE
if (alert.alert_type === CaseType.FRAUD_AND_AML) {
  await this.caseCreateService.createCaseWithInvestigationTask(
    CaseType.FRAUD, userId, tenantId, alert.case_id, priority, ...
  );
  await this.caseCreateService.createCaseWithInvestigationTask(
    CaseType.AML, userId, tenantId, alert.case_id, priority, ...
  );
}

// AFTER
if (alert.alert_type === CaseType.FRAUD_AND_AML) {
  // Record the grouping — replaces the container Case row
  await this.prisma.investigationGroup.upsert({
    where: { alert_id: alert.alert_id },
    update: {},
    create: { alert_id: alert.alert_id, tenant_id: tenantId },
  });
  await this.caseCreateService.createCaseWithInvestigationTask(
    CaseType.FRAUD, userId, tenantId, null, priority,   // parentCaseId = null
    CaseCreationType.AUTOMATIC_SYSTEM, 'SUPERVISOR',
  );
  await this.caseCreateService.createCaseWithInvestigationTask(
    CaseType.AML, userId, tenantId, null, priority,     // parentCaseId = null
    CaseCreationType.AUTOMATIC_SYSTEM, 'SUPERVISOR',
  );
}
```

Apply the same `null` parentCaseId change to the auto-triage path at ~L612.

**File: `backend/src/modules/case/services/case-creation.service.ts`**

Update `createCaseWithInvestigationTask` signature to accept `null` for `parentCaseId`:

```ts
// BEFORE (line 71)
async createCaseWithInvestigationTask(
  alertType: CaseType,
  userId: string,
  tenantId: string,
  parentCaseId: number,
  ...
): Promise<unknown>

// AFTER
async createCaseWithInvestigationTask(
  alertType: CaseType,
  userId: string,
  tenantId: string,
  parentCaseId: number | null,   // null = standalone, no container parent
  ...
): Promise<unknown>
```

Inside the function, the `parentId: parentCaseId` line already passes the value through to the Prisma create call — no other change needed. With `null` passed in, `parent_id` will be `null` on both child rows, which is correct — they are now independent cases.

---

#### Step B3 — Remove Parent Status Propagation (Four Services)

Each propagation block follows the same pattern: `if (updatedCase.parent_id) { ... }`. Delete the entire `if` block in each location. Leave all surrounding code intact.

**`backend/src/modules/task/task.service.ts` — `promoteParentCaseToInProgress` (L303–L353):**

```ts
// BEFORE — line 303
if (updatedCase.parent_id) {
  await this.promoteParentCaseToInProgress(updatedCase.parent_id, updatedCase, tx);
}

// AFTER — delete those 3 lines

// Also delete the entire private `promoteParentCaseToInProgress` method (L318–L353)
// — it has no remaining callers.
```

**`backend/src/modules/task/services/task-lifecycle.service.ts` — three blocks:**

Block 1 (assignTask — L65–L81):
```ts
// BEFORE
if (updatedCase.parent_id) {
  const subCase = await tx.case.findFirst({ ... });
  if (updatedCase.status === CaseStatus.STATUS_10_ASSIGNED && subCase?.status === CaseStatus.STATUS_10_ASSIGNED) {
    await tx.case.update({ where: { case_id: updatedCase.parent_id }, data: { status: CaseStatus.STATUS_10_ASSIGNED, updated_at: new Date() } });
  }
}

// AFTER — delete the entire if block (lines 65–81)
```

Block 2 (reassignTask — L168–L184): same structure, same deletion.

Block 3 (unassignTask — L273–L292): same structure, same deletion.

**`backend/src/modules/case/case.service.ts` — suspend (L90–L107) and resume (L229–L246):**

```ts
// BEFORE — suspend, lines 90–107
if (updatedCase.parent_id) {
  const subCase = await prisma.case.findFirst({ where: { parent_id: updatedCase.parent_id, ... } });
  if (updatedCase.status === CaseStatus.STATUS_21_SUSPENDED && subCase?.status === CaseStatus.STATUS_21_SUSPENDED) {
    await prisma.case.update({ where: { case_id: updatedCase.parent_id }, data: { status: CaseStatus.STATUS_21_SUSPENDED, ... } });
  }
}

// AFTER — delete the entire if block

// BEFORE — resume, lines 229–246 (same pattern)
if (updatedCase.parent_id) { ... }

// AFTER — delete the entire if block
```

**`backend/src/modules/case/services/case-reopening.service.ts` — two blocks:**

Block 1 (supervisorReopenCase — L68–L73):
```ts
// BEFORE
if (updatedCase.parent_id) {
  await tx.case.update({
    where: { case_id: updatedCase.parent_id },
    data: { status: CaseStatus.STATUS_20_IN_PROGRESS, updated_at: new Date() },
  });
}

// AFTER — delete the entire if block
```

Block 2 (~L244–L249): same structure, same deletion.

---

#### Step B4 — Remove the Permission Bypass (`case.repository.ts`)

**File: `backend/src/modules/repository/case.repository.ts` — `findCaseWithPermissionCheck` (L185–L215)**

```ts
// BEFORE
if (isValidUuid) {
  whereCondition.OR = [
    {
      case_type: { in: ['FRAUD_AND_AML'] },   // bypass — any in-tenant user
    },
    {
      AND: [
        { case_type: { notIn: ['FRAUD_AND_AML'] } },
        {
          OR: [
            { case_owner_user_id: userId },
            { tasks: { some: { assigned_user_id: userId, name: { in: [...] } } } },
          ],
        },
      ],
    },
  ];
}

// AFTER — simplified: no FRAUD_AND_AML exception needed
if (isValidUuid) {
  whereCondition.OR = [
    { case_owner_user_id: userId },
    { tasks: { some: { assigned_user_id: userId, name: { in: ['Investigate Case', 'Investigate case', 'investigate case'] } } } },
  ];
}
```

No container row = no need for the bypass. Child FRAUD and AML cases get normal owner/assignee permission checks.

---

#### Step B5 — Remove the FRAUD_AND_AML Closure Gate (`case-closure-approval.service.ts`)

**File: `backend/src/modules/case/services/case-closure-approval.service.ts` (L112–L146)**

```ts
// BEFORE
const isFraudAndAmlCase = caseData.case_type === 'FRAUD_AND_AML';
let primaryTask: Task | null = null;

if (isFraudAndAmlCase) {
  if (role === 'CMS_SUPERVISOR') {
    const subCase = await this.prismaService.case.findMany({ where: { parent_id: caseId, ... } });
    if (subCase.length > 0) {
      const areSubCasesClosable = subCase.every((c) => CLOSED_CASE_STATUSES.includes(c.status));
      if (!areSubCasesClosable) throw new ConflictException({ message: '...' });
    } else {
      throw new BadRequestException({ message: 'Sub Cases does not exist...' });
    }
  } else {
    throw new BadRequestException({ message: 'Only a Supervisor can close FRAUD_AND_AML Case' });
  }
} else {
  // Single investigation case
  const investigationTask = caseData.tasks.filter(...);
  ...
}

// AFTER — delete the isFraudAndAmlCase block entirely; the else branch becomes the only path
const isFraudAndAmlCase = caseData.case_type === 'FRAUD_AND_AML';
let primaryTask: Task | null = null;

// Single investigation case (all cases now follow this path)
const investigationTask = caseData.tasks.filter(
  (task) =>
    task.name === TASK_NAMES.INVESTIGATE_CASE &&
    (task.status === TaskStatus.STATUS_20_IN_PROGRESS || task.status === TaskStatus.STATUS_30_COMPLETED),
);
// ... rest of the existing single-case path
```

Also remove the `isFraudAndAmlCase` variable declaration — it's no longer used.

---

#### Step B6 — Update the BPMN (`cms.bpmn20.xml`)

The `isFraudNAMLGateway` exclusive gateway currently routes `isFraudNAML == true` to the `investigateCase` user task (correct — child cases go directly to investigation). The gateway itself is fine; it is the **container case's** entry into the BPMN that is being removed by not creating it anymore.

Child cases created in Step B2 call `executeFlowableCaseCreationEvent` with `isFraudNAML = false` (they are `FRAUD` and `AML` typed, not `FRAUD_AND_AML`). They follow the normal `flowNotFraudNAML → completeCaseCreation` path.

**No BPMN change is required for the container-removal fix.** The gateway is no longer reached for any `FRAUD_AND_AML`-typed case instance because no such `Case` row is created to start a process for. The `isFraudNAML` process variable and the gateway can be **cleaned up** in a follow-on PR but do not block Track B.

If a true Flowable parallel join is later wanted (to coordinate combined closure), that is a separate feature addition — it does not block removing the broken container model.

---

#### Step B7 — Retire `STATUS_84_COMPLETED`

This step is only safe after Track B is in production and all existing container rows have been migrated (Step B8). Do not merge before that.

**`backend/src/constants/case.constants.ts`:**

```ts
// BEFORE
export const CASE_CLOSURE_OUTCOMES = [
  'STATUS_81_CLOSED_REFUTED',
  'STATUS_82_CLOSED_CONFIRMED',
  'STATUS_83_CLOSED_INCONCLUSIVE',
  'STATUS_84_COMPLETED',   // ← remove
] as const;

export const CLOSED_CASE_STATUSES: CaseStatus[] = [
  CaseStatus.STATUS_81_CLOSED_REFUTED,
  CaseStatus.STATUS_82_CLOSED_CONFIRMED,
  CaseStatus.STATUS_83_CLOSED_INCONCLUSIVE,
  CaseStatus.STATUS_84_COMPLETED,   // ← remove
  CaseStatus.STATUS_71_AUTOCLOSED_CONFIRMED,
  CaseStatus.STATUS_72_AUTOCLOSED_REFUTED,
];

export const INACTIVE_CASE_STATUSES: CaseStatus[] = [
  CaseStatus.STATUS_81_CLOSED_REFUTED,
  CaseStatus.STATUS_82_CLOSED_CONFIRMED,
  CaseStatus.STATUS_83_CLOSED_INCONCLUSIVE,
  CaseStatus.STATUS_84_COMPLETED,   // ← remove
  CaseStatus.STATUS_99_ABANDONED,
];
```

**`backend/src/utils/enums/case-enum.ts`:**

```ts
// BEFORE
export enum CaseClosureOutcome {
  CLOSED_REFUTED    = 'STATUS_81_CLOSED_REFUTED',
  CLOSED_CONFIRMED  = 'STATUS_82_CLOSED_CONFIRMED',
  CLOSED_INCONCLUSIVE = 'STATUS_83_CLOSED_INCONCLUSIVE',
  COMPLETE          = 'STATUS_84_COMPLETED',   // ← remove
}

// AFTER
export enum CaseClosureOutcome {
  CLOSED_REFUTED      = 'STATUS_81_CLOSED_REFUTED',
  CLOSED_CONFIRMED    = 'STATUS_82_CLOSED_CONFIRMED',
  CLOSED_INCONCLUSIVE = 'STATUS_83_CLOSED_INCONCLUSIVE',
}
```

**`backend/prisma/schema.prisma`:**

Remove `STATUS_84_COMPLETED` from the `CaseStatus` enum after the data migration (Step B8) confirms zero rows with that value.

**`backend/src/modules/report/report.service.ts`:**

Track A already removed `STATUS_84` from this file's private constants — nothing left to do here.

---

#### Step B8 — Data Migration

Two-phase SQL migration to handle existing container rows:

```sql
-- Phase 1: For each FRAUD_AND_AML container case (parent_id IS NULL, case_type = FRAUD_AND_AML),
-- create a corresponding investigation_group row linking the alert.
INSERT INTO investigation_groups (alert_id, tenant_id, created_at)
SELECT a.alert_id, c.tenant_id, NOW()
FROM cases c
JOIN alerts a ON a.case_id = c.case_id
WHERE c.case_type = 'FRAUD_AND_AML'
  AND c.parent_id IS NULL
ON CONFLICT (alert_id) DO NOTHING;

-- Phase 2: Re-point the two child cases' alert association.
-- Children currently have parent_id set to the container case_id.
-- After migration children are standalone; update their alert link.
-- (Only needed if alert.case_id points to the container — check schema)
-- The alert already points to the container via alert.case_id; update it
-- to point to the FRAUD child (or leave NULL and use investigation_group).
-- This depends on how the frontend resolves alert->case linkage.
-- Safest approach: set alert.case_id = NULL for FRAUD_AND_AML alerts
-- and resolve via investigation_groups going forward.
UPDATE alerts
SET case_id = NULL
WHERE alert_id IN (
  SELECT alert_id FROM investigation_groups
);

-- Phase 3: Reclassify container Case rows as closed-inconclusive
-- so they can be dropped from the enum without orphaned data.
UPDATE cases
SET status = 'STATUS_83_CLOSED_INCONCLUSIVE'
WHERE case_type = 'FRAUD_AND_AML'
  AND parent_id IS NULL;

-- Phase 4: Verify zero remaining STATUS_84 rows
-- before dropping enum value (run as validation, not as migration step)
SELECT COUNT(*) FROM cases WHERE status = 'STATUS_84_COMPLETED';  -- must be 0
```

After Phase 3 passes validation, run the Prisma migration that drops `STATUS_84_COMPLETED` from the `CaseStatus` enum.

---

#### Step B9 — Frontend Changes

**`frontend/src/features/cases/components/CloseCaseModal.tsx` (L99–L108) — remove hardcoded STATUS_84 override:**

```tsx
// BEFORE
useEffect(() => {
  if (caseData?.type === 'FRAUD_AND_AML') {
    const combined = 'STATUS_84_COMPLETED';
    setFormData((prev) => ({
      ...prev,
      recommendedOutcome: combined as CloseCaseDto['recommendedOutcome'],
    }));
  }
}, [caseData?.type, subCasesDetails]);

// AFTER — delete the entire useEffect block.
// FRAUD_AND_AML container cases no longer exist; this path is unreachable.
```

Also remove the sub-cases display block (L155–L205) — the `caseData?.type === 'FRAUD_AND_AML'` branch that renders the Sub-Cases Closure Status table. It has no data to show and no container to close.

The "Final Outcome" locked-to-COMPLETED section (L208–L221) — delete it. The `caseData?.type === 'FRAUD_AND_AML'` ternary collapses to the standard outcome select for all cases.

The "SUPERVISOR → Close Case (AFTER report)" button (L348–L357) that is gated on `caseData?.type === 'FRAUD_AND_AML'` — delete it. No container cases remain; the standard Generate Investigation Report → Close flow applies to all cases.

**`frontend/src/features/cases/components/CaseFilters.tsx` (L68):**

```tsx
// BEFORE
{ value: 'STATUS_84_COMPLETED', label: 'Completed' },

// AFTER — remove this line from statusOptions
```

**`frontend/src/features/cases/components/casesTable.utils.ts` (L47):**

```ts
// BEFORE
STATUS_84_COMPLETED: 'bg-green-50 text-green-700',

// AFTER — remove this key from statusColors in getStatusColor
```

Keep `FRAUD_AND_AML` in `getTypeColor` (L57) — alerts are still typed `FRAUD_AND_AML`; only the Case row is gone. If the case list should not show a `FRAUD_AND_AML` type badge at all (because those cases no longer exist), remove it from `getTypeColor` as well in a cleanup pass.

**`frontend/src/features/alerts/types/triage.types.ts` (L133, L247):**

The `parent_id?: number` field on the Case type interface — remove it. The `FRAUD_AND_AML` value in `CaseStatus` const object — check whether it's in the CaseStatus const or CaseType const. `FRAUD_AND_AML` stays in the `AlertType`/`CaseType` const because alerts still carry that type. The `parent_id` field on the Case interface can be removed once all existing `parent_id` references in the UI are removed.

---

## Test Cases

### Unit Tests (new — add to test suite)

#### Reporting: no triple-count

```ts
describe('ReportsService — LEAF_CASE_FILTER', () => {
  it('excludes FRAUD_AND_AML container rows from total case count', async () => {
    // Seed: 1 FRAUD_AND_AML container (parent_id=null, case_type=FRAUD_AND_AML)
    //        1 FRAUD child (parent_id=container.case_id, case_type=FRAUD)
    //        1 AML child (parent_id=container.case_id, case_type=AML)
    // Expect: totalCases = 2 (FRAUD + AML only)
    const result = await service.getCaseStatus('30d', { tenantId: 'T1' });
    expect(result.stats.totalCases).toBe(2);
  });

  it('caseTypes breakdown does not include FRAUD_AND_AML bucket', async () => {
    const result = await service.getCaseStatus('30d', { tenantId: 'T1' });
    const types = result.caseTypes.map((t) => t.name);
    expect(types).not.toContain('FRAUD_AND_AML');
  });

  it('closed totals do not include STATUS_84_COMPLETED container close', async () => {
    // Seed: container in STATUS_84_COMPLETED, children in STATUS_82_CLOSED_CONFIRMED
    const result = await service.getCaseStatus('30d', { tenantId: 'T1' });
    // closedCases = 2 (FRAUD + AML), not 3
    expect(result.stats.closedCases).toBe(2);
  });
});
```

#### No container Case row created on FRAUD_AND_AML triage

```ts
describe('TriageService — FRAUD_AND_AML', () => {
  it('creates exactly two Case rows (FRAUD + AML) for a FRAUD_AND_AML alert', async () => {
    const before = await prisma.case.count();
    await triageService.handleManualTriage(alertId, dto, userId, tenantId);
    const after = await prisma.case.count();
    expect(after - before).toBe(2);   // not 3

    const cases = await prisma.case.findMany({ where: { alert: { alert_id: alertId } } });
    expect(cases.map((c) => c.case_type).sort()).toEqual(['AML', 'FRAUD']);
    expect(cases.every((c) => c.parent_id === null)).toBe(true);
  });

  it('creates an InvestigationGroup record linking the alert', async () => {
    await triageService.handleManualTriage(alertId, dto, userId, tenantId);
    const group = await prisma.investigationGroup.findUnique({ where: { alert_id: alertId } });
    expect(group).toBeTruthy();
    expect(group!.tenant_id).toBe(tenantId);
  });
});
```

#### Status propagation no longer fires

```ts
describe('TaskLifecycleService — no parent propagation', () => {
  it('assigning a FRAUD child task does not update any parent case', async () => {
    // Seed FRAUD case with parent_id = null (post-Track B: no container)
    await taskService.assignTask(taskId, investigatorId, userId, tenantId, ...);
    const fraudCase = await prisma.case.findUnique({ where: { case_id: fraudCaseId } });
    expect(fraudCase!.status).toBe(CaseStatus.STATUS_10_ASSIGNED);

    // Assert no other case was updated (no container exists to propagate to)
    const otherCases = await prisma.case.findMany({ where: { case_id: { not: fraudCaseId }, tenant_id: tenantId } });
    // AML sibling is untouched
    expect(otherCases.every((c) => c.status !== CaseStatus.STATUS_10_ASSIGNED)).toBe(true);
  });
});
```

#### Permission check: no FRAUD_AND_AML bypass

```ts
describe('CaseRepository — findCaseWithPermissionCheck', () => {
  it('does not allow arbitrary in-tenant user to access FRAUD or AML case they do not own', async () => {
    // Seed FRAUD case owned by investigator A
    await expect(
      caseRepo.findCaseWithPermissionCheck(fraudCaseId, tenantId, investigatorBId)
    ).rejects.toThrow();
  });

  it('allows assigned investigator to access their FRAUD case', async () => {
    const result = await caseRepo.findCaseWithPermissionCheck(fraudCaseId, tenantId, investigatorAId);
    expect(result.case_id).toBe(fraudCaseId);
  });
});
```

#### `STATUS_84` no longer produced

```ts
describe('CloseCaseModal — no STATUS_84 output', () => {
  it('FRAUD case closure uses STATUS_81/82/83 outcomes only', async () => {
    // Render CloseCaseModal with FRAUD caseData
    // Check the recommendedOutcome select options
    // None should be STATUS_84_COMPLETED
    const options = screen.getAllByRole('option').map((o) => o.getAttribute('value'));
    expect(options).not.toContain('STATUS_84_COMPLETED');
  });
});
```

### Integration / E2E Tests

#### Full FRAUD_AND_AML alert lifecycle (post-Track B)

```
Scenario: triage a FRAUD_AND_AML alert end-to-end

1. Ingest alert with alert_type = FRAUD_AND_AML
2. Triage alert as FRAUD_AND_AML
3. Assert: exactly 2 Case rows in DB (types FRAUD and AML), both parent_id = null
4. Assert: 1 InvestigationGroup row with alert_id
5. Assert: 0 Case rows with case_type = FRAUD_AND_AML

6. Assign and work both cases independently
7. Close FRAUD case with STATUS_82_CLOSED_CONFIRMED
8. Close AML case with STATUS_83_CLOSED_INCONCLUSIVE
9. Assert: no parent case update occurs on either closure

10. GET /reports — assert totalCases = 2, caseTypes has FRAUD (1) and AML (1)
```

#### Report outcome reconciliation

```
Scenario: closed totals equal outcome breakdown totals

1. Create 3 alerts: 1 FRAUD_AND_AML, 1 FRAUD, 1 AML
2. Close all cases with outcome STATUS_82
3. GET /reports
4. Assert: closedCases = 4 (2 from FRAUD_AND_AML, 1 FRAUD, 1 AML)
5. Assert: outcomes.confirmed = 4
6. Assert: closedCases == outcomes.confirmed + outcomes.resolved + outcomes.inconclusive
   (reconciliation identity holds)
```

#### Permission: FRAUD child case has normal access control

```
Scenario: FRAUD child case follows standard permission model

1. Create FRAUD case with investigator A as owner
2. Investigator B (different user, same tenant) attempts to access case
3. Assert: 403 Forbidden
4. Investigator A accesses case
5. Assert: 200 OK
```

### Data Migration Validation

```sql
-- After Phase 1: every FRAUD_AND_AML alert has an investigation_group row
SELECT COUNT(*) FROM alerts a
LEFT JOIN investigation_groups ig ON ig.alert_id = a.alert_id
WHERE a.alert_type = 'FRAUD_AND_AML' AND ig.id IS NULL;
-- must be 0

-- After Phase 3: zero STATUS_84 rows remain
SELECT COUNT(*) FROM cases WHERE status = 'STATUS_84_COMPLETED';
-- must be 0

-- After Track B ships: zero FRAUD_AND_AML Case rows remain
SELECT COUNT(*) FROM cases WHERE case_type = 'FRAUD_AND_AML';
-- must be 0

-- Verify child cases now have parent_id = null
SELECT COUNT(*) FROM cases WHERE parent_id IS NOT NULL;
-- must be 0 (all existing children were linked to now-retired containers)
```

### Manual / UAT Checks

| # | Scenario | Steps | Expected Result |
|---|---|---|---|
| M1 | Triple-count fix (Track A) | Create 1 FRAUD_AND_AML alert; triage it. Open dashboard. | Total cases = 2, not 3 |
| M2 | Case type breakdown (Track A) | Open reports caseTypes chart. | Shows FRAUD (1) and AML (1), no FRAUD_AND_AML bucket |
| M3 | Closed totals reconcile (Track A) | Close both child cases. Check closed count vs outcome breakdown. | Numbers match |
| M4 | No container in case list (Track B) | Open case list; filter by type FRAUD_AND_AML. | Zero results returned |
| M5 | FRAUD and AML cases are independent | Assign FRAUD case. Check AML case status. | AML case unchanged |
| M6 | Close FRAUD case independently | Close FRAUD case with any outcome. | AML case is unaffected; no parent case updated |
| M7 | No STATUS_84 in close modal | Open close modal for FRAUD case. | Outcome dropdown shows 81/82/83 only |
| M8 | Permission: investigator B cannot see A's case | Log in as investigator B; attempt to view case owned by A. | 403 or case absent from list |
| M9 | Completed status absent from filter | Open case filter status dropdown. | "Completed (STATUS_84)" option absent |
| M10 | Investigation group created | Triage FRAUD_AND_AML alert; query investigation_groups table. | 1 row with correct alert_id and tenant_id |

---

## Overall Impact of the Fix

| Area | Before | After |
|---|---|---|
| Case rows per FRAUD_AND_AML alert | 3 (container + FRAUD + AML) | 2 (FRAUD + AML only) |
| Total case count in reports | Triple-counted | Accurate |
| Case type breakdown | FRAUD_AND_AML bucket inflates totals | FRAUD and AML buckets only |
| Closed totals | Never reconcile (STATUS_84 outside outcome set) | Reconcile exactly |
| Status propagation code | Spread across 4 services, 7 locations | Removed entirely |
| Permission model | FRAUD_AND_AML container bypasses owner/assignee checks | Standard permission checks apply to all cases |
| STATUS_84_COMPLETED | Load-bearing synthetic status, hardcoded in frontend | Retired; only STATUS_81/82/83 are valid outcomes |
| FRAUD/AML case independence | AML case status changes when FRAUD is assigned/suspended/reopened | Each case is fully independent |
| Combined closure guard | Supervisor-only, requires both children closed, blocks normal close path | Removed; each case closes independently |
| DB grouping of FRAUD+AML pair | Via Case.parent_id self-reference | Via InvestigationGroup table (alert-keyed, no status/owner) |
| BPMN | Exclusive gateway routes FRAUD_AND_AML to same user task via app code | No BPMN change required; child cases follow normal path |

---

## Fix Summary

The fix for issue #214 corrects a domain modelling error that has been generating compounding side effects across the entire case lifecycle since it was introduced. When a `FRAUD_AND_AML` alert arrives, the system creates three `Case` rows: a container typed `FRAUD_AND_AML` and two child investigations typed `FRAUD` and `AML`. The container has no real owner, no real verdict, and no real lifecycle — it exists solely as a grouping mechanism built on top of the `Case` entity. Because the domain model does not have a dedicated grouping concept, the codebase has accumulated five layers of bespoke workaround code to make this arrangement function: a synthetic terminal status (`STATUS_84_COMPLETED`) invented outside the BPMN, a permission bypass that lets any in-tenant user access the ownerless container, hand-coded parent-to-child status propagation duplicated across four separate backend services, a special-cased closure gate that blocks the container from closing until both children are done, and a frontend override that hardcodes `STATUS_84` as the close outcome whenever the case type is `FRAUD_AND_AML`. On top of all of this, the report service has no awareness of container rows and triple-counts every `FRAUD_AND_AML` alert — one count for each of the three `Case` rows — making all case totals, type breakdowns, and outcome reconciliation permanently incorrect.

The fix addresses this in two tracks with a clear sequencing dependency between them.

Track A ships first and is a single-file change to `report.service.ts`. A new static constant `LEAF_CASE_FILTER = { NOT: { case_type: CaseType.FRAUD_AND_AML } }` is spread into the `buildCommonCaseFilters` helper that every report query calls. Because all counts, groupBys, trend queries, and workload queries flow through that helper, the one-line addition propagates to every report in the system. Two direct queries that bypass the helper are patched individually. `STATUS_84_COMPLETED` is removed from the service's private closed-status list and distribution map. The result: total case counts become accurate, the case-type breakdown stops showing a `FRAUD_AND_AML` bucket, closed totals reconcile with outcome breakdowns, and the investigator workload report stops attributing the ownerless container to the case creator. Track A requires no schema migration, no frontend changes, and no changes to any lifecycle code. It can be reviewed, merged, and deployed in a matter of hours.

Track B removes the underlying model. It ships as a coordinated release after Track A is in production. A new `InvestigationGroup` table is added to the schema — a lightweight, intentionally thin record with only `id`, `alert_id`, and `tenant_id`, and no status, no owner, and no lifecycle. This is the correct abstraction for what the container case was trying to be: a named grouping of two independent investigations linked to a single alert. The triage service stops creating the container `Case` row and instead writes an `InvestigationGroup` record; the two child cases are created with `parent_id = null`, making them fully independent entities. With no container row to propagate to, all seven `if (updatedCase.parent_id)` blocks scattered across `task.service.ts`, `task-lifecycle.service.ts`, `case.service.ts`, and `case-reopening.service.ts` are deleted outright. The `FRAUD_AND_AML` OR-branch in the permission check — the bypass that allowed any in-tenant user to see the ownerless container — is removed; FRAUD and AML child cases go through the standard owner and assignee checks like every other case. The special closure gate in `case-closure-approval.service.ts` that required supervisor role and both children to be closed before the container could close is also removed; each case closes independently through the normal path.

Once Track B is in production and a data migration has backfilled `InvestigationGroup` rows for all existing alerts and reclassified all existing container `Case` rows to a terminal status, `STATUS_84_COMPLETED` is retired from `case.constants.ts`, `case-enum.ts`, the Prisma schema, and all frontend components. The `CloseCaseModal` loses its hardcoded override, its sub-case display block, and its container-specific close button. The status filter dropdown loses the "Completed" option. The badge colour map loses the `STATUS_84` entry.

The BPMN requires no change. Child cases enter the workflow as `FRAUD` or `AML` typed instances and follow the standard investigation path. The `isFraudNAMLGateway` exclusive gateway and the `isFraudNAML` process variable can be cleaned up in a follow-on commit but do not block the release.

After both tracks land, the codebase is simpler, the data is correct, and every case — regardless of alert type — is owned, independently closeable, and counted exactly once in every report.
