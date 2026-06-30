## Complete Fix — Exact Code Changes

This section details every file change required to implement the full settled design. Changes are ordered so each step compiles independently — Track A (stop the bleed) can be merged on its own; Track B layers on top.

---

### Track A — Stop the Cron Overwrite (PR 1 of 2)

One file, one deletion. Shippable in isolation; safe to merge before any enum rename.

#### `backend/src/modules/alert-priority/alert-priority.service.ts`

Remove the `case.update` block (lines 65–72). The alert itself still gets its `priority` and `priority_score` updated (alert-level display is still useful); only the case write is removed.

**Before:**
```ts
// Update associated case priority if it exists
if (alert.case_id) {
  await this.prisma.case.update({
    where: { case_id: alert.case_id },
    data: {
      priority,
    },
  });
}
```

**After:** *(delete those 9 lines entirely)*

No other changes in this file for Track A. The `urgencyThresholds` and `defaultSlaHours` fields remain — they are still needed for the alert-level update and will be repurposed in Track B.

**Impact of Track A alone:**
- `case.priority` is now written exactly once: at triage (via `case-creation.service.ts` / `triage.service.ts`). It is never overwritten again.
- `alert.priority` continues to reflect SLA age — acceptable as an alert-level urgency indicator.
- `sla_state` is still invisible. SLA notifications still fire against `Task.sla_deadline` (task-level; separate from the case `priority` field). Nothing breaks.

---

### Track B — Full Enum Rename and SLA Model (PR 2 of 2)

Track B assumes Track A is already merged. Every change below is against the post–Track A codebase.

---

#### Step B1 — Prisma Schema (`backend/prisma/schema.prisma`)

Three changes: rename the `Priority` enum, add `sla_due_at` to `Case`, and add the `SlaPolicy` tenant model.

**Change 1 — Rename `Priority` enum (line 241–246):**

```prisma
// BEFORE
enum Priority {
  NEW
  URGENT
  CRITICAL
  BREACH
}

// AFTER
enum Priority {
  LOW
  MEDIUM
  HIGH
}
```

**Change 2 — Add `sla_due_at` to `Case` model (after line 110, inside the `Case` model block):**

```prisma
model Case {
  case_id              Int                  @id @default(autoincrement())
  case_creator_user_id String               @db.Uuid
  case_owner_user_id   String?              @db.Uuid
  tenant_id            String
  status               CaseStatus
  priority             Priority             @default(MEDIUM)   // was @default(NEW)
  sla_due_at           DateTime?            @db.Timestamp(6)   // ADD THIS LINE
  created_at           DateTime             @default(now()) @db.Timestamp(6)
  updated_at           DateTime             @updatedAt @db.Timestamp(6)
  final_outcome        CaseStatus?
  parent_id            Int?
  case_type            CaseType?
  case_creation_type   CaseCreationType
  alert                Alert?
  comments             Comment[]
  tasks                Task[]
  evidence             Evidence[]
  transactionProfiles  TransactionProfile[]

  @@index([sla_due_at])   // ADD THIS LINE
  @@map("cases")
}
```

Note: `Task.sla_deadline` and `Task.sla_duration_hours` (lines 135–136) are task-level SLA fields used by the existing notification system — leave them untouched.

**Change 3 — Add `SlaPolicy` model (insert after the `ReferenceId` model, before the `Priority` enum):**

```prisma
model SlaPolicy {
  id              Int      @id @default(autoincrement())
  tenant_id       String
  priority        Priority
  target_hours    Int
  at_risk_pct     Float    @default(0.5)
  due_soon_pct    Float    @default(0.8)
  created_at      DateTime @default(now()) @db.Timestamp(6)
  updated_at      DateTime @updatedAt @db.Timestamp(6)

  @@unique([tenant_id, priority])
  @@map("sla_policies")
}
```

**Prisma migration commands:**
```bash
cd backend
npx prisma migrate dev --name "divorce_sla_from_priority"
```

The migration will contain:
1. `ALTER TYPE "Priority" RENAME VALUE 'NEW' TO 'LOW'` — Postgres supports in-place enum rename
2. `ALTER TYPE "Priority" RENAME VALUE 'URGENT' TO 'MEDIUM'`
3. `ALTER TYPE "Priority" RENAME VALUE 'CRITICAL' TO 'MEDIUM'` — **cannot do two renames to the same value in a single ALTER**; this requires a two-step: rename `CRITICAL` to `MEDIUM_OLD`, then remove after data migration
4. `ALTER TYPE "Priority" RENAME VALUE 'BREACH' TO 'HIGH'`
5. `ALTER TABLE "cases" ADD COLUMN "sla_due_at" TIMESTAMP(6)`
6. `CREATE TABLE "sla_policies" (...)`

The `CRITICAL → MEDIUM` collapse requires a data migration script (see Step B9).

---

#### Step B2 — `SlaPolicy` Seed Data (`backend/prisma/seed.ts` or a migration seed)

Add default system-wide rows for all three priorities. Without these, the SLA functions have no config to resolve against.

