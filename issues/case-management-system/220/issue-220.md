# Issue #220 — Divorce SLA Management from Prioritisation Beyond Creation

**Repository:** tazama-lf/case-management-system  
**Issue:** [#220 — Divorce SLA management from prioritisation beyond creation](https://github.com/tazama-lf/case-management-system/issues/220)  
**Author:** Justus-at-Tazama  
**State:** Open  
**Report Date:** 2026-06-29  

---

## Executive Summary

The `Priority` enum (`NEW / URGENT / CRITICAL / BREACH`) conflates two orthogonal concepts — **inherent severity** and **SLA time progress** — into a single field that a background cron job overwrites every hour.

The result: investigators cannot assign or trust a case's priority, because the field does not represent "how serious is this case?" but rather "how long has this case been sitting here?" A supervisor who escalates a case to `CRITICAL` based on financial exposure will have that label silently overwritten the next time the cron fires, reverting it to whatever the SLA clock says at that moment.

The issue's author has already delivered a complete design document (the issue body) that separates the two axes into a `priority` severity field (`LOW / MEDIUM / HIGH`) and a derived `sla_state` (`ON_TRACK / AT_RISK / DUE_SOON / BREACHED`). This report maps the broken code exactly, quantifies the blast radius, and produces a delivery plan aligned to the settled design.

---

## How It Works Today — Confirmed in Code

### The "Original Sin" — `alert-priority.service.ts`

The root of the defect is a single equation at [backend/src/modules/alert-priority/alert-priority.service.ts:43–45](../case-management-system/backend/src/modules/alert-priority/alert-priority.service.ts):

```ts
const slaProgress = elapsedHours / slaHours;   // slaHours = DEFAULT_SLA_HOURS (default 72h)
// Determine priority based on priority_score thresholds
const priorityScore = slaProgress;              // SLA clock progress IS the priority score
```

`slaProgress` (a pure time ratio) is assigned directly to `priorityScore`. The same variable is then used to determine `Priority`:

```ts
// L46–54
let priority: Priority = Priority.NEW;
if (priorityScore >= this.urgencyThresholds[2]) {      // >= 1.0 (PRIORITY_THIRD_HALF)
  priority = Priority.BREACH;
} else if (priorityScore >= this.urgencyThresholds[1]) { // >= 0.66 (PRIORITY_SECOND_HALF)
  priority = Priority.CRITICAL;
} else if (priorityScore >= this.urgencyThresholds[0]) { // >= 0.33 (PRIORITY_FIRST_HALF)
  priority = Priority.URGENT;
} else {
  priority = Priority.NEW;
}
```

Then both the alert and the associated case are overwritten:

```ts
// L57–72
await this.prisma.alert.update({
  where: { alert_id: alert.alert_id },
  data: { priority, priority_score: priorityScore },
});
if (alert.case_id) {
  await this.prisma.case.update({
    where: { case_id: alert.case_id },
    data: { priority },
  });
}
```

This runs on the cron schedule defined in [alert-priority.task.ts:18](../case-management-system/backend/src/modules/alert-priority/alert-priority.task.ts):

```ts
const expression = this.configService.get<string>('ALERT_PRIORITY_CRON_SCHEDULE') ?? '0 * * * *';
```

Default: **every hour at minute 0**. Every tick, `case.priority` is overwritten by the SLA clock — irrespective of what a human set.

### The Shared Mapper — `case-priority.util.ts`

[backend/src/modules/shared/utils/case-priority.util.ts:9–25](../case-management-system/backend/src/modules/shared/utils/case-priority.util.ts) defines the same mapping used at creation time:

```ts
determinePriority(priorityScore: number): Priority {
  const urgencyThresholds = [
    parseFloat(this.configService.get<string>('PRIORITY_FIRST_HALF', '0.33')),
    parseFloat(this.configService.get<string>('PRIORITY_SECOND_HALF', '0.66')),
    parseFloat(this.configService.get<string>('PRIORITY_THIRD_HALF', '1.0')),
  ];
  if (priorityScore >= urgencyThresholds[2]) return Priority.BREACH;
  else if (priorityScore >= urgencyThresholds[1]) return Priority.CRITICAL;
  else if (priorityScore >= urgencyThresholds[0]) return Priority.URGENT;
  else return Priority.NEW;
}
```

At case creation, `priorityScore` is sourced from the Tazama scoring engine (the typology/rule result). That score is a genuine severity signal. The problem is that the cron job feeds a time ratio into the **same function** via `alert-priority.service.ts`, producing the same enum values from a completely different input. The function cannot know which kind of `0..1` value it is receiving.

### The Schema — `schema.prisma`

```prisma
// L84
Alert.priority   Priority?  @default(NEW)

// L108
Case.priority    Priority   @default(NEW)

// L135–136
Case.sla_deadline      DateTime?   @db.Timestamp(6)
Case.sla_duration_hours Int?

// L146
@@index([sla_deadline])

// L241–245
enum Priority { NEW  URGENT  CRITICAL  BREACH }
```

The schema already has `sla_deadline` and `sla_duration_hours` on `Case` — the infrastructure for the deadline-based model exists but is **never populated or read** by the current cron job. The job uses the hardcoded `DEFAULT_SLA_HOURS` env var instead.

### Priority Set at Creation — Three Code Paths

`CasePriorityUtil.determinePriority` is called in three places at case creation, all passing the Tazama risk score (a genuine severity signal):

| File | Line | Context |
|---|---|---|
| `triage.service.ts` | L302 | Alert triage → case creation |
| `case-creation.service.ts` | L136 | Manual case creation |
| `case-creation-approval.service.ts` | L62 | Supervisor case creation approval |

The initial priority is correctly set from the risk score. The hourly cron then **overwrites it** with the SLA clock value.

### Priority Read Throughout the System

The Priority enum permeates the system:

**Backend (non-test files):**

| File | Usage |
|---|---|
| `report.service.ts:64` | `where.priority = filters.priority` in case queries |
| `report.service.ts:471–491` | `Priority.NEW → "Low"`, `Priority.CRITICAL/URGENT → "Medium"`, `Priority.BREACH → "High"` bucket mapping for workload reports |
| `alert.statistics.service.ts:78` | `whereClause.priority = priority.toUpperCase()` |
| `case-query.service.ts:86,96,329` | Priority filter on owned cases, task-assigned cases, base case filters |
| `case.service.ts:966` | `casePriority` passed to notification payloads |
| `triage.service.ts:429,451,541,599,642,664` | Priority passed in notification payloads at triage |
| `notification.constants.ts:174,237,295` | HTML email templates render `data.casePriority` with colour |
| `notification.service.ts:226,249` | Priority in SLA warning/breach notification data |

**Frontend (non-test files — ~55 files reference `Priority`):**

| File | Usage |
|---|---|
| `casesTable.utils.ts:62–69` | Badge colours: `NEW=gray`, `URGENT=yellow`, `CRITICAL=orange`, `BREACH=red` |
| `CaseFilters.tsx:87–90` | Filter dropdown: `NEW, URGENT, CRITICAL, BREACH` |
| `report.service.ts` priority filter | Reports filters for workload/ageing |

**Three competing vocabularies already exist in the tree** (confirmed by the issue author's blast-radius assessment):
1. `NEW / URGENT / CRITICAL / BREACH` — the broken enum (Prisma schema + most code)
2. `Low / Medium / High` — the semantic labels in `report.service.ts` workload bucketing
3. `NORMAL` — the fallback string in `notification.constants.ts:174` (`data.casePriority ?? 'NORMAL'`)

None of these three agree. The reports already silently translate the broken enum into a 3-bucket human label, which accidentally anticipates the correct design.

---

## Root Cause Analysis

The defect has one sentence: **the hourly cron job uses SLA elapsed time as the input to `determinePriority()` and overwrites `case.priority` with the result**.

Every other problem flows from this:

1. **Priority is not stable.** Any manual or triage-set priority is overwritten within the hour.
2. **Priority cannot encode severity.** A `HIGH` financial-fraud case that has been open for 5 minutes is `NEW`. One that has been open for 50 hours is `BREACH`, regardless of its risk score.
3. **SLA progress is not visible to operators.** Since the priority field encodes the SLA ladder, there is no separate, derivable SLA state. The case list cannot show "HIGH severity, on track" — it can only show `BREACH`.
4. **Filtering is broken.** Filtering by `BREACH` returns all cases past 72h, not all high-risk cases. Filtering by `NEW` returns all recently-opened cases, not all low-risk cases.
5. **Reports are wrong.** The workload report in `report.service.ts:470–491` maps `Priority.BREACH → "High"` for workload counting. A 73-hour `LOW`-risk case shows as a "High priority" case in the report.
6. **SLA notifications have no stable trigger.** `TASK_SLA_WARNING` and `TASK_SLA_BREACH` are sent via `notification.service.ts`, but the idempotency guarantee is fragile because the state that triggers them (`priority`) can move backwards (e.g. if `DEFAULT_SLA_HOURS` changes) or is re-sent every tick without deduplication.

---

## The Settled Design (from Issue Body)

The issue author's design separates the field into two orthogonal axes:

### Axis 1 — Priority (severity, stable)

```
enum Priority { LOW, MEDIUM, HIGH }
```

- Set once at triage from `priorityScore`.
- Never written by the SLA job.
- Mutable only through an explicit, audited `PriorityChanged` event (supervisor/admin action).
- On priority change: `sla_due_at := created_at + slaTarget(tenant, priority_new)` — re-anchored on `created_at`, not `now` (non-gameable).

### Axis 2 — SLA State (derived, never stored)

```
sla_state ∈ { ON_TRACK, AT_RISK, DUE_SOON, BREACHED }
```

- Derived on read from `now vs sla_due_at`.
- Never stored as canonical state — cannot go stale.
- Four states because two distinct escalation triggers exist: an unclaimed `AT_RISK` case (claim-chase) and an owned `DUE_SOON` case (support-chase).

### Priority Calibrates the Clock

Priority drives the SLA budget: `HIGH → 24h`, `MEDIUM → 72h`, `LOW → 168h` (tenant-configurable). Breakpoint percentages (`AT_RISK at 50%`, `DUE_SOON at 80%`) are shared across priorities in MVP; the function signature already keys them per-priority for cheap future divergence.

### The SLA Job's Surviving Role

The cron job stops writing any canonical state. Its only job becomes: derive `sla_state` per open case and fire idempotent escalation notifications keyed on `(case_id, sla_state)` pairs — never re-notifying the same state twice.

---

## Blast Radius — Full File Inventory

### Core (must change — defines the broken semantics)

| File | Change Required |
|---|---|
| `backend/prisma/schema.prisma:241–245` | Rename `Priority` enum: `NEW→LOW, URGENT→MEDIUM, CRITICAL→MEDIUM, BREACH→HIGH`; add `sla_due_at DateTime`; add `SlaPolicy` tenant model; remove `sla_deadline`, `sla_duration_hours` (or repurpose) |
| `backend/src/modules/shared/utils/case-priority.util.ts` | `determinePriority` → 3-bucket severity mapper (`LOW/MEDIUM/HIGH`) |
| `backend/src/modules/alert-priority/alert-priority.service.ts` | Stop writing `priority`; derive `sla_state` from `sla_due_at`; fire idempotent escalation notifications |
| `backend/src/modules/alert-priority/alert-priority.task.ts` | No logic change; triggers the updated service |

### Backend — Priority Reads/Filters (update enum references)

| File | Lines | Change |
|---|---|---|
| `backend/src/modules/report/report.service.ts` | L3, L64, L471–491 | Update import; remap `LOW/MEDIUM/HIGH` workload buckets (the current 3-bucket translation is accidentally correct, just using wrong source values) |
| `backend/src/modules/alert/alert.statistics.service.ts` | L78 | Update priority cast |
| `backend/src/modules/case/services/case-query.service.ts` | L86, L96, L329 | Update priority filter types |
| `backend/src/modules/case/case.service.ts` | L966 | Notification payload type |
| `backend/src/modules/triage/triage.service.ts` | L302–303, L429, L451, L541, L599, L642, L664 | New enum values in calls + notifications |
| `backend/src/modules/case/services/case-creation.service.ts` | L136 | `determinePriority` call (input unchanged, output enum values change) |
| `backend/src/modules/case/services/case-creation-approval.service.ts` | L62, L196, L205, L413 | Same as above |
| `backend/src/constants/notification.constants.ts` | L174, L237, L295 | Update colour mapping from `NEW/URGENT/CRITICAL/BREACH` to `LOW/MEDIUM/HIGH` |
| `backend/src/utils/interfaces/notification.interface.ts` | L15–16 | `TASK_SLA_WARNING / TASK_SLA_BREACH` notification types (rename if needed) |
| `backend/src/modules/notification/notification.service.ts` | L78, L82, L178, L189, L195, L226–250 | Update notification payloads; add `sla_state` field |

### Backend — New Code Required

| Component | Description |
|---|---|
| `SlaPolicy` Prisma model | Tenant-keyed SLA targets per priority (`target_hours`, `at_risk_pct`, `due_soon_pct`); falls back to system defaults |
| `sla-state.util.ts` | `determineSlaState(elapsedHours, priority, slaPolicy) → SlaState` pure function |
| `SlaEscalationRecord` table / in-memory cache | Tracks `(case_id, sla_state)` pairs for idempotent notification deduplication |
| `PriorityChanged` domain event | Records actor, timestamp, old/new priority, reason; triggers `sla_due_at` re-stamp |
| Priority actuator endpoint | Supervisor/admin action to change priority mid-lifecycle with audit |

### Frontend (~55 files — enum label update only for most)

| File | Change |
|---|---|
| `casesTable.utils.ts:62–69` | Badge colours: `LOW=gray, MEDIUM=yellow, HIGH=orange/red` |
| `CaseFilters.tsx:87–90` | Filter options: `LOW, MEDIUM, HIGH` |
| All other priority display components | Rename enum values; add `sla_state` display alongside priority |
| Dashboard / reports filters | Update priority vocabulary; add SLA state filter |

### Database Migration

| Step | Description |
|---|---|
| Schema migration | Rename `Priority` enum values (Postgres `ALTER TYPE` or full rename); add `sla_due_at` column; add `SlaPolicy` table |
| Data backfill | Compute `sla_due_at = created_at + defaultSlaHours` for all open cases; map old enum to new: `NEW → LOW, URGENT → MEDIUM, CRITICAL → MEDIUM, BREACH → HIGH` |
| Validate | Confirm `BREACH` cases get `sla_due_at` in the past (correctly `BREACHED`); confirm `NEW` cases get appropriate `sla_due_at` |

---

## Side-Effect Map

| What breaks if left unfixed | Impact |
|---|---|
| Priority filter in case list returns wrong cases | Investigators filter for "urgent" and get a time-ordered list, not a severity-ordered list |
| Workload report `HIGH priority` count | Reports show `BREACH` (old SLA bucket) as "High" — inflated/deflated depending on SLA age vs risk score |
| Any manually-set priority | Overwritten within one cron cycle (≤1h) |
| SLA notifications | No deduplication; a case near the threshold gets the same notification re-fired every tick |
| Audit trail | No `PriorityChanged` events → cannot reconstruct who escalated a case and when |
| FRAUD_AND_AML container case | Priority is driven by the alert's SLA clock — the container case (issue #214) is in `STATUS_84_COMPLETED` and excluded from the cron's `WHERE` clause on open alerts, so its priority is frozen at whatever the hourly job last computed |

---

## Effort Assessment

This is a **medium-to-large refactor** — the core logic change is small (3 files), but the blast radius across ~42 backend files and ~55 frontend files requires systematic enum-value updates and a non-trivial DB migration.

### Track A — Stop the Bleeding (2–3 days)

Minimum viable fix: stop the cron from overwriting `priority`.

1. **`alert-priority.service.ts`** — Remove the `case.update({ priority })` write. The cron still updates `Alert.priority` and `priority_score` for alert-level display, but stops touching `case.priority`. *(1h)*
2. **`case-priority.util.ts`** — No change yet; it still returns the old 4-value enum at creation, but at least the cron no longer clobbers it. *(0h)*
3. **Test** — Verify a case that has been open for 50 hours no longer shows `BREACH` if it was triaged as low-risk. *(2–4h)*

**Outcome:** Priority is now stable. It may not be meaningful (still maps risk score to the old `NEW/URGENT/CRITICAL/BREACH` ladder), but it is no longer overwritten. Investigators can trust the field.

**Caveat:** This leaves `sla_state` invisible and SLA notifications broken (no deduplication). It is a safe holding position while Track B is prepared.

### Track B — Full Design Implementation (5–8 days)

Implements the settled design from the issue body exactly.

| Step | Files | Effort |
|---|---|---|
| B1: Schema migration | `schema.prisma`, DB migration script | 1 day |
| B2: `SlaPolicy` table + seed | New migration + seed data | 0.5 day |
| B3: `sla-state.util.ts` pure function | New file | 0.5 day |
| B4: Rewrite `case-priority.util.ts` → 3-bucket | 1 file | 2h |
| B5: Rewrite `alert-priority.service.ts` → derive SLA state, fire escalations, stop writing priority | 1 file | 1 day |
| B6: Priority actuator + `PriorityChanged` event | New endpoint + event | 1 day |
| B7: Backend enum sweep (42 files) | Update import references + remap report buckets | 1 day |
| B8: Frontend enum sweep (~55 files) | Update badge colours, filter options, display | 1 day |
| B9: Data backfill + validation | Migration + QA | 0.5 day |

**Total Track B:** ~6–7 working days (with Track A already shipped, so no blocking dependency).

### Recommended Sequencing

Given the 3-day UAT deadline:

1. **Ship Track A today** (2–3h). Stops the priority overwrite. Can be deployed independently.
2. **Schedule Track B for post-deadline.** It is a significant refactor touching every layer; rushing it risks regressions.
3. The issue's design document is already complete — Track B is a well-specified implementation task, not a design exercise.

---

## Acceptance Criteria

### Track A

- [ ] A case opened more than 24h ago does not have its `priority` changed by the background cron job.
- [ ] A priority set by a supervisor during triage persists until explicitly changed.
- [ ] `alert.priority` and `priority_score` continue to update correctly (SLA tracking for alert-level display).

### Track B

- [ ] `Priority` enum is `LOW / MEDIUM / HIGH` in schema and across all code.
- [ ] `case.priority` is never written by the SLA recalculation job.
- [ ] `sla_due_at` is populated for all cases at creation time.
- [ ] `sla_state` is derivable from `now vs sla_due_at` without a DB read on the field itself.
- [ ] Changing `priority` via the actuator endpoint re-anchors `sla_due_at` on `created_at + target(new_priority)`.
- [ ] Priority change is recorded as a `PriorityChanged` audit event with actor, timestamp, old/new values, and reason.
- [ ] Escalation notifications are not re-fired for a `(case_id, sla_state)` pair that has already been notified.
- [ ] The workload report `HIGH / MEDIUM / LOW` priority buckets correctly reflect case severity, not SLA age.
- [ ] Filter by `HIGH` priority returns high-risk cases regardless of how long they have been open.
- [ ] All existing tests pass; new unit tests cover `determineSlaState()` pure function across all four states.

---
