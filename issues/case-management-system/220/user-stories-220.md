# Issue #220 — User Stories: Divorce SLA Management from Prioritisation Beyond Creation

**Assigned developer:** 1 developer  
**Tracks:** Track A (quick fix) → Track B (full separation)  
**Merge order:** Track A ships independently. Track B follows after Issue #214 Track A is merged.

---

## US-220-01 — Stop the Cron Job from Overwriting Case Priority

**Title:** Stop hourly cron from silently overwriting investigator-assigned case priority

**Body:**  
Currently, a background cron job runs every hour and overwrites `case.priority` using a formula based purely on elapsed time divided by SLA duration. This means a supervisor who escalates a case to CRITICAL will have that escalation silently undone within the hour. Investigators and supervisors cannot trust the priority field, and any manually-set priority is effectively meaningless.

The fix is to remove the 9 lines in `alert-priority.service.ts` (lines 43–72) that write `case.priority`. After this change, `case.priority` is written exactly once — at triage — from the real Tazama risk score. The cron job continues running but no longer touches case priority.

**Scope (Track A):**  
- File: `backend/src/modules/alert-priority/alert-priority.service.ts`
- Remove the block that calls `determinePriority(slaProgress)` and writes the result to `case.priority`
- No schema migration, no frontend changes, no other files touched
- The cron schedule itself (`alert-priority.task.ts`) is not changed

**Acceptance Criteria:**
- [ ] A case that has been open for more than 24 hours does NOT have its priority changed by the cron job
- [ ] A priority set manually by a supervisor persists unchanged across multiple cron cycles
- [ ] Alert-level priority and risk score still update as before (alert display is unaffected)
- [ ] No regression in case creation — new cases still receive a priority at triage
- [ ] Unit test confirms the service no longer writes `case.priority`

**Testing:**
- Create a case, note its priority, manually change it via the UI, wait for the cron to tick — confirm priority is unchanged
- Verify alert-level data still refreshes correctly

---

## US-220-02 — Introduce a Stable Three-Bucket Severity Enum

**Title:** Replace the four-value priority enum (NEW/URGENT/CRITICAL/BREACH) with a clean severity enum (LOW/MEDIUM/HIGH)

**Body:**  
The existing `Priority` enum values (`NEW`, `URGENT`, `CRITICAL`, `BREACH`) are time-based names that make no sense as permanent severity labels. `BREACH` especially is an SLA event, not a case severity. This story replaces the enum with three clean severity buckets (`LOW`, `MEDIUM`, `HIGH`) that reflect actual risk, not elapsed time.

The `case-priority.util.ts` file is rewritten so its sole job is mapping a Tazama risk score (0.0–1.0) to `LOW | MEDIUM | HIGH`. The thresholds: ≥ 0.7 → HIGH, ≥ 0.4 → MEDIUM, otherwise → LOW.

A two-phase Prisma migration handles the enum rename safely (PostgreSQL cannot rename two values to the same target in a single `ALTER TYPE`):
- Phase 1: Add `LOW`, `MEDIUM`, `HIGH`; migrate existing data (`NEW` → `LOW`, `URGENT` → `MEDIUM`, `CRITICAL` → `HIGH`, `BREACH` → `HIGH`)
- Phase 2: Drop old enum values

**Scope (Track B — backend enum):**  
- File: `backend/src/modules/alert-priority/case-priority.util.ts` — rewrite as 3-bucket mapper
- Database migration: two-phase enum rename
- Backfill: all existing `BREACH` and `CRITICAL` cases → `HIGH`; `URGENT` → `MEDIUM`; `NEW` → `LOW`
- All backend files that reference the old enum (~12 files) must be swept for import/usage updates — no logic changes, just enum value updates

**Acceptance Criteria:**
- [ ] `determinePriority(score)` returns `LOW | MEDIUM | HIGH` only
- [ ] No `NEW`, `URGENT`, `CRITICAL`, or `BREACH` values exist anywhere in backend code after migration
- [ ] Database migration runs cleanly with zero data loss — every existing case has a valid new priority value
- [ ] Backfill script confirmed: `SELECT count(*) FROM cases WHERE priority IS NULL` → 0
- [ ] All backend tests pass with the new enum

**Testing:**
- Run migration on a staging database copy; verify row counts before and after
- Verify new cases created post-migration receive `LOW`, `MEDIUM`, or `HIGH`
- Confirm no TypeScript compilation errors across ~12 swept files