```ts
// In prisma/seed.ts or a dedicated migration
await prisma.slaPolicy.createMany({
  data: [
    { tenant_id: 'SYSTEM_DEFAULT', priority: 'LOW',    target_hours: 168, at_risk_pct: 0.5, due_soon_pct: 0.8 },
    { tenant_id: 'SYSTEM_DEFAULT', priority: 'MEDIUM', target_hours: 72,  at_risk_pct: 0.5, due_soon_pct: 0.8 },
    { tenant_id: 'SYSTEM_DEFAULT', priority: 'HIGH',   target_hours: 24,  at_risk_pct: 0.5, due_soon_pct: 0.8 },
  ],
  skipDuplicates: true,
});
```

---

#### Step B3 — New File: `sla-state.util.ts`

Create `backend/src/modules/shared/utils/sla-state.util.ts`. This is a pure function with no dependencies — easily unit-tested in isolation.

```ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../../../prisma/prisma.service';
import { Priority } from '@prisma/client-cms';

export type SlaState = 'ON_TRACK' | 'AT_RISK' | 'DUE_SOON' | 'BREACHED';

export interface SlaPolicy {
  target_hours: number;
  at_risk_pct: number;
  due_soon_pct: number;
}

export const SYSTEM_DEFAULT_SLA_POLICY: Record<Priority, SlaPolicy> = {
  HIGH:   { target_hours: 24,  at_risk_pct: 0.5, due_soon_pct: 0.8 },
  MEDIUM: { target_hours: 72,  at_risk_pct: 0.5, due_soon_pct: 0.8 },
  LOW:    { target_hours: 168, at_risk_pct: 0.5, due_soon_pct: 0.8 },
};

export function determineSlaState(
  now: Date,
  slaDueAt: Date,
  createdAt: Date,
  policy: SlaPolicy,
): SlaState {
  const totalMs = policy.target_hours * 60 * 60 * 1000;
  const elapsedMs = now.getTime() - createdAt.getTime();
  const progress = elapsedMs / totalMs;

  if (progress >= 1.0)                  return 'BREACHED';
  if (progress >= policy.due_soon_pct)  return 'DUE_SOON';
  if (progress >= policy.at_risk_pct)   return 'AT_RISK';
  return 'ON_TRACK';
}

export function computeSlaDueAt(createdAt: Date, policy: SlaPolicy): Date {
  return new Date(createdAt.getTime() + policy.target_hours * 60 * 60 * 1000);
}

@Injectable()
export class SlaStateUtil {
  constructor(private readonly prisma: PrismaService) {}

  async getPolicy(tenantId: string, priority: Priority): Promise<SlaPolicy> {
    const row = await this.prisma.slaPolicy.findFirst({
      where: { tenant_id: tenantId, priority },
    });
    if (row) return { target_hours: row.target_hours, at_risk_pct: row.at_risk_pct, due_soon_pct: row.due_soon_pct };

    const systemDefault = await this.prisma.slaPolicy.findFirst({
      where: { tenant_id: 'SYSTEM_DEFAULT', priority },
    });
    if (systemDefault) return { target_hours: systemDefault.target_hours, at_risk_pct: systemDefault.at_risk_pct, due_soon_pct: systemDefault.due_soon_pct };

    return SYSTEM_DEFAULT_SLA_POLICY[priority];
  }

  async getSlaStateForCase(caseId: number, tenantId: string, priority: Priority, createdAt: Date): Promise<SlaState> {
    const policy = await this.getPolicy(tenantId, priority);
    return determineSlaState(new Date(), computeSlaDueAt(createdAt, policy), createdAt, policy);
  }
}
```

Register `SlaStateUtil` as a provider in `SharedModule` (or wherever `CasePriorityUtil` is registered).

---

#### Step B4 — Rewrite `case-priority.util.ts`

**File:** `backend/src/modules/shared/utils/case-priority.util.ts`

```ts
// FULL FILE REPLACEMENT
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { Priority } from '@prisma/client-cms';

@Injectable()
export class CasePriorityUtil {
  constructor(private readonly configService: ConfigService) {}

  // Maps a 0..1 risk score from the Tazama scoring engine to a severity bucket.
  // Thresholds are configurable but default to thirds.
  // This function must NEVER be called with an SLA time-progress ratio.
  determinePriority(riskScore: number): Priority {
    const highThreshold = parseFloat(this.configService.get<string>('PRIORITY_HIGH_THRESHOLD', '0.66'));
    const mediumThreshold = parseFloat(this.configService.get<string>('PRIORITY_MEDIUM_THRESHOLD', '0.33'));

    if (riskScore >= highThreshold)   return Priority.HIGH;
    if (riskScore >= mediumThreshold) return Priority.MEDIUM;
    return Priority.LOW;
  }
}
```

Rename env vars (update `.env.example` and deployment config):
- `PRIORITY_FIRST_HALF` → `PRIORITY_MEDIUM_THRESHOLD` (default `0.33`)
- `PRIORITY_SECOND_HALF` → `PRIORITY_HIGH_THRESHOLD` (default `0.66`)
- `PRIORITY_THIRD_HALF` → *(removed — no longer needed)*

---

#### Step B5 — Rewrite `alert-priority.service.ts`

The cron job's new responsibility: derive `sla_state` per open alert, fire idempotent escalation notifications. It no longer writes `priority` to anything (Track A already removed the case write; now remove the alert priority write too, since that was also SLA-derived).

