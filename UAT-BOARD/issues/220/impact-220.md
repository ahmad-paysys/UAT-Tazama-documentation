# Impact Study — Issue #220: Divorce SLA Management from Prioritisation

**Issue:** [#220](https://github.com/tazama-lf/case-management-system/issues/220)  
**Study Date:** 2026-06-30  
**Author:** Ahmad Khalid  
**Repository:** tazama-lf/case-management-system

---

## Summary

Issue #220 identifies a design defect where the `Priority` enum encodes two orthogonal concepts — inherent case severity and elapsed SLA time — in a single database field that a background cron job overwrites every hour. The fix separates these into a stable `priority` severity field (`LOW / MEDIUM / HIGH`) and a derived, never-stored `sla_state` (`ON_TRACK / AT_RISK / DUE_SOON / BREACHED`). The fix is delivered in two tracks: Track A (one file, one deletion) stops the cron overwrite immediately; Track B completes the full separation with a schema migration and enum sweep across the entire codebase.

This impact study is based on verification of every affected file against the current codebase state.

---

## Confirmed Root Cause

The defect is confirmed in the codebase as described. In [alert-priority.service.ts](../case-management-system/backend/src/modules/alert-priority/alert-priority.service.ts):

- Line 43: `const slaProgress = elapsedHours / slaHours` — a pure time ratio
- Line 45: `const priorityScore = slaProgress` — SLA time ratio is assigned directly to the priority score input
- Lines 46–54: `priorityScore` is fed into the same `determinePriority` thresholds that case creation uses with a genuine Tazama risk score, producing `NEW / URGENT / CRITICAL / BREACH`
- Lines 65–72: The resulting `priority` value is written to `case.priority` via `prisma.case.update`

This runs on the cron defined in [alert-priority.task.ts:18](../case-management-system/backend/src/modules/alert-priority/alert-priority.task.ts) — default `0 * * * *`, every hour. Every tick, `case.priority` is overwritten.

The schema already has `sla_deadline DateTime?` and `sla_duration_hours Int?` on the `Case` model (schema.prisma lines 135–136, with index at line 146), confirming the deadline-based infrastructure was partially anticipated but never connected to the cron job.

---

## Track A — Stop the Cron Overwrite

### What Changes

**One file, nine lines deleted:**

`backend/src/modules/alert-priority/alert-priority.service.ts` — remove lines 65–72 (the `if (alert.case_id)` block that writes `case.priority`). The alert's own `priority` and `priority_score` fields continue to be updated, which is acceptable as an alert-level urgency indicator.

### Impact

| Dimension | Effect |
|---|---|
| Files changed | 1 |
| Schema migration required | No |
| Frontend changes required | No |
| Downtime required | No |
| Risk of regression | Very low — pure deletion of one DB write |
| Reversibility | Trivially reversible — restore the 9 lines |

**What this fixes immediately:**
- `case.priority` is now written exactly once: at creation or triage via `CasePriorityUtil.determinePriority` with a genuine Tazama risk score
- Any manually-set or triage-set priority persists until the case is closed — it is no longer overwritten within the next cron cycle
- Investigators and supervisors can trust the field

**What this does not fix:**
- The `Priority` enum still uses `NEW / URGENT / CRITICAL / BREACH` vocabulary, which encodes time concepts even if the values are now stable
- There is no visible SLA state — operators cannot see "HIGH severity, on-track for SLA"
- Filtering by `BREACH` still means "was assigned that label at triage" rather than "is past deadline"
- Escalation notifications via `notification.service.ts` have no deduplication and may re-fire unexpectedly

**Track A is safe to ship in isolation.** It creates a stable holding position while Track B is developed.

---

## Track B — Full Separation of Priority and SLA

### What Changes

Track B implements the complete design from the issue: rename the enum, add `sla_due_at` and a `SlaPolicy` table, rewrite the cron job to derive and emit SLA state without writing to `case.priority`, stamp `sla_due_at` at case creation, and add an audited priority actuator endpoint.

### Schema Impact

**`backend/prisma/schema.prisma`:**

| Change | Description |
|---|---|
| Rename `Priority` enum | `NEW → LOW`, `URGENT → MEDIUM`, `CRITICAL → MEDIUM` (collapse), `BREACH → HIGH` |
| `Case.priority` default | `@default(NEW)` → `@default(MEDIUM)` |
| Add `Case.sla_due_at` | New `DateTime?` column with index |
| Add `SlaPolicy` model | New table: `id`, `tenant_id`, `priority`, `target_hours`, `at_risk_pct`, `due_soon_pct`; unique on `[tenant_id, priority]` |
| Retain `sla_deadline`, `sla_duration_hours` | These are task-level SLA fields — leave untouched |

The `CRITICAL → MEDIUM` collapse requires a two-step migration: update all `CRITICAL` rows to `URGENT` before the enum rename, because Postgres cannot rename two values to the same target in a single `ALTER TYPE`.

**Migration script risk:** Medium. The `ALTER TYPE ... RENAME VALUE` approach is in-place and does not require a full table rewrite in Postgres 14+. The data backfill (`sla_due_at = created_at + 72h` for all open cases) is a straightforward `UPDATE`. Closed cases are left with `sla_due_at = NULL`.

### Backend Code Impact

**Core rewrites (3 files):**

| File | Nature of Change |
|---|---|
| `alert-priority.service.ts` | Full rewrite: stop writing priority entirely; derive `sla_state` per open case; emit idempotent event-emitter events keyed on `(case_id, sla_state)` |
| `case-priority.util.ts` | Full rewrite: 4-value enum mapping → 3-bucket severity mapping (`LOW / MEDIUM / HIGH`); rename env vars `PRIORITY_FIRST_HALF → PRIORITY_MEDIUM_THRESHOLD`, `PRIORITY_SECOND_HALF → PRIORITY_HIGH_THRESHOLD`, remove `PRIORITY_THIRD_HALF` |
| `alert-priority.task.ts` | No logic change; triggers the updated service |

**New files (4):**

| File | Description |
|---|---|
| `sla-state.util.ts` | Pure function `determineSlaState(now, slaDueAt, createdAt, policy) → SlaState`; `SlaStateUtil` injectable service that resolves `SlaPolicy` from DB with tenant and system-default fallback |
| `case-priority.service.ts` | Priority actuator: `changePriority(caseId, tenantId, actorId, newPriority, reason)` — re-anchors `sla_due_at` on `created_at`, emits `PriorityChanged` audit event |
| `SlaPolicy` seed | Default rows for `SYSTEM_DEFAULT` tenant: `LOW=168h`, `MEDIUM=72h`, `HIGH=24h` |
| `SlaEscalationRecord` (optional) | Persistent deduplication store if in-memory Map is insufficient across restarts |

**Backend enum sweep (~12 backend files):**

| File | Lines | Change Required |
|---|---|---|
| `report.service.ts` | L3, L64, L470–491 | Update import; remap workload buckets from `NEW/URGENT+CRITICAL/BREACH` to `LOW/MEDIUM/HIGH` |
| `alert.statistics.service.ts` | L78 | Update priority cast (code unchanged; update API docs) |
| `case-query.service.ts` | L86, L96, L329 | Update priority filter types |
| `case.service.ts` | L966 | Notification payload type |
| `triage.service.ts` | L302–303, L429, L451, L541, L599, L642, L664 | New enum values in creation calls + notification payloads |
| `case-creation.service.ts` | L136 | Add `sla_due_at` stamp; inject `SlaStateUtil` |
| `case-creation-approval.service.ts` | L62, L196, L205, L413 | Add `sla_due_at` stamp; inject `SlaStateUtil` |
| `notification.constants.ts` | L174, L237, L295 | Update colour mapping; remove `NORMAL` fallback alias |
| `notification.interface.ts` | L15–16 | Add `CASE_SLA_AT_RISK`, `CASE_SLA_DUE_SOON`, `CASE_SLA_BREACHED` notification types; add `CaseSlaPayload` |
| `notification.service.ts` | L78, L82, L178, L189, L195, L226–250 | Update payloads; add `sla_state` field |
| `case.controller.ts` | New endpoint | `PATCH /cases/:id/priority` gated to `SUPERVISOR` role |
| `alert.service.ts` | Alert creation | Enum default value update |

**Note on `report.service.ts`:** The current workload bucketing at lines 470–491 already maps the broken 4-value enum to 3 human labels (`Low / Medium / High`). This is an accidental anticipation of the correct design. After Track B the mapping becomes one-to-one and semantically correct.

### Frontend Code Impact

The frontend `Priority` const object and related utilities are declared in `frontend/src/features/alerts/types/triage.types.ts`. Renaming the four values there cascades to all components that import and use them. The TypeScript compiler will surface every stale reference as a type error, making the sweep mechanical once the type is updated.

**Key UI files requiring direct changes:**

| File | Lines | Change |
|---|---|---|
| `triage.types.ts` | L3–8 | `{ NEW, URGENT, CRITICAL, BREACH }` → `{ LOW, MEDIUM, HIGH }` |
| `alertTransformers.ts` | L108–135 | Drop `'critical'` severity tier; collapse `mapPriorityToSeverity` and `mapSeverityToPriority` to 3-value |
| `casesTable.utils.ts` | L62–70 | Badge colours: `LOW=blue`, `MEDIUM=yellow`, `HIGH=red` (removes `CRITICAL=orange` tier) |
| `CaseFilters.tsx` | L85–91 | Filter options: `LOW, MEDIUM, HIGH` |
| `ManualTriageModal.tsx` | L193–199, L285–291 | 4-branch conditional colour logic → 3-branch |

**Test fixtures (~30–40 test files):** Replace all `Priority.NEW → Priority.LOW`, `Priority.URGENT → Priority.MEDIUM`, `Priority.CRITICAL → Priority.MEDIUM`, `Priority.BREACH → Priority.HIGH` and their string-literal equivalents. The `alertTransformers.test.ts` `mapSeverityToPriority` test drops the `'critical' → 'BREACH'` case.

**Estimated frontend sweep scope:** ~55 non-test files with priority references; ~30–40 test files with fixture data.

---

## Side Effects and Risks

### Risks of Track A Alone

| Risk | Likelihood | Mitigation |
|---|---|---|
| SLA escalation notifications become unreliable | Medium | Notifications are already unreliable (no deduplication). Track A does not make this worse; it just leaves it unfixed. |
| `alert.priority` continues to reflect SLA age | Expected | Acceptable as alert-level urgency display; it is not used for case filtering |
| `BREACH` badge appears on cases past 72h indefinitely | Expected | Priority is now frozen at triage value; old `BREACH` cases remain `BREACH` until Track B renames the enum |

### Risks of Track B

| Risk | Likelihood | Mitigation |
|---|---|---|
| `CRITICAL → MEDIUM` data collapse is irreversible | Low | Run backfill on staging first; validate zero `CRITICAL` rows before enum rename |
| `sla_due_at` backfill uses 72h default regardless of actual risk | Accepted tradeoff | Issue design explicitly recommends 72h as the safe conservative default for backfill |
| In-memory deduplication lost on process restart | Low | Produces at most one duplicate notification per restart event; acceptable for MVP |
| Frontend type error cascade during sweep | Expected | TypeScript compiler surfaces every stale reference; the sweep is mechanical, not speculative |
| `PRIORITY_THIRD_HALF` env var removed | Low | Deployment config must be updated; no runtime fallback |

### Cross-Issue Dependency

Issue #220 and issue #214 share `report.service.ts`. The Track B change to `report.service.ts` (rename priority buckets from `NEW/URGENT+CRITICAL/BREACH` to `LOW/MEDIUM/HIGH`) must be coordinated with the Track A fix for #214 (which adds a `LEAF_CASE_FILTER` to the same file). If both PRs are open simultaneously, a merge conflict in `report.service.ts` is likely. The recommended sequencing is: merge #214 Track A first (the simpler, lower-risk change), then merge #220 Track B with the already-present `LEAF_CASE_FILTER`.

---

## Effort Estimate

| Track | Files | Effort |
|---|---|---|
| Track A | 1 (9 lines deleted) | 1–2 hours including test |
| Track B — schema + migration | 2 files + migration SQL | 1 day |
| Track B — `SlaPolicy` seed + `sla-state.util.ts` | 2 new files | 0.5 day |
| Track B — `case-priority.util.ts` rewrite | 1 file | 2 hours |
| Track B — `alert-priority.service.ts` rewrite | 1 file | 1 day |
| Track B — priority actuator + `PriorityChanged` event | 2 new files + controller | 1 day |
| Track B — backend enum sweep (~12 files) | 12 files | 1 day |
| Track B — frontend enum sweep (~55 files) | 55+ files | 1 day |
| Track B — data backfill validation | SQL + QA | 0.5 day |
| **Track A total** | | **~2 hours** |
| **Track B total** | | **~6–7 days** |

---

## Acceptance Criteria (Verification Checklist)

### Track A
- [ ] A case that has been open for more than 24 hours does not have its `priority` field changed by the background cron job
- [ ] A priority set by a supervisor during triage persists across multiple cron cycles
- [ ] `alert.priority` and `alert.priority_score` continue to update correctly for alert-level display

### Track B
- [ ] `Priority` enum in schema and all code is `LOW / MEDIUM / HIGH`; no references to `NEW`, `URGENT`, `CRITICAL`, or `BREACH` remain in non-migration code
- [ ] `case.priority` is never written by the SLA recalculation job
- [ ] `sla_due_at` is populated for all cases at creation time
- [ ] Changing `priority` via the actuator endpoint re-anchors `sla_due_at` on `created_at + target(new_priority)` — not on `now`
- [ ] Priority change is recorded as a `PriorityChanged` audit event with actor, timestamp, old/new values, and reason
- [ ] Escalation notifications are not re-fired for a `(case_id, sla_state)` pair that has already been notified in the current process lifetime
- [ ] Workload report `HIGH / MEDIUM / LOW` priority buckets correctly reflect case severity, not SLA age
- [ ] Filter by `HIGH` returns high-risk cases regardless of how long they have been open
- [ ] All open cases have `sla_due_at` populated after migration
- [ ] Zero cases or alerts have a `priority` value outside `LOW / MEDIUM / HIGH` after migration

---

## Recommended Sequencing

1. **Ship Track A now** — one PR, one file, no migration, no frontend work. Stops priority instability immediately.
2. **Merge #214 Track A** before opening #220 Track B to avoid a merge conflict in `report.service.ts`.
3. **Ship Track B as a coordinated release** — schema migration, backend sweep, frontend sweep, and data backfill in one branch. Do not split Track B into sub-PRs; the enum rename makes every partial state uncompilable.