---

## US-220-03 — Add `sla_due_at` Deadline Stamping at Case Creation

**Title:** Stamp an absolute SLA deadline (`sla_due_at`) on every case at the moment of creation

**Body:**  
Instead of calculating SLA progress at runtime from `created_at` and a config value, each case should carry its own absolute deadline timestamp — `sla_due_at` — stamped at creation. This field is set once, based on the case's priority and the tenant's SLA policy, and is never updated by a cron job.

This makes SLA state a simple comparison: `now > sla_due_at` means breached. The field is re-anchored if a supervisor changes priority, but always calculated from `case.created_at` (not from `now`), so investigators cannot game the deadline by triggering a priority bump.

**Scope (Track B — new field):**  
- Add `sla_due_at DateTime?` column to the `Case` model in `schema.prisma`
- Add `SlaPolicy` seed table (tenant-keyed): `HIGH = 24h`, `MEDIUM = 72h`, `LOW = 168h`, with fallback system defaults
- Update `triage.service.ts`, `case-creation.service.ts`, and `case-creation-approval.service.ts` to stamp `sla_due_at = created_at + slaPolicy.target_hours` at creation
- Backfill script: all existing open cases get `sla_due_at` stamped from their `created_at` + conservative 72h default; closed cases left NULL
- `sla_due_at` is never written by the cron job

**Acceptance Criteria:**
- [ ] Every newly created case has a non-null `sla_due_at` value
- [ ] `sla_due_at` equals `created_at + slaPolicy.target_hours` for the case's tenant and priority
- [ ] `sla_due_at` is not altered by the cron job under any circumstances
- [ ] Backfill: `SELECT count(*) FROM cases WHERE status != 'closed' AND sla_due_at IS NULL` → 0 after migration
- [ ] When a supervisor changes a case's priority, `sla_due_at` is recalculated as `created_at + new_target_hours` (anchored to creation, not now)

**Testing:**
- Create cases for HIGH, MEDIUM, LOW priority tenants and verify the correct deadline is stamped
- Change a case's priority and verify the deadline recalculates from `created_at`, not from the time of change
- Confirm the backfill ran correctly on staging data

---

## US-220-04 — Derive SLA State at Read Time (Never Store It)

**Title:** Compute `sla_state` dynamically from `sla_due_at` — never persist it

**Body:**  
`sla_state` (`ON_TRACK | AT_RISK | DUE_SOON | BREACHED`) is a derived value, not a stored field. It should be computed at read time by comparing `now` to `sla_due_at`. Storing it would mean it becomes stale the moment it is written.

A new pure utility function `determineSlaState(slaDueAt, now)` is created. The cron job calls this for each open case to decide whether to fire a notification. The API and frontend use the same function to display the correct badge.

**Scope (Track B — new utility):**  
- New file: `backend/src/modules/alert-priority/sla-state.util.ts`
  - Pure function: `determineSlaState(slaDueAt: Date, now: Date): SlaState`
  - Thresholds: `BREACHED` if `now >= slaDueAt`, `DUE_SOON` if within 20% of budget, `AT_RISK` if within 50%, otherwise `ON_TRACK`
- Update the cron job (`alert-priority.service.ts`) to call this function per case
- API response DTOs: add computed `sla_state` field when returning case data (not a DB column)
- No Prisma schema change for this story — `sla_state` is never stored

**Acceptance Criteria:**
- [ ] `determineSlaState` is a pure function with no side effects
- [ ] `sla_state` does not exist as a column in the database
- [ ] Case API responses include `sla_state` computed from `sla_due_at` and current server time
- [ ] Unit tests for `determineSlaState` cover all four state transitions with boundary values
- [ ] Cases with `sla_due_at = null` return `sla_state = null` (no crash)

**Testing:**
- Unit test: pass timestamps covering each of the four states; verify correct output
- Integration test: fetch a case via API near its `sla_due_at` boundary and confirm state is correct
- Confirm no `sla_state` column appears in the database schema

---

## US-220-05 — Make SLA Escalation Notifications Idempotent

**Title:** Prevent duplicate SLA escalation emails/notifications — fire at most once per state transition