**File:** `backend/src/modules/alert-priority/alert-priority.service.ts`

```ts
import { Injectable, Logger, OnModuleInit } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { EventEmitter2 } from '@nestjs/event-emitter';
import { PrismaService } from '../../../prisma/prisma.service';
import { SlaStateUtil, determineSlaState, computeSlaDueAt } from '../shared/utils/sla-state.util';
import type { SlaState } from '../shared/utils/sla-state.util';
import { Priority } from '@prisma/client-cms';

// In-memory deduplication: tracks which (caseId, slaState) notifications have
// already been sent. Cleared on process restart — acceptable: a restart produces
// at most one duplicate notification per open case.
const sentNotifications = new Map<string, SlaState>();

@Injectable()
export class AlertPriorityService implements OnModuleInit {
  private readonly logger = new Logger(AlertPriorityService.name);

  constructor(
    private readonly prisma: PrismaService,
    private readonly configService: ConfigService,
    private readonly eventEmitter: EventEmitter2,
    private readonly slaStateUtil: SlaStateUtil,
  ) {}

  onModuleInit(): void {
    this.logger.log('SLA escalation service initialized.');
  }

  async runRecalculation(): Promise<void> {
    this.logger.log('Starting SLA escalation check...');

    const openCases = await this.prisma.case.findMany({
      where: {
        status: {
          notIn: [
            'STATUS_71_AUTOCLOSED_CONFIRMED',
            'STATUS_72_AUTOCLOSED_REFUTED',
            'STATUS_81_CLOSED_REFUTED',
            'STATUS_82_CLOSED_CONFIRMED',
            'STATUS_83_CLOSED_INCONCLUSIVE',
            'STATUS_84_COMPLETED',
          ],
        },
      },
      select: {
        case_id: true,
        tenant_id: true,
        priority: true,
        created_at: true,
        sla_due_at: true,
        case_owner_user_id: true,
        alert: { select: { alert_id: true } },
      },
    });

    if (!openCases.length) {
      this.logger.log('No open cases to process.');
      return;
    }

    await Promise.all(
      openCases.map(async (c) => {
        try {
          const policy = await this.slaStateUtil.getPolicy(c.tenant_id, c.priority);
          const slaDueAt = c.sla_due_at ?? computeSlaDueAt(c.created_at, policy);
          const slaState = determineSlaState(new Date(), slaDueAt, c.created_at, policy);
          const dedupeKey = `${c.case_id}:${slaState}`;
          const isOwned = !!c.case_owner_user_id;

          if (sentNotifications.get(dedupeKey) === slaState) {
            return; // already notified for this state
          }

          if (slaState === 'AT_RISK' && !isOwned) {
            this.eventEmitter.emit('case.sla-at-risk-unowned', { caseId: c.case_id, tenantId: c.tenant_id, slaState, slaDueAt });
          } else if (slaState === 'DUE_SOON' && isOwned) {
            this.eventEmitter.emit('case.sla-due-soon-owned', { caseId: c.case_id, tenantId: c.tenant_id, slaState, slaDueAt, ownerId: c.case_owner_user_id });
          } else if (slaState === 'DUE_SOON' && !isOwned) {
            this.eventEmitter.emit('case.sla-due-soon-unowned', { caseId: c.case_id, tenantId: c.tenant_id, slaState, slaDueAt });
          } else if (slaState === 'BREACHED') {
            this.eventEmitter.emit('case.sla-breached', { caseId: c.case_id, tenantId: c.tenant_id, slaState, slaDueAt, ownerId: c.case_owner_user_id });
          }

          sentNotifications.set(dedupeKey, slaState);
          this.logger.debug(`Case ${c.case_id}: sla_state=${slaState}, priority=${c.priority}`);
        } catch (err) {
          this.logger.error(`Failed to process case ${c.case_id}: ${err}`, (err as Error).stack);
        }
      }),
    );

    this.logger.log('SLA escalation check complete.');
  }
}
```

Update `AlertPriorityModule` to inject `EventEmitter2` and `SlaStateUtil`.

---

#### Step B6 — Stamp `sla_due_at` at Case Creation

Three files set `priority` at creation and must also now stamp `sla_due_at`.

**`backend/src/modules/case/services/case-creation.service.ts` — `manualCaseCreation` (around line 136):**

```ts
// BEFORE
const priority = this.casePriorityUtil.determinePriority(priorityScore);
const caseDetail: CreateCaseDto = {
  tenantId,
  ...
  priority,
  ...
};

// AFTER
const priority = this.casePriorityUtil.determinePriority(priorityScore);
const slaPolicy = await this.slaStateUtil.getPolicy(tenantId, priority);
const sla_due_at = computeSlaDueAt(new Date(), slaPolicy);
const caseDetail: CreateCaseDto = {
  tenantId,
  ...
  priority,
  sla_due_at,
  ...
};
```

Apply the same pattern in:
- `backend/src/modules/triage/triage.service.ts` — wherever `createCaseWithInvestigationTask` is called after `determinePriority` (around line 302)
- `backend/src/modules/case/services/case-creation-approval.service.ts` — around line 62

Inject `SlaStateUtil` into each of these three services.

---

#### Step B7 — Add Priority Actuator Endpoint

New file: `backend/src/modules/case/services/case-priority.service.ts`

