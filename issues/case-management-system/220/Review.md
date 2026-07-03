# Code Review — `paysys/SLA-task` branch vs Issue #220

**Reviewer:** Ahmad Khalid
**Review Date:** 2026-07-03
**Branch:** [paysys/SLA-task](https://github.com/tazama-lf/case-management-system/commits/paysys/SLA-task/)
**Original Issue:** [#220 — Divorce SLA management from prioritisation beyond creation](https://github.com/tazama-lf/case-management-system/issues/220)
**Author of the changes:** Ibad Khan

---

## Overall Verdict

**Strong implementation — ~90% aligned with the settled design, with a few minor deviations.**

The 7 SLA-task commits implement all 9 user stories in a well-sequenced way. Migration approach is sound, the derived `sla_state` model is correctly implemented, and idempotency uses a persistent DB ledger (superior to the in-memory Map suggested in `solution-220.md`).

---

## Commit-by-Commit Assessment

### ✅ [`53f30e27` — Track A: stop the cron overwrite](https://github.com/tazama-lf/case-management-system/commit/53f30e27)

- **Achieves the intent** (US-220-01): the cron no longer writes `case.priority`.
- **⚠️ Style issue:** Instead of deleting the 9 lines (or the whole method as the solution doc says), the entire `runRecalculation()` method was **commented out**, and the caller in `alert-priority.task.ts` was also commented rather than removed. This is dead code that Track B fully replaces two commits later — so acceptable *because* it was superseded quickly, but not ideal as a standalone Track A.

### ✅ [`8d161748` — Enum rename LOW/MEDIUM/HIGH](https://github.com/tazama-lf/case-management-system/commit/8d161748)

- **US-220-02 fully delivered.** Two-phase Postgres migration is textbook-correct: rename `NEW → LOW`, `URGENT → MEDIUM`, `CRITICAL → HIGH` in phase 1, backfill `BREACH` rows to `HIGH` via `UPDATE`, then rebuild-drop the enum in phase 2. Handles both `cases` and `alerts` and resets the `DEFAULT` literal. Backend/test sweep is thorough (~24 files).
- **⚠️ Deviation from solution doc:** `solution-220.md` maps `CRITICAL → MEDIUM`; the branch maps `CRITICAL → HIGH`. This actually matches `user-stories-220.md` (US-220-02: "`CRITICAL` → `HIGH`") and is arguably safer (no severity downgrade for historical `CRITICAL` cases). Worth noting only for the internal doc inconsistency.
- **⚠️ Minor:** Removed `ConfigService` from `CasePriorityUtil` and hard-coded thresholds (`0.7` / `0.4`) rather than making them configurable via renamed env vars. Solution doc suggested keeping them tenant-configurable.

### ✅ [`c51f02b1` — sla_due_at + SlaPolicy](https://github.com/tazama-lf/case-management-system/commit/c51f02b1)

- **US-220-03 delivered.** `SlaPolicy` table + `DEFAULT` tenant seed (`24/72/168h`) + `sla_due_at` column + open-case backfill with the conservative 72h default — all correct.
- **Elegant design choice:** `SlaPolicyUtil.calculateSlaDueAt()` is centralised in `CaseRepository.updateCase()` — any priority update automatically re-stamps `sla_due_at` anchored on `created_at`. This means the actuator (added in a later commit) gets re-anchoring for free without ceremony. Non-gameable, per the spec.
- **⚠️ Missing:** `at_risk_pct` / `due_soon_pct` columns on `SlaPolicy` were dropped from the design. This works because the ratios were moved to constants in `sla-state.util.ts` (`0.2` / `0.5`), but it removes tenant-level tunability — a documented spec item.

### ✅ [`1c4d363a` — Derived sla_state + rewritten cron + idempotent escalations](https://github.com/tazama-lf/case-management-system/commit/1c4d363a)

- **US-220-04 + US-220-05 delivered together.** Pure `determineSlaState()` function keyed on remaining ratio (not elapsed ratio) — clean, tested; `sla_state` is never persisted on `Case`.
- **Idempotency is done correctly:** `SlaEscalationRecord` table with `UNIQUE(case_id, sla_state)`. Check-before-send + insert-after-send + `P2002` concurrent-write handling. Persistent across restarts — this **exceeds** the in-memory Map suggested in `solution-220.md`.
- **Notification-then-record ordering is correct:** if notification fails, no ledger row is written, so the next cron tick retries. Well-reasoned.
- **⚠️ Minor:** The cron only fires for `AT_RISK` (unclaimed) and `DUE_SOON` (claimed) — the `BREACHED` escalation path from `solution-220.md` and the `case.sla-breached` event are not emitted. If a `BREACHED` notification is expected by ops, this is a gap.
- **⚠️ Minor:** The `alert-priority.task.ts` file was fully deleted in favour of `@Cron(CronExpression.EVERY_HOUR)` on the service. Fine, but the `ALERT_PRIORITY_CRON_SCHEDULE` env var override no longer works.

### ✅ [`8ee8d34f` — Priority actuator + PriorityChanged audit](https://github.com/tazama-lf/case-management-system/commit/8ee8d34f)

- **US-220-06 delivered.** `PATCH /cases/:caseId/priority`, `SUPERVISOR`-only, emits `CasePriorityChangedEvent`, `@Audit()` decorated, wired to the audit-log interceptor.
- **⚠️ Auth issue:** Uses `@RequireAuthenticated()` and checks the role manually in the service (`if (actorRole !== 'SUPERVISOR') throw ForbiddenException`). Every other endpoint on this controller uses `@RequireSupervisorRole()` decorator. Inconsistent, and moving the check to the service means bad tests/mocks could bypass — decorator-level enforcement is safer and idiomatic here.
- **⚠️ Role string comparison** uses `!== 'SUPERVISOR'` — no enum constant.

### ✅ [`e290549c` — Frontend enum sweep + SlaStateBadge](https://github.com/tazama-lf/case-management-system/commit/e290549c)

- **US-220-07 delivered.** All ~40 frontend files swept. New `SlaStateBadge.tsx` component added and rendered alongside priority in cases table + case details tab. Filter dropdowns updated to `LOW/MEDIUM/HIGH`. Test fixtures updated. Reasonable scope.

### ✅ [`80e64788` — Reports fix](https://github.com/tazama-lf/case-management-system/commit/80e64788)

- **US-220-08 delivered.** Workload bucket predicates simplified to `priority === Priority.LOW/MEDIUM/HIGH`.
- **⚠️ Behaviour change worth noting:** The commit **removed** the `!CLOSED_STATUSES.includes(c.status)` filter, so buckets now count closed cases too. Comment says this is intentional (*"counts span all cases in scope, matching totalCases"*). This is a legitimate design choice but a semantic change to the report output — worth confirming with a stakeholder before merge, and worth flagging that it may collide with issue #214's `LEAF_CASE_FILTER` work.

---

## Cross-Cutting Concerns

| # | Concern | Severity |
|---|---|---|
| 1 | Track A commit only comments code; not a clean isolated deliverable | Low (Track B replaces it days later) |
| 2 | `at_risk_pct` / `due_soon_pct` not on `SlaPolicy`, so tenants can't tune ratios | Medium |
| 3 | `BREACHED` notification path not implemented | Medium |
| 4 | Priority actuator role check in service, not via `@RequireSupervisorRole()` decorator | Medium — inconsistent with rest of controller |
| 5 | Env-var configurable cron schedule lost (hard-coded `EVERY_HOUR`) | Low |
| 6 | Report bucket now includes closed cases (implicit scope change) | Medium — needs product sign-off |
| 7 | US-220-09 end-to-end regression is not code; must be executed on staging | Expected |

---

## Recommendation

**Ship after addressing items 3, 4, and 6.** The core defect (SLA time ratio overwriting severity) is fully resolved, migrations are safe (two-phase, backfilled, idempotent), and the derived-state architecture is cleaner than what `solution-220.md` prescribed. The remaining items are polish and one product-decision on report semantics — none block correctness of the primary fix.