**Body:**  
Currently the cron job fires SLA notifications on every tick with no deduplication. A case that is `BREACHED` will trigger a notification every hour indefinitely. This story adds idempotency: each `(case_id, sla_state)` pair fires a notification at most once.

This is implemented by keying a deduplication check on `(case_id, sla_state)`. Two escalation triggers exist: an unclaimed case reaching `AT_RISK` (claim-chase notification) and an owned case reaching `DUE_SOON` (support-chase notification). Neither fires again once sent.

**Scope (Track B — cron deduplication):**  
- Update `alert-priority.service.ts` to check whether a notification has already been sent for `(case_id, sla_state)` before firing
- Optional: new `SlaEscalationRecord` table (`case_id`, `sla_state`, `notified_at`) for persistent deduplication across restarts
- Simpler alternative: add `sla_notified_states` JSONB column on `Case` to track which states have been notified
- Update `notification.constants.ts` and `notification.service.ts` for the two new notification payloads (claim-chase, support-chase)

**Acceptance Criteria:**
- [ ] A case that enters `AT_RISK` state triggers a claim-chase notification exactly once — not on every subsequent cron tick
- [ ] A case that enters `DUE_SOON` state triggers a support-chase notification exactly once
- [ ] A case that moves from `DUE_SOON` back to `AT_RISK` (priority upgraded) would fire `AT_RISK` notification again if not previously sent
- [ ] Container restart does not cause re-notification for already-notified states
- [ ] Notification payload includes: case ID, case type, SLA state, time remaining, assignee

**Testing:**
- Run cron twice for a case in `AT_RISK`; verify only one notification sent
- Verify notification payload fields match the spec
- Verify a case that was never notified fires correctly on first tick

---

## US-220-06 — Add Priority Change Audit Trail

**Title:** Record a `PriorityChanged` audit event whenever a supervisor changes a case's priority

**Body:**  
Currently there is no record of who changed a case's priority, when, from what value, to what value, or why. This makes compliance reporting impossible and removes accountability. This story adds an audited priority change endpoint.

A `PriorityChanged` domain event is emitted whenever `case.priority` is updated after triage. The event records: `actor` (user ID), `timestamp`, `old_priority`, `new_priority`, and optional `reason` text.

**Scope (Track B — audit endpoint):**  
- New file: `backend/src/modules/alert-priority/case-priority.service.ts`
  - `changePriority(caseId, newPriority, actorId, reason?)` method
  - Validates actor has supervisor role
  - Updates `case.priority`
  - Recalculates and updates `sla_due_at` anchored to `case.created_at`
  - Emits `PriorityChanged` domain event to audit log
- Wire up the new service to the existing case-priority API endpoint

**Acceptance Criteria:**
- [ ] Every priority change via the supervisor UI creates an audit record with actor, timestamp, old value, new value
- [ ] Non-supervisors cannot call the priority change endpoint (403 returned)
- [ ] Audit records are queryable from the audit log for compliance reports
- [ ] `sla_due_at` is recalculated immediately when priority changes (see US-220-03)
- [ ] Unit test: verify event is emitted with correct payload on priority change

**Testing:**
- Log in as a supervisor, change a case's priority, verify audit record created
- Log in as an investigator, attempt to change priority, verify 403 response
- Verify the audit record payload matches the spec fields

---

## US-220-07 — Update Frontend for New Priority Enum and SLA State Display

**Title:** Update all frontend components to use new `LOW/MEDIUM/HIGH` priority and display `sla_state` badge

**Body:**  
The frontend currently references the old `NEW/URGENT/CRITICAL/BREACH` enum values in approximately 55 files — badge colours, filter dropdowns, type definitions, sort logic, and test fixtures. These all need updating to `LOW/MEDIUM/HIGH`. Additionally, the frontend should now display `sla_state` as a separate badge alongside priority.

**Scope (Track B — frontend sweep):**  
- Update all badge colour definitions: `LOW` → green/blue, `MEDIUM` → yellow/amber, `HIGH` → red/orange
- Update all filter dropdowns that list priority values
- Update TypeScript type definitions (remove `CRITICAL`, remove `BREACH`, add `LOW/MEDIUM/HIGH`)
- Add `sla_state` badge component (ON_TRACK/AT_RISK/DUE_SOON/BREACHED with distinct colours)
- Remove the `CRITICAL` tier (previously mapped to what is now `HIGH`)
- Update ~30–40 test fixtures that use the old enum values
- Do NOT remove any priority filter functionality — just rename the values