```ts
import { Injectable, NotFoundException, ForbiddenException } from '@nestjs/common';
import { PrismaService } from '../../../prisma/prisma.service';
import { EventEmitter2 } from '@nestjs/event-emitter';
import { Priority } from '@prisma/client-cms';
import { SlaStateUtil, computeSlaDueAt } from '../../shared/utils/sla-state.util';

@Injectable()
export class CasePriorityService {
  constructor(
    private readonly prisma: PrismaService,
    private readonly eventEmitter: EventEmitter2,
    private readonly slaStateUtil: SlaStateUtil,
  ) {}

  async changePriority(
    caseId: number,
    tenantId: string,
    actorId: string,
    newPriority: Priority,
    reason: string,
  ): Promise<void> {
    const existing = await this.prisma.case.findFirst({
      where: { case_id: caseId, tenant_id: tenantId },
      select: { case_id: true, priority: true, created_at: true },
    });
    if (!existing) throw new NotFoundException(`Case ${caseId} not found`);

    const oldPriority = existing.priority;
    if (oldPriority === newPriority) return;

    // Re-anchor deadline on created_at (not now) — non-gameable
    const policy = await this.slaStateUtil.getPolicy(tenantId, newPriority);
    const sla_due_at = computeSlaDueAt(existing.created_at, policy);

    await this.prisma.case.update({
      where: { case_id: caseId },
      data: { priority: newPriority, sla_due_at },
    });

    this.eventEmitter.emit('case.priority-changed', {
      caseId,
      tenantId,
      actorId,
      oldPriority,
      newPriority,
      reason,
      timestamp: new Date(),
      newSlaDueAt: sla_due_at,
    });
  }
}
```

Add a `PATCH /cases/:id/priority` endpoint in `case.controller.ts` gated to `SUPERVISOR` role, calling this service. The `PriorityChanged` event is consumed by the notification service to send an audit email/log entry.

---

#### Step B8 — Backend Enum Reference Sweep

All files that reference the old enum values by name. For each: replace string literals and imports as shown.

**`backend/src/modules/report/report.service.ts` (lines 470–491):**

```ts
// BEFORE
const lowPriorityCases = allCases.filter(
  (c) => c.priority === Priority.NEW && ...
).length;
const mediumPriorityCases = allCases.filter(
  (c) => (c.priority === Priority.CRITICAL || c.priority === Priority.URGENT) && ...
).length;
const highPriorityCases = allCases.filter(
  (c) => c.priority === Priority.BREACH && ...
).length;

// AFTER
const lowPriorityCases = allCases.filter(
  (c) => c.priority === Priority.LOW && ...
).length;
const mediumPriorityCases = allCases.filter(
  (c) => c.priority === Priority.MEDIUM && ...
).length;
const highPriorityCases = allCases.filter(
  (c) => c.priority === Priority.HIGH && ...
).length;
```

**`backend/src/constants/notification.constants.ts` (line 336–347, `getPriorityColor`):**

```ts
// BEFORE
export function getPriorityColor(priority?: string): string {
  switch (priority?.toUpperCase()) {
    case 'HIGH':   return '#dc3545';
    case 'MEDIUM': return '#ffc107';
    case 'LOW':
    case 'NORMAL':
    default:       return '#28a745';
  }
}

// AFTER — function body is already correct (uses HIGH/MEDIUM/LOW);
// only change: remove 'NORMAL' fallback alias, add explicit LOW case
export function getPriorityColor(priority?: string): string {
  switch (priority?.toUpperCase()) {
    case 'HIGH':   return '#dc3545';
    case 'MEDIUM': return '#ffc107';
    case 'LOW':    return '#28a745';
    default:       return '#6c757d'; // unknown priority — neutral gray
  }
}
```

Note: the `slaBreach` template at line 200–207 already uses `HIGH/MEDIUM/LOW/INFO` in its severity colour map — that block needs no change.

**`backend/src/utils/interfaces/notification.interface.ts` — add `SlaEventPayload` extension:**

```ts
// ADD new notification types for case-level SLA events
export type NotificationType =
  | 'TASK_ASSIGNED'
  // ... existing types ...
  | 'TASK_SLA_WARNING'
  | 'TASK_SLA_BREACH'
  | 'CASE_SLA_AT_RISK'        // ADD
  | 'CASE_SLA_DUE_SOON'       // ADD
  | 'CASE_SLA_BREACHED'       // ADD
  | 'TASK_OVERDUE'
  | 'GENERIC';

// ADD case SLA payload
export interface CaseSlaPayload {
  caseId: number;
  tenantId: string;
  slaState: 'AT_RISK' | 'DUE_SOON' | 'BREACHED';
  slaDueAt: Date;
  ownerId?: string | null;
}
```

**`backend/src/modules/triage/triage.service.ts` — priority notification calls (lines 429, 451, 541, 599, 642, 664):**

All these pass `casePriority: alert.priority` or `casePriority: priority` into notification payloads. No code change needed here — the payload value is whatever `Priority` the case holds, which will now be `LOW/MEDIUM/HIGH`. The notification template renders it as a string — it just works.

**`backend/src/modules/case/services/case-creation.service.ts` and `case-creation-approval.service.ts`:**

Import `SlaStateUtil` and `computeSlaDueAt` (Step B6 already covers the logic change). No further enum-related changes needed here — `determinePriority` now returns the new values.

**`backend/src/modules/alert/alert.statistics.service.ts` (line 78):**

```ts
// BEFORE
whereClause.priority = priority.toUpperCase() as Priority;

// AFTER — same line, but valid values from caller are now LOW/MEDIUM/HIGH
whereClause.priority = priority.toUpperCase() as Priority;
// No code change; update API docs / OpenAPI spec to advertise new enum values
```

---

#### Step B9 — Data Migration Script

Create `backend/prisma/migrations/<timestamp>_priority_data_backfill/migration.sql` (or run as a seeded migration):

```sql
-- Step 1: Collapse CRITICAL → MEDIUM (done before enum rename)
UPDATE cases   SET priority = 'URGENT'  WHERE priority = 'CRITICAL';
UPDATE alerts  SET priority = 'URGENT'  WHERE priority = 'CRITICAL';

-- Step 2: Rename enum values in Postgres
-- Prisma generates these; shown here for review
ALTER TYPE "Priority" RENAME VALUE 'NEW'     TO 'LOW';
ALTER TYPE "Priority" RENAME VALUE 'URGENT'  TO 'MEDIUM';
ALTER TYPE "Priority" RENAME VALUE 'BREACH'  TO 'HIGH';
-- CRITICAL is already gone (collapsed to URGENT above, now MEDIUM)

-- Step 3: Backfill sla_due_at for open cases
-- Uses MEDIUM target (72h) as the safe default — avoids false BREACHED for old NEW cases
UPDATE cases
SET sla_due_at = created_at + INTERVAL '72 hours'
WHERE sla_due_at IS NULL
  AND status NOT IN (
    'STATUS_71_AUTOCLOSED_CONFIRMED','STATUS_72_AUTOCLOSED_REFUTED',
    'STATUS_81_CLOSED_REFUTED','STATUS_82_CLOSED_CONFIRMED',
    'STATUS_83_CLOSED_INCONCLUSIVE','STATUS_84_COMPLETED'
  );

-- Closed cases don't need sla_due_at — leave NULL
```

---

#### Step B10 — Frontend Enum Sweep

**`frontend/src/features/alerts/types/triage.types.ts` (lines 3–8):**

```ts
// BEFORE
export const Priority = {
  NEW: 'NEW',
  URGENT: 'URGENT',
  CRITICAL: 'CRITICAL',
  BREACH: 'BREACH',
} as const;

// AFTER
export const Priority = {
  LOW: 'LOW',
  MEDIUM: 'MEDIUM',
  HIGH: 'HIGH',
} as const;
export type Priority = typeof Priority[keyof typeof Priority];
```

**`frontend/src/features/alerts/utils/alertTransformers.ts` (lines 108–135):**

```ts
// BEFORE
function mapPriorityToSeverity(priority: Priority): 'low' | 'medium' | 'high' | 'critical' {
  switch (priority) {
    case 'NEW':      return 'low';
    case 'URGENT':   return 'medium';
    case 'CRITICAL': return 'high';
    case 'BREACH':   return 'critical';
  }
}

export function mapSeverityToPriority(severity: 'low' | 'medium' | 'high' | 'critical'): Priority {
  switch (severity) {
    case 'low':      return 'NEW';
    case 'medium':   return 'URGENT';
    case 'high':     return 'CRITICAL';
    case 'critical': return 'BREACH';
  }
}

// AFTER
function mapPriorityToSeverity(priority: Priority): 'low' | 'medium' | 'high' {
  switch (priority) {
    case 'LOW':    return 'low';
    case 'MEDIUM': return 'medium';
    case 'HIGH':   return 'high';
  }
}

export function mapSeverityToPriority(severity: 'low' | 'medium' | 'high'): Priority {
  switch (severity) {
    case 'low':    return 'LOW';
    case 'medium': return 'MEDIUM';
    case 'high':   return 'HIGH';
  }
}
```

**`frontend/src/features/cases/components/casesTable.utils.ts` (lines 62–70):**

```ts
// BEFORE
export const getPriorityColor = (priority: string): string => {
  const priorityColors: Record<string, string> = {
    NEW:      'bg-blue-50 text-blue-700 ring-blue-200',
    URGENT:   'bg-yellow-50 text-yellow-700 ring-yellow-200',
    CRITICAL: 'bg-orange-50 text-orange-700 ring-orange-200',
    BREACH:   'bg-red-50 text-red-700 ring-red-200',
  };
  return priorityColors[priority] || 'bg-gray-50 text-gray-700 ring-gray-200';
};

// AFTER
export const getPriorityColor = (priority: string): string => {
  const priorityColors: Record<string, string> = {
    LOW:    'bg-blue-50 text-blue-700 ring-blue-200',
    MEDIUM: 'bg-yellow-50 text-yellow-700 ring-yellow-200',
    HIGH:   'bg-red-50 text-red-700 ring-red-200',
  };
  return priorityColors[priority] || 'bg-gray-50 text-gray-700 ring-gray-200';
};
```

**`frontend/src/features/cases/components/CaseFilters.tsx` (lines 85–91):**