**Acceptance Criteria:**
- [ ] No reference to `NEW`, `URGENT`, `CRITICAL`, or `BREACH` remains in any frontend file
- [ ] Priority filter dropdown shows `Low`, `Medium`, `High` options only
- [ ] Case list and detail views show `sla_state` badge independently from priority badge
- [ ] `HIGH` priority cases display in red/orange; `MEDIUM` in yellow; `LOW` in green/blue
- [ ] `BREACHED` SLA state displays in red; `DUE_SOON` in amber; `AT_RISK` in yellow; `ON_TRACK` in green
- [ ] All frontend tests pass with updated fixtures
- [ ] No TypeScript compilation errors

**Testing:**
- Visual review of case list with cases at all three priorities and all four SLA states
- Verify priority filter returns correct results for each value
- Verify no old enum values appear in the browser network tab responses or console errors

---

## US-220-08 — Fix Workload and Priority Reports to Reflect Severity

**Title:** Update workload and priority reports so buckets reflect risk severity, not SLA elapsed time

**Body:**  
The workload report currently uses priority buckets that were being set by the SLA cron job — so "HIGH priority" in a report actually means "cases that have been open a long time," not "high-risk cases." After the enum change, report buckets must be updated to reflect the new `LOW/MEDIUM/HIGH` severity values.

**Scope (Track B — reports):**  
- File: `backend/src/modules/report/report.service.ts`
- Update workload bucket definitions from old enum values to `LOW | MEDIUM | HIGH`
- Update priority distribution query grouping
- Verify that report totals now reflect case risk distribution, not time-in-queue distribution
- Note: this file also has the `LEAF_CASE_FILTER` from Issue #214 Track A — do not remove that filter

**Acceptance Criteria:**
- [ ] The workload report "High Priority" bucket contains cases with `priority = HIGH` (risk-based), not old `CRITICAL/BREACH`
- [ ] Report totals reconcile: `totalCases = low + medium + high`
- [ ] No `NEW`, `URGENT`, `CRITICAL`, or `BREACH` values appear in any report output
- [ ] Existing `LEAF_CASE_FILTER` from Issue #214 is preserved (not accidentally removed)

**Testing:**
- Run the report against a staging dataset; verify bucket totals match direct SQL count per priority
- Confirm the `LEAF_CASE_FILTER` is still present in the generated queries

---

## US-220-09 — End-to-End Integration Testing for Issue #220

**Title:** Full regression and integration test pass for Issue #220 after all stories are merged

**Body:**  
This story is to be completed after all other US-220 stories are merged and deployed to the staging environment together. It validates the complete system behaviour end-to-end, covering all changed surfaces from both Track A and Track B.

**Scope:**  
This is a testing-only story — no code changes.

**Test scenarios to cover:**

*Priority integrity:*
- Create a new case; confirm priority is set from risk score, not from SLA time
- Supervisor changes priority to HIGH; wait for cron to run; confirm priority is still HIGH
- Investigator attempts to change priority; confirm 403 error

*SLA deadlines:*
- Create cases for HIGH, MEDIUM, LOW priority; verify `sla_due_at` values match policy (24h, 72h, 168h)
- Advance the clock past `sla_due_at` for a case; verify `sla_state` returns `BREACHED`
- Change priority; verify `sla_due_at` recalculates from `created_at`, not from time of change

*Notifications:*
- Run cron for a case in `AT_RISK`; verify exactly one notification sent
- Run cron again; verify no duplicate notification

*Audit trail:*
- Change priority three times; verify three audit records with correct actor, old/new values

*Reports:*
- Verify workload report buckets reflect risk severity
- Verify no old enum values (`NEW`, `URGENT`, `CRITICAL`, `BREACH`) appear anywhere in the UI or API responses

*Frontend:*
- Verify priority badges show `Low`, `Medium`, `High` with correct colours
- Verify `sla_state` badge shows separately and updates correctly
- Verify filter dropdowns list only `Low`, `Medium`, `High`

**Acceptance Criteria:**
- [ ] All test scenarios above pass on staging
- [ ] No TypeScript errors in the frontend build
- [ ] Database contains zero rows with old enum values
- [ ] No regression in case creation, triage, or investigator workflow
- [ ] Performance: cron job completes in under 60 seconds for 10,000 open cases