```tsx
// BEFORE
const priorityOptions = [
  { value: '',         label: 'All Priorities' },
  { value: 'NEW',      label: 'New' },
  { value: 'URGENT',   label: 'Urgent' },
  { value: 'CRITICAL', label: 'Critical' },
  { value: 'BREACH',   label: 'Breach' },
];

// AFTER
const priorityOptions = [
  { value: '',       label: 'All Priorities' },
  { value: 'LOW',    label: 'Low' },
  { value: 'MEDIUM', label: 'Medium' },
  { value: 'HIGH',   label: 'High' },
];
```

**`frontend/src/features/alerts/components/ManualTriageModal.tsx` (lines 193–199, 285–291):**

```tsx
// BEFORE (both occurrences)
priority === 'BREACH'
  ? 'text-red-600 border-red-200'
  : priority === 'CRITICAL'
    ? 'text-orange-600 border-orange-200'
    : priority === 'URGENT'
      ? 'text-yellow-600 border-yellow-200'
      : 'text-blue-600 border-blue-200'

// AFTER (both occurrences)
priority === 'HIGH'
  ? 'text-red-600 border-red-200'
  : priority === 'MEDIUM'
    ? 'text-yellow-600 border-yellow-200'
    : 'text-blue-600 border-blue-200'
```

**Frontend test files (update fixture data — all `triage.types.test.ts`, `alertTransformers.test.ts`, `AlertsDetailModal.test.tsx`, `useAlertOperations.test.ts`, `triageservice.test.ts`):**

In every test fixture: replace `Priority.NEW → Priority.LOW`, `Priority.URGENT → Priority.MEDIUM`, `Priority.CRITICAL → Priority.MEDIUM`, `Priority.BREACH → Priority.HIGH`. String literal occurrences: `'NEW' → 'LOW'`, `'URGENT' → 'MEDIUM'`, `'CRITICAL' → 'MEDIUM'`, `'BREACH' → 'HIGH'`.

The transformer test for `mapSeverityToPriority` must also drop the `'critical' → 'BREACH'` case:
```ts
// alertTransformers.test.ts — AFTER
expect(transformers.mapSeverityToPriority('low')).toBe('LOW');
expect(transformers.mapSeverityToPriority('medium')).toBe('MEDIUM');
expect(transformers.mapSeverityToPriority('high')).toBe('HIGH');
```

---

## Test Cases

### Unit Tests (new — add to test suite)

#### `sla-state.util.spec.ts`

```ts
describe('determineSlaState', () => {
  const policy = { target_hours: 72, at_risk_pct: 0.5, due_soon_pct: 0.8 };
  const createdAt = new Date('2026-01-01T00:00:00Z');

  it('returns ON_TRACK when elapsed < 50% of budget', () => {
    const now = new Date('2026-01-01T35:00:00Z'); // 35h < 36h threshold
    expect(determineSlaState(now, computeSlaDueAt(createdAt, policy), createdAt, policy)).toBe('ON_TRACK');
  });

  it('returns AT_RISK when elapsed >= 50% and < 80%', () => {
    const now = new Date('2026-01-01T37:00:00Z'); // 37h, past 36h threshold
    expect(determineSlaState(now, computeSlaDueAt(createdAt, policy), createdAt, policy)).toBe('AT_RISK');
  });

  it('returns DUE_SOON when elapsed >= 80% and < 100%', () => {
    const now = new Date('2026-01-01T58:00:00Z'); // 58h, past 57.6h threshold
    expect(determineSlaState(now, computeSlaDueAt(createdAt, policy), createdAt, policy)).toBe('DUE_SOON');
  });

  it('returns BREACHED when elapsed >= 100%', () => {
    const now = new Date('2026-01-04T00:00:00Z'); // 72h exactly
    expect(determineSlaState(now, computeSlaDueAt(createdAt, policy), createdAt, policy)).toBe('BREACHED');
  });

  it('can move backwards after de-escalation (HIGH -> LOW lengthens budget)', () => {
    // Case open 20h, priority was HIGH (24h budget → BREACHED), now changed to LOW (168h budget)
    const policyLow = { target_hours: 168, at_risk_pct: 0.5, due_soon_pct: 0.8 };
    const now = new Date('2026-01-01T20:00:00Z');
    expect(determineSlaState(now, computeSlaDueAt(createdAt, policyLow), createdAt, policyLow)).toBe('ON_TRACK');
  });
});
```

#### `case-priority.util.spec.ts`

```ts
describe('CasePriorityUtil.determinePriority', () => {
  it('maps score < 0.33 to LOW', () => {
    expect(util.determinePriority(0.1)).toBe(Priority.LOW);
  });
  it('maps score >= 0.33 and < 0.66 to MEDIUM', () => {
    expect(util.determinePriority(0.5)).toBe(Priority.MEDIUM);
  });
  it('maps score >= 0.66 to HIGH', () => {
    expect(util.determinePriority(0.9)).toBe(Priority.HIGH);
  });
  it('maps score = 1.0 to HIGH (not BREACHED — BREACH is SLA, not severity)', () => {
    expect(util.determinePriority(1.0)).toBe(Priority.HIGH);
  });
});
```

### Integration / E2E Tests

#### Priority Stability (regression for the original sin)

```
Scenario: hourly cron does not overwrite case priority

1. Triage alert with priorityScore=0.9 → case.priority = HIGH
2. Wait (or mock) for AlertPriorityService.runRecalculation() to run
3. Assert: case.priority is still HIGH
4. Assert: alert.priority is still HIGH (cron no longer writes alert priority either)
```

#### SLA Due-At Stamped at Creation

```
Scenario: case created with HIGH priority gets sla_due_at = created_at + 24h

1. Create case with priorityScore=0.9 (→ HIGH, 24h SLA)
2. Read case from DB
3. Assert: sla_due_at ≈ created_at + 24h (within 1 second)
```

#### Priority Re-Anchor on Change

```
Scenario: changing priority re-anchors sla_due_at on created_at, not now

1. Create case with MEDIUM priority (72h SLA), created_at = T
2. Wait simulated 50h (mock clock to T+50)
3. Call PATCH /cases/:id/priority { newPriority: HIGH, reason: "..." }
4. Assert: sla_due_at = T + 24h (not T+50 + 24h)
5. Assert: sla_state is now BREACHED (50h > 24h HIGH budget)
6. Assert: escalation event fired (case.sla-breached emitted)
```

#### Idempotent Escalation Notifications

```
Scenario: same SLA state does not fire duplicate notifications

1. Case is at AT_RISK state
2. Run cron twice
3. Assert: case.sla-at-risk-unowned event emitted exactly once
4. Case transitions to DUE_SOON
5. Run cron once
6. Assert: case.sla-due-soon-* event emitted once (new state → new notification)
```

#### Priority Filter Returns Correct Cases

```
Scenario: filtering by HIGH returns severity-high cases, not time-expired cases

1. Create case A: priorityScore=0.9 (HIGH), opened 5 minutes ago
2. Create case B: priorityScore=0.1 (LOW), opened 100 hours ago
3. GET /cases?priority=HIGH
4. Assert: response contains case A
5. Assert: response does NOT contain case B
```

#### Workload Report Priority Buckets

```
Scenario: workload report HIGH count reflects severity, not SLA age

1. Create 3 cases: HIGH/MEDIUM/LOW priority, all recently opened
2. GET /reports/workload
3. Assert: highPriorityCases = 1 (only the HIGH severity case)
4. Assert: lowPriorityCases = 1 (only the LOW severity case)
```

#### Data Migration Validation

```sql
-- Run after migration, assert zero rows in old enum values
SELECT COUNT(*) FROM cases  WHERE priority NOT IN ('LOW','MEDIUM','HIGH');  -- must be 0
SELECT COUNT(*) FROM alerts WHERE priority NOT IN ('LOW','MEDIUM','HIGH');  -- must be 0

-- Assert all open cases have sla_due_at populated
SELECT COUNT(*) FROM cases
WHERE sla_due_at IS NULL
  AND status NOT IN (
    'STATUS_71_AUTOCLOSED_CONFIRMED','STATUS_72_AUTOCLOSED_REFUTED',
    'STATUS_81_CLOSED_REFUTED','STATUS_82_CLOSED_CONFIRMED',
    'STATUS_83_CLOSED_INCONCLUSIVE','STATUS_84_COMPLETED'
  );
-- must be 0
```

### Manual / UAT Checks

| # | Scenario | Steps | Expected Result |
|---|---|---|---|
| M1 | Priority not overwritten | Set a case to HIGH. Wait 1h. Reload case. | Priority still HIGH |
| M2 | Case filter by severity | Filter cases by HIGH. | Returns only high-risk cases, not all old cases |
| M3 | Workload report buckets | Open workload report. | HIGH = cases with high risk score, not cases past 72h |
| M4 | Priority badge colours | View case list. | HIGH = red, MEDIUM = yellow, LOW = blue |
| M5 | Manual triage shows correct label | Triage an alert with high risk score. | Priority auto-calc shows HIGH, not BREACH |
| M6 | Priority change by supervisor | Supervisor changes case from MEDIUM to HIGH. | Priority updates; `sla_due_at` recalculates from `created_at` |
| M7 | SLA notification not duplicated | Leave a case in AT_RISK state. Check email logs after 2 cron cycles. | At-risk notification appears once, not twice |
| M8 | Backfilled cases visible | View cases opened before migration. | All show LOW/MEDIUM/HIGH badge (no blank or unknown) |

---

## Overall Impact of the Fix

| Area | Before | After |
|---|---|---|
| `case.priority` | Overwritten every hour by SLA clock | Stable; set once at triage, changed only via audited actuator |
| Severity vs SLA | Merged into one field | Orthogonal: `priority` = severity, `sla_due_at` / `sla_state` = timing |
| Priority filter | Returns cases by SLA age | Returns cases by inherent risk score |
| Workload report | HIGH = past 72h | HIGH = high-risk severity |
| SLA visibility | Invisible (encoded as `BREACH`) | Explicit `sla_state` computable at read time |
| Escalation notifications | Potentially re-fired every tick | Idempotent: at most one notification per `(case_id, sla_state)` pair |
| Audit trail | No record of who escalated | `PriorityChanged` event records actor, timestamp, old/new values, reason |
| Multi-tenant SLA targets | Single env var for all tenants | Per-tenant `SlaPolicy` table; per-priority SLA budgets |
| DB enum values | `NEW / URGENT / CRITICAL / BREACH` | `LOW / MEDIUM / HIGH` |

---

## Key File Reference

| File | Role in the defect |
|---|---|
| [alert-priority.service.ts](../case-management-system/backend/src/modules/alert-priority/alert-priority.service.ts) | The original sin — writes `case.priority` from SLA time ratio every cron tick |
| [alert-priority.task.ts](../case-management-system/backend/src/modules/alert-priority/alert-priority.task.ts) | Cron trigger — default `0 * * * *` (hourly) |
| [case-priority.util.ts](../case-management-system/backend/src/modules/shared/utils/case-priority.util.ts) | Shared mapper used at both creation and in the cron — must become 3-bucket severity mapper |
| [schema.prisma:241–245](../case-management-system/backend/prisma/schema.prisma) | `Priority` enum definition; `sla_deadline` / `sla_duration_hours` already present but unused by cron |
| [report.service.ts:470–491](../case-management-system/backend/src/modules/report/report.service.ts) | Workload report silently re-maps 4-value enum to 3-bucket labels — accidentally correct model, wrong source values |
| [casesTable.utils.ts:62–69](../case-management-system/frontend/src/features/cases/components/casesTable.utils.ts) | Frontend badge colours hardcoded to `NEW/URGENT/CRITICAL/BREACH` |
| [CaseFilters.tsx:87–90](../case-management-system/frontend/src/features/cases/components/CaseFilters.tsx) | Frontend filter dropdown hardcoded to `NEW/URGENT/CRITICAL/BREACH` |

---

## Fix Summary

The fix for issue #220 resolves a fundamental design error: the `Priority` enum was made to carry two completely different pieces of information at once — how serious a case inherently is, and how much time has passed since it was opened. A background cron job ran every hour and overwrote `case.priority` with a value derived purely from elapsed time divided by a 72-hour SLA window. Any priority set by a supervisor or the triage system based on actual risk was silently replaced within the hour. As a result, investigators could not trust the priority field, filters by priority returned cases sorted by age rather than severity, and workload reports flagged cases as "high priority" simply because they had been sitting open for more than three days.

The fix separates the two concepts permanently into orthogonal fields. Priority becomes a stable severity label — `LOW`, `MEDIUM`, or `HIGH` — set once at triage from the Tazama risk score and only changeable through an explicit, audited supervisor action. It is never written by the SLA job again. SLA tracking becomes a derived calculation: a `sla_due_at` deadline is stamped on the case at creation based on its priority and the tenant's configured SLA target for that severity level. At any point, the SLA state — `ON_TRACK`, `AT_RISK`, `DUE_SOON`, or `BREACHED` — is computed on read by comparing the current time to that stored deadline. Nothing is overwritten; nothing can go stale.

Track A of the fix delivers the most critical part in a single-file change and can ship independently without any schema migration or frontend work. It removes the nine lines in `alert-priority.service.ts` that write `case.priority` from the SLA time ratio. After that one deletion, priority is stable. Cases triaged as high-risk remain high-risk regardless of how old they are. The cron job continues updating alert-level priority and score for alert-dashboard display, which is a reasonable use of the time ratio at that level. Track A is safe to merge immediately.

Track B completes the design as a follow-on release. It renames the `Priority` enum values from `NEW / URGENT / CRITICAL / BREACH` to `LOW / MEDIUM / HIGH`, removing `NEW` (which encoded "recently opened", not a severity) and `BREACH` (which encoded "past deadline", an SLA state, not a severity level). The schema gains a `sla_due_at` column on the `Case` model and a new `SlaPolicy` table that holds per-tenant, per-priority SLA targets, making the system genuinely multi-tenant rather than driven by a single environment variable. A new `sla-state.util.ts` pure function computes `sla_state` from `now` versus the stored deadline, receiving the policy as input so per-priority SLA budgets are structurally isolated and can diverge in future without changing call sites. The rewritten cron job stops writing any canonical state entirely; its sole remaining job is to derive `sla_state` per open case and fire idempotent escalation notifications keyed on `(case_id, sla_state)` pairs, ensuring each state change triggers at most one notification rather than re-firing every tick. A new priority actuator endpoint lets supervisors change severity mid-lifecycle through an audited `PriorityChanged` event that re-anchors the deadline on the case's original `created_at` timestamp — a deliberate, non-gameable choice ensuring an already-old case escalated to `HIGH` immediately reflects its urgency rather than receiving a fresh clock.

The frontend enum sweep updates badge colours, filter dropdowns, and test fixtures across approximately 55 files to the new three-value vocabulary. Three competing priority vocabularies already in the tree — `NEW/URGENT/CRITICAL/BREACH` in most code, `Low/Medium/High` in report workload buckets, and `NORMAL` in notification templates — converge on a single consistent set. A data migration backfills `sla_due_at` for all open cases using the `MEDIUM` default target and collapses `CRITICAL` and `URGENT` into `MEDIUM` before the enum rename, since both were mid-tier labels for the same severity band.

The end result is a system where priority is meaningful, stable, and auditable; SLA state is always current without needing a job to maintain it; reports reflect actual case risk rather than case age; and the escalation path is well-defined, idempotent, and tied to the correct operational signals rather than a continuous per-tick overwrite.
