# Testing Report

## Case Management System — Investigation Model, SLA/Priority Overhaul & BIAR Alert Enrichment

- **Prepared for:** Product Owner
- **Date:** 13 July 2026
- **Development scope:** paysys-pmo tracker issues **#814** and **#823**, plus BIAR alert-enrichment work
- **Underlying CMS issues:** [CMS #214](https://github.com/tazama-lf/case-management-system/issues/214), [CMS #220](https://github.com/tazama-lf/case-management-system/issues/220)
- **Implementation:** [CMS PR #233](https://github.com/tazama-lf/case-management-system/pull/233), [CMS PR #234](https://github.com/tazama-lf/case-management-system/pull/234), [CMS PR #240](https://github.com/tazama-lf/case-management-system/pull/240), [BIAR PR #107](https://github.com/tazama-lf/biar/pull/107), [BIAR PR #118](https://github.com/tazama-lf/biar/pull/118)

---

## 1. Executive Summary

This report defines the **User Acceptance Testing (UAT)** scope for three related improvements to the Case Management System (CMS) that were developed in parallel and are being released together. All three are visible in the CMS web application and, downstream, in the BIAR data-lakehouse dashboards.

| Track | Business Problem | What Changed |
|-------|------------------|--------------|
| **Track A — Investigation Modelling** (paysys-pmo #814 / CMS #214) | A single FRAUD_AND_AML alert generated **three** cases in the system — one synthetic "container" plus one FRAUD and one AML investigation — causing triple-counted reports, permission bypass, an unnatural "Completed" status, and forced supervisor closures. | The container case has been removed. A FRAUD_AND_AML alert now creates **exactly two cases** (FRAUD and AML), each with its own owner, its own lifecycle, and standard permissions. Reports and totals reconcile correctly. |
| **Track B — SLA / Priority Split** (paysys-pmo #823 / CMS #220) | A single `Priority` field was being silently overwritten every hour by a background job based on case age. Supervisors' manual escalations were reverted within an hour. Priority meant "how old", not "how serious." | Priority (severity) and SLA state (timing) are now two independent, orthogonal fields. Priority is stable (LOW / MEDIUM / HIGH) and only changed via an audited supervisor action. A new SLA State (ON_TRACK / AT_RISK / DUE_SOON / BREACHED) is derived live from the case's deadline. |
| **Track C — BIAR Alert Enrichment** (BIAR PR #107) | Alert visualisations did not consistently expose the **sub-rule references** behind an alert. Investigators had to guess *why* a rule fired. **Band** and **exit-condition reason** were not modelled downstream at all. Investigation notes captured on tasks did not flow to the downstream data lakehouse, and the Alert Navigator lineage for transaction status / ID / amount was inconsistent. | The CMS alert view now displays a **sub-rule reference** for each firing rule. **Band** and **exit-condition reason** are now emitted to the BIAR lakehouse (`alerts_nav_rules`) for downstream analytics — they are not yet rendered in the CMS UI. **Investigation notes** are carried end-to-end into the lakehouse. **Alert Navigator** transaction lineage (status, ID, amount, currency) now enriches correctly from the current transactions Gold schema. |

**Overall status of the tracks**

| Track | Sub-issues Complete | Outstanding |
|-------|--------------------:|-------------|
| paysys-pmo #814 (CMS #214) | 8 / 9 | Only the external Tazama review sub-issue (#834) is open — no development pending. |
| paysys-pmo #823 (CMS #220) | 9 / 10 | Only the external Tazama review sub-issue (#833) is open — no development pending. |
| BIAR Alert Enrichment (BIAR PR #107) | Merged | No sub-issue tracker — delivered as a single BIAR change. |

The system is therefore ready for Product Owner acceptance testing against the criteria in this document.

---

## 2. Business Context (What the Product Owner Needs to Know)

### 2.1 Track A — Why the "container" case had to go

Before this release, the FRAUD_AND_AML alert type behaved as follows:

- An alert with combined FRAUD and AML risk produced **one container case** (typed FRAUD_AND_AML) plus **two child cases** (FRAUD and AML).
- The container had no real owner, no genuine investigation lifecycle, and needed a fabricated "Completed" status (STATUS_84) that sat outside the normal Confirmed / Refuted / Inconclusive outcomes.
- Reports **triple-counted** each FRAUD_AND_AML alert: one alert appeared as three case rows in totals, on the Case Type breakdown chart, and in the investigator workload chart.
- Closing the container required a **supervisor** — an extra manual step after both children were already closed by investigators.
- A **permission bypass** existed: because the container had no real owner, any user in the tenant could open it.
- If a child case was reopened after the container had "Completed", the two rows fell into contradictory states.

**After this release:** the container concept is gone. Grouping between siblings is preserved through a lightweight, invisible-to-users link (Investigation Group). The end-user experience has three visible improvements:

1. Each FRAUD_AND_AML alert produces exactly two cases in every list, filter, and report.
2. The Close-Case flow has one consistent form for every case, offering only the three real outcomes.
3. Reports reconcile — total = open + closed, and closed = confirmed + refuted + inconclusive.

### 2.2 Track B — Why Priority and SLA had to be separated

Before this release, one field (`Priority`) was doing two jobs:

- It should have said "**how serious** is this case?" (a risk judgement, set once at triage).
- Instead an hourly cron job kept **overwriting it** to reflect "**how old** is this case?" (NEW after creation → URGENT past 1/3 SLA → CRITICAL past 2/3 SLA → BREACH past SLA).

The consequences were serious:

- Supervisors could not durably escalate a case — a manual bump to CRITICAL was silently reverted within the hour.
- Filtering by BREACH returned every old case regardless of risk; filtering by NEW returned every young case regardless of risk.
- The "High Priority" bucket on the workload report was populated by **old** cases, not **high-risk** cases.
- Investigators had no visible SLA state independent of priority — no way to see "high severity, on track" vs "high severity, breached".
- No audit trail existed for priority changes.
- SLA notifications could re-fire hourly on the same case.

**After this release:** two orthogonal fields.

| Field | Purpose | Values | Who sets it | Storage |
|-------|---------|--------|-------------|---------|
| **Priority** | Severity — a risk judgement | Low / Medium / High | Auto at triage from risk score; only supervisors can change afterwards, with a mandatory reason and an audit record. | Stored, never mutated by any background job. |
| **SLA State** | Timing — where we are against the deadline | On Track / At Risk / Due Soon / Breached | Derived live from `now` vs the case's deadline. | Not stored — computed at read time so it can never be stale. |

Priority calibrates the SLA clock: HIGH = 24-hour target, MEDIUM = 72-hour, LOW = 168-hour (tenant-configurable). The clock starts when a case reaches **Ready For Assignment** (that moment is recorded as `sla_started_at`). If a supervisor bumps priority mid-investigation, the deadline is recomputed as `sla_started_at + priority-target` — anchored to when the SLA clock started, not to the moment of the change. For cases still in DRAFT / PENDING_CASE_CREATION_APPROVAL, the clock has not yet started, so `sla_due_at` is `null` and a priority change on such a case leaves it `null`. Note that if a case is reopened, the SLA clock is restarted from the reopen moment — so on a reopened case, `sla_started_at` is not the original creation date. This design still prevents the old "old case gamed into on-track by lowering its priority" pattern, because within a single active investigation the anchor is fixed.

### 2.3 Track C — Why alert enrichment matters

Investigators reviewing an alert previously saw *that* a rule fired but not the finer detail behind it. Specifically:

- **Sub-rule references** — a rule can trigger for several underlying reasons (sub-rules). Without seeing which sub-rule fired, the investigator has to reconstruct the "why" manually.
- **Band** — the band captures the qualitative bucket of the rule's evaluation. It gives the investigator a quick "how bad" indicator per rule, independent of the raw score.
- **Exit-condition reasons** — the natural-language rationale for why the rule stopped where it did. Essential for evidence packs and audit review.
- **Investigation notes** — investigators capture notes on tasks during a case. Previously these did not travel into the BIAR lakehouse, so downstream analytics and management dashboards could not surface investigator context.
- **Alert Navigator transaction lineage** — the Alert Navigator view is used to trace an alert back to the underlying transaction (status, ID, amount, currency). Previously this enrichment was inconsistent because it drew from the wrong Gold-schema fields.

**After this release:** the CMS alert view surfaces the **sub-rule reference** on each firing rule. **Band** and **exit-condition reason** are emitted to the BIAR lakehouse (`alerts_nav_rules`) for downstream analytics but are **not yet rendered in the CMS UI** — CMS-side rendering is a follow-up. Investigation notes flow to the lakehouse for downstream reporting, and Alert Navigator lineage is accurate against the current transactions Gold schema.

---

## 3. Acceptance Criteria (Consolidated)

The following acceptance criteria define what must be true for the Product Owner to sign off on this release. Each criterion is written to be verifiable from the frontend by a non-technical tester.

### 3.1 Track A — Investigation Modelling

**A1. Case count and list**
- **Manual triage path:** a FRAUD_AND_AML alert manually triaged by an investigator produces exactly **two** cases (one FRAUD, one AML). Neither has case type `FRAUD_AND_AML`.
- **Reports and dashboards** apply a container-exclusion filter (`NON_CONTAINER_CASE_FILTER`), so aggregate totals, case-type breakdowns, and investigator workload all count each FRAUD_AND_AML alert as two cases.
- **Latent code-level gap (not reachable in production today):** the AI-triage code path in `triage.service.ts` contains a FRAUD_AND_AML branch that, if ever executed with a true-positive prediction of type FRAUD_AND_AML, would leave the originating case with `case_type = FRAUD_AND_AML` and create FRAUD and AML sibling cases (three rows total). `getAllCases` does not apply the container-exclusion filter, so such a row would appear on the Cases dashboard when filtered by `FRAUD_AND_AML`. This branch is **not reachable at runtime** because `predictAlert` hardcodes `isTruePositive: false` and discards the AI model response, so no true-positive AI prediction is produced. The AI/ML triage component is not yet implemented. This is recorded as a latent defect to address **before** AI triage is enabled, not a live bug against this release.

**A2. Close-Case modal**
- On any FRAUD or AML case, the Close-Case modal offers exactly three outcomes: **Closed Confirmed**, **Closed Refuted**, **Closed Inconclusive**.
- The **"Completed"** outcome (STATUS_84) does not appear anywhere in the application.
- No **"Sub-Cases Closure Status"** table appears in the Close-Case modal.
- No supervisor-only **distinct "Close Case (AFTER report)"** button appears. Supervisors do see a "Generate Investigation Report" submit path (`frontend/src/features/cases/components/CloseCaseModal.tsx:233-244`) that drives closure after report approval, while investigators see "Submit for Approval" (`frontend/src/features/cases/components/CloseCaseModal.tsx:221-231`); this is the standard supervisor-approval flow, not a hidden second closure button. A normal investigator can submit closure of FRAUD or AML independently.

**A3. Case Filters screen**
- The Status filter drop-down does **not** contain "Completed" (STATUS_84).
- No STATUS_84 badge colour appears anywhere in the UI (case rows, case detail, dashboards).

**A4. Permissions**
- A user who is neither owner nor assignee of an *owned or task-assigned* FRAUD or AML case receives a **403** on any attempt to access, close, or modify it. Unassigned cases and cases still in `STATUS_02_READY_FOR_ASSIGNMENT` remain visible to investigators by design (`backend/src/modules/case/services/case-query.service.ts:864-873`), matching the dashboard's pickup semantics.
- FRAUD_AND_AML no longer receives any permission special-case.

**A5. Independence of sibling cases**
- Assigning the FRAUD case does not change the AML case.
- Closing the FRAUD case does not change the AML case (and vice versa).
- Suspending or reopening one case has no effect on any other case.

**A6. Reports and dashboards**
- **Guaranteed by code:** Reports apply the container-exclusion filter. A single FRAUD_AND_AML alert contributes **2** cases (not 3) to report totals.
- **Guaranteed by code:** Case Type breakdown shows **FRAUD** and **AML** buckets only — no `FRAUD_AND_AML` bucket.
- **Guaranteed by code:** Investigator Workload does not attribute the (former) ownerless container to any investigator.
- **Observable at runtime (not code-enforced):** Closed Cases should equal Closed Confirmed + Closed Refuted + Closed Inconclusive plus system auto-closures 71/72. Total Cases should equal Open + Closed. Average Resolution Time should not be distorted by former instant container closures. The tester should observe and confirm these identities in the UAT dataset.

### 3.2 Track B — SLA / Priority Split

**B1. Priority values**
- Every Priority drop-down, badge, filter, and export shows only **Low**, **Medium**, **High**.
- The legacy values `NEW`, `URGENT`, `CRITICAL`, and `BREACH` do not appear anywhere as Priority **enum** values. (`BREACHED` still exists in the codebase as an **SLA State** value — a separate concept — which is expected; see B4.)
- **Cosmetic follow-up:** the dashboard tile description helper (`frontend/src/features/dashboard/services/dashboardService.ts:85-99`) still renders legacy adjectives — "Breached cases…" for HIGH and "Critical/Urgent cases…" for MEDIUM — in user-visible tile copy. This is descriptive text, not an enum value, but the vocabulary is inconsistent with the Low/Medium/High rename and should be cleaned up.
- Low / Medium / High badges are visually distinct from each other. Specific colour choices are to be observed by the tester in the UAT environment (not enforced by code-level constants checked in this report).

**B2. Priority stability**
- Priority is writable only by (a) triage at initial case creation and (b) the supervisor endpoint `PATCH /cases/:caseId/priority` guarded by `@RequireSupervisorRole()`. Investigators are explicitly **excluded by design** — the codebase has a `RequireInvestigatorOrSupervisorRole` decorator available, and it was deliberately not applied here (`backend/src/modules/alert-priority/case-priority.service.ts:25` — inline comment documents the supervisor-only intent).
- The SLA escalation cron performs **no** writes to the `case.priority` column — its only mutation is inserting into `SlaEscalationRecord` (`backend/src/modules/alert-priority/alert-priority.service.ts:77-169`). A case's priority therefore does not drift over any elapsed time, regardless of how many cron cycles run.
- New cases still receive a priority automatically at triage, derived from the risk score.

**B3. SLA deadline**
- The SLA clock starts when a case first reaches **STATUS_02 Ready For Assignment** (RFA). That moment is recorded as `sla_started_at`, and the deadline `sla_due_at = sla_started_at + priority-target` is stamped at that time.
- Cases in **DRAFT** or **PENDING_CASE_CREATION_APPROVAL** have `sla_due_at = null` by design (the clock has not started yet).
- Priority-target values: **HIGH = 24 h**, **MEDIUM = 72 h**, **LOW = 168 h** (tenant-configurable via SLA policies).
- If a supervisor changes priority on a case whose SLA clock has already started, the deadline is recomputed as **`sla_started_at + new-priority-target`** — anchored to the moment the SLA clock started (`sla_started_at`), **not** to the case's original `created_at` and **not** to the moment of the change.
- If a supervisor changes priority on a case that has **not yet reached RFA**, `sla_due_at` is left as `null` (nothing to recompute) and no error is thrown.
- If a case is **reopened**, the SLA clock is restarted from the reopen moment — the new `sla_started_at` is the reopen timestamp, not the original creation. A subsequent priority change on a reopened case is therefore anchored to the reopen moment.

**B4. SLA State badge**
- The case list and case detail views display an **SLA State** badge **independently** of the Priority badge.
- Values and colours: **On Track** (green), **At Risk** (yellow), **Due Soon** (amber), **Breached** (red).
- A case without a deadline shows a neutral grey **"N/A"** SLA badge (rendered by `frontend/src/shared/components/ui/SlaStateBadge.tsx:11-13,24`), not a red or amber one, and no error.

**B5. Priority-change permissions and audit**
- The supervisor priority-change endpoint (`PATCH /cases/:caseId/priority`) is implemented and guarded by `@RequireSupervisorRole()` (`backend/src/modules/case/case.controller.ts:649-670`). The endpoint persists via Prisma and re-anchors `sla_due_at` correctly.
- A non-supervisor call is rejected by `TazamaAuthGuard` with **HTTP 401** (`UnauthorizedException` thrown at `backend/src/guards/tazama-auth.guard.ts:50` via `evaluateClaimResult` at lines 163-170). The guard file imports only `UnauthorizedException` — no `ForbiddenException` is thrown anywhere on this path.
- Every successful priority change is captured by the `@Audit()` interceptor, producing an audit-store record with actor, timestamp, and outcome.
- **Critical gap — no frontend affordance:** the CMS frontend has **no UI control** that calls this endpoint. There is no `changePriority` client method on `caseService.ts`, no `ChangePriorityModal`, and no supervisor row-action or menu item on the case list / case detail views (verified by grep across `frontend/src` for `changePriority` / `"Change Priority"` / `PATCH /cases/*/priority` — zero hits). A supervisor **cannot change a case's priority through the product UI today**; the endpoint is reachable only via direct API call (curl / Postman with a `CMS_SUPERVISOR` JWT). Adding the UI surface is a required follow-up before B5 can be exercised as a user-facing capability.
- **Additional gap — orphaned domain event:** `CasePriorityService.changePriority` emits `case.priority.changed`, but a codebase-wide search for `@OnEvent('case.priority.changed')` returns **zero listeners**. Consequently nothing writes to `caseHistory`, and the case-history / task-log tab does not surface priority changes even if the endpoint is invoked directly. Flowable is bypassed entirely — the only BPMN reference to priority is the initial-triage form field (`backend/src/modules/bpmn/cms.bpmn20.xml:59`); no service task, execution listener, or task listener consumes priority changes.

**B6. Notifications**
- Each `(case, SLA-state)` transition fires **at most one** notification, enforced by the unique constraint `(case_id, sla_state)` on the escalation-records ledger. No hourly re-notification on breached cases.
- **Three** distinct notification types are emitted:
  - **`CASE_CLAIM_CHASE`** — case reaches **AT_RISK** while **unclaimed** (nudges someone to claim it).
  - **`CASE_SUPPORT_CHASE`** — case reaches **DUE_SOON** while **owned** (nudges the owner / supervisor).
  - **`CASE_SLA_BREACHED`** — case crosses the deadline.
- **Known gap:** an unclaimed case that reaches **DUE_SOON** does **not** currently trigger a dedicated notification of its own — this transition falls through the escalation logic. Flag as a possible follow-up if the Product Owner requires an unclaimed-Due-Soon nudge.
- The notification payload includes: case ID, case type, SLA state, time remaining, and assignee (where applicable).

**B7. Filtering**
- Filtering by **High** returns high-risk cases regardless of age.
- Filtering by **Low** returns low-risk cases regardless of age.
- The Priority filter drop-down lists exactly: Low, Medium, High.

**B8. Reports**
- The Workload report's High Priority bucket contains cases with Priority = HIGH (a severity measure), **not** old cases.
- Report totals should reconcile: `totalCases ≈ low + medium + high`. Note: `totalCases` is computed as an independent `prisma.case.count` (`backend/src/modules/report/report.service.ts:461`), not as the sum of the Low/Medium/High buckets. `Case.priority` is non-nullable with `@default(LOW)` (`backend/prisma/schema.prisma:109`), so the buckets should partition the total — but because the two counts are issued as independent queries with independent filter derivations, the tester should still observe the identity holds in the UAT dataset.

### 3.3 Track C — BIAR Alert Enrichment

**C1. Sub-rule references on the alert view**
- On any alert whose rules have sub-rules, the alert detail view (or the alert visualisation the investigator uses) displays a **sub-rule reference** for each firing rule.
- Alerts whose rules have no sub-rules render cleanly with the field simply absent (no error, no empty placeholder that misleads).

**C2. Band available per rule (lakehouse only in this release)**
- Each firing rule's **band** value (with the matched band reason where applicable) is emitted to the BIAR lakehouse `alerts_nav_rules` view as part of this release. Verify via a BIAR dashboard or a Jupyter query against `alerts_nav_rules`.
- **Not surfaced in the CMS alert view in this release** — the CMS frontend does not render band values. Rendering in CMS is a follow-up if the Product Owner requires investigator-facing display.

**C3. Exit-condition reason available per rule (lakehouse only in this release)**
- Each firing rule's **exit-condition reason** is emitted to the BIAR lakehouse `alerts_nav_rules` view (`matched_exit_condition_reason` and `exit_condition_reasons_json`). Verify via a BIAR dashboard or a Jupyter query against `alerts_nav_rules`.
- **Not surfaced in the CMS alert view in this release** — the CMS frontend does not render exit-condition reasons. Rendering in CMS is a follow-up if the Product Owner requires investigator-facing display.

**C4. Investigation notes flow to lakehouse**
- Investigation notes entered on a task in the CMS are captured in the lakehouse `investigationNotes` field on the corresponding task record.
- Editing an existing note in the CMS results in the updated content appearing on the next lakehouse refresh cycle.
- Notes are preserved end-to-end without truncation (subject to any documented length limits).

**C5. Alert Navigator transaction lineage is accurate**
- Opening the Alert Navigator (or the CMS view backed by it) for any alert shows the correct **transaction status**, **transaction ID**, **transaction amount**, and **transaction currency** for the underlying transaction.
- These fields match what is recorded on the transaction itself (they no longer draw from a stale or wrong source).
- Instructing / instructed agent details continue to populate as before.

**C6. Backward compatibility**
- `gold/tasks.status` continues to expose the **normalised upper-case** status (backward compatibility with existing dashboards).
- The raw source status is available separately as `status_raw` — dashboards depending on either value continue to work.

---

## 4. Test Environment & Preparation

Before executing the tests below, the tester should:

1. **Environment**: Access the CMS web application on the UAT environment using a modern browser (Chrome / Edge / Firefox current versions). Ensure browser cache is cleared so no stale JavaScript is loaded.
2. **Test accounts required**:
   - **Investigator A** — standard investigator role, assigned to at least one FRAUD case.
   - **Investigator B** — standard investigator role, assigned to at least one AML case.
   - **Supervisor** — a supervisor account for close-case flows and the new priority-change action.
   - **Uninvolved user** — a normal user with no assignment to a specific FRAUD/AML case (used to verify 403 behaviour).
3. **Test data required**:
   - At least one alert of type **FRAUD_AND_AML** (fresh) available for triage.
   - At least two existing FRAUD cases and two AML cases already open.
   - One existing case pre-populated with **Priority = HIGH** and an SLA deadline within 2 hours (to observe SLA state transitions in real time).
   - A historical alert / case created before this release, to verify migrated priority values (old NEW/URGENT/CRITICAL/BREACH values should now show as LOW/MEDIUM/HIGH per migration).
   - **A case that has been reopened at least once** (so its `sla_started_at` reflects the reopen moment, not the original creation) — used to verify that a supervisor priority change on a reopened case anchors correctly to the reopen moment (see Scenario 9).
   - **A case currently in DRAFT / PENDING_CASE_CREATION_APPROVAL** (no SLA deadline yet) — used to verify that a priority change on a pre-RFA case leaves `sla_due_at` as null without error (see Scenario 9).
   - **At least one alert triggered by a rule that carries sub-rules, band, and exit-condition reasons** (for Track C — sub-rule reference, band, and exit-condition reason display).
   - **At least one task on an existing case with an investigation note already recorded** (for Track C — lakehouse propagation check).
4. **BIAR dashboards** (optional cross-service check): access to the BIAR dashboard for confirming lakehouse-side counts. This is not the Product Owner's primary responsibility but is available for cross-verification.

---

## 5. Test Scenarios

Each scenario is a self-contained walkthrough. Steps assume the tester is signed into the CMS web app.

> **Current environment limitations affecting some scenarios**
>
> - **Supervisor "Change Priority" UI is not present in this release.** The `PATCH /cases/:caseId/priority` endpoint is implemented and role-guarded (see B5), but the CMS frontend does not yet expose a control that calls it. Scenarios that depend on a supervisor priority change through the product UI are therefore **Not Applicable (N/A)** for UAT and are marked as such below. Backend behaviour can still be exercised out-of-band via the API (curl / Postman with a `CMS_SUPERVISOR` JWT) if the tester wants to confirm code paths, but this is not part of UAT sign-off.
> - **Notifications are code-complete but not yet configured on the environment side.** The notification-emit code paths for `CASE_CLAIM_CHASE`, `CASE_SUPPORT_CHASE`, and `CASE_SLA_BREACHED` are in place, but the notification channel / delivery configuration has not been wired up on the UAT environment. Any scenario step that expects a notification to *arrive* should be treated as **deferred** — record it as a limitation rather than a defect. SLA-state badge transitions and idempotency ledger entries are still fully observable and must pass.

### Scenario 1 — A new FRAUD_AND_AML alert creates only two cases

**Acceptance criteria covered:** A1, A5

1. Sign in as **Investigator A**.
2. Navigate to **Alerts Dashboard** from the main navigation.
3. Locate a fresh FRAUD_AND_AML alert (or ingest one). Click the alert row to open the **Alert Details** modal.
4. Trigger triage from the modal using the **manual triage flow**.
5. Once triage completes, close the modal and navigate to the **Cases** area.
6. Filter or search by the alert reference / customer.

**Expected result (manual triage path)**
- Exactly **two** case rows appear for this alert — one with case type **FRAUD** and one with case type **AML**.
- Neither case has type `FRAUD_AND_AML`.
- Both cases have an owner assigned.

7. Open the FRAUD case. Under the **Linked Items** (or equivalent) area, confirm the AML sibling is discoverable via the shared investigation group. There is **no** "parent case" panel.

**Note on the AI-triage / auto-completion path**
- The AI-triage (predicted-outcome) path is **not enabled** in this release — the AI/ML component is not implemented and `predictAlert` returns a hardcoded false-positive, so no auto-close or auto-completion via prediction occurs at runtime. Manual triage is the only path that produces cases from a FRAUD_AND_AML alert today. See A1 for the latent code-level gap that must be addressed before AI triage is enabled.

---

### Scenario 2 — Sibling cases are independent

**Acceptance criteria covered:** A5

1. Sign in as **Investigator A**. Open the FRAUD case created in Scenario 1.
2. Assign yourself as owner (or note the current owner).
3. Sign in (in a second browser session) as **Investigator B**. Open the sibling AML case.
4. On the AML case, change the assignee to Investigator B.
5. On the FRAUD case tab, refresh the page.
6. As Investigator A, close the FRAUD case (see Scenario 3).
7. Refresh the AML case as Investigator B.

**Expected result**
- After step 5: the FRAUD case's owner / assignee is unchanged by any action taken on the AML case, and the AML case's owner / assignee is unchanged by any action taken on the FRAUD case.
- After step 7: the AML case's status is unchanged. It remains open and independently closable.

---

### Scenario 3 — Close-Case modal offers only real outcomes

**Acceptance criteria covered:** A2

1. Sign in as **Investigator A**. Open any FRAUD or AML case ready to be closed.
2. Click **Close Case** in the Case Actions panel.
3. Inspect the **Final Outcome** drop-down.
4. Select an outcome, provide required notes, and confirm.

**Expected result**
- At step 3, exactly three options appear in the drop-down: **Closed Confirmed**, **Closed Refuted**, **Closed Inconclusive**.
- **"Completed"** is **not** an option.
- No **"Sub-Cases Closure Status"** table is rendered inside the modal.
- No supervisor-only "Close Case (AFTER report)" secondary button appears.
- After step 4, the case closes to the chosen status. No sibling case is affected. No supervisor approval is required.

---

### Scenario 4 — Filters and dashboards no longer expose the container concept

**Acceptance criteria covered:** A1, A3, A6

1. Sign in as any user with report access.
2. Navigate to **Cases** and open the **Filters** panel.
3. Inspect the **Status** filter drop-down.
4. Inspect the **Case Type** filter. If `FRAUD_AND_AML` is still present as a legacy filter value, select it and apply the filter.
5. Navigate to **Reports → Case Status** (or equivalent).
6. Observe Total Cases against (Open Cases + Closed Cases). Observe Closed Cases against the sum of Closed Confirmed + Closed Refuted + Closed Inconclusive (+ system auto-closures 71/72).

**Expected result**
- At step 3, no **"Completed"** (STATUS_84) option appears in the drop-down.
- At step 4, zero rows are returned in this release (manual triage does not persist a FRAUD_AND_AML case, and the AI-triage path is not enabled). If any rows are returned, they are pre-release historical container cases — record them as a data cleanup item for the Product Owner rather than a functional defect.
- At step 6:
  - **Guaranteed:** No orphan "Completed" bucket exists in any status breakdown.
  - **Guaranteed:** The **Case Type breakdown** chart shows only FRAUD and AML segments — no `FRAUD_AND_AML` segment.
  - **Guaranteed:** The **Investigator Workload** chart contains no unowned "container" entries.
  - **Observable (not code-enforced):** the tester should confirm that `Total = Open + Closed` and `Closed = 81 + 82 + 83 + 71 + 72` balance in the UAT dataset. Note any discrepancy for the Product Owner.

---

### Scenario 5 — Permission enforcement on FRAUD / AML cases

**Acceptance criteria covered:** A4

1. Note the ID of a specific FRAUD case to which the **Uninvolved user** has no ownership or assignment.
2. Sign in as the **Uninvolved user**.
3. Attempt to open the case directly by navigating to its URL, or by any search / listing path that would surface it.

**Expected result**
- Access is denied (a **403** error is shown, or the case is filtered out of the list). No FRAUD_AND_AML permission bypass exists.

---

### Scenario 6 — Priority values everywhere are Low / Medium / High

**Acceptance criteria covered:** B1, B7

1. Navigate to **Cases → Filters**. Open the Priority drop-down.
2. Filter by **High**. Inspect the returned rows.
3. Repeat for **Low** and **Medium**.
4. Navigate to **Alerts Dashboard**. Open its priority filter.
5. Open any case that existed before this release (a historically migrated case).

**Expected result**
- At step 1: exactly three options — **Low**, **Medium**, **High**. No `NEW`, `URGENT`, `CRITICAL`, or `BREACH` value appears.
- At step 2: the rows include recent HIGH-severity cases as well as older HIGH-severity cases. Age is not the filter criterion.
- At step 4: only Low / Medium / High are listed there too. Alert cards display the new colours (Low blue/green, Medium amber, High red/orange).
- At step 5: the priority is displayed as **Low**, **Medium**, or **High** — the migration converted NEW → LOW, URGENT → MEDIUM, CRITICAL → HIGH, BREACH → HIGH.

---

### Scenario 7 — Priority is stable and no longer overwritten by the background job  🚫 **Not Applicable (this release)**

**Acceptance criteria covered:** B2

> **N/A reason:** the supervisor **Change Priority** UI control is not present in this release (see B5 and the Section 5 limitations note). This scenario cannot be executed end-to-end through the product until that control ships. The backend guarantee that priority is no longer overwritten by the SLA cron is still verifiable via a direct API call for a supervisor who has an existing priority value on a case, but that route is out of UAT scope.
>
> *Original steps preserved below for reference; skip during UAT execution.*

1. ~~Sign in as **Supervisor**. Open a case currently at **Priority = Low**.~~
2. ~~Use the **Change Priority** action in the Case Actions panel. Change priority to **High** and provide a reason (e.g., "Manual escalation — new intelligence").~~
3. ~~Confirm the change; verify the badge now shows **High**.~~
4. ~~Wait at least **two hours** (so the SLA cron has run at least twice), or ask a system administrator to trigger the SLA job twice.~~
5. ~~Return to the same case.~~

**Expected result (deferred)**
- Priority remains **High**. No background job has reverted it.
- An audit record exists for the priority change (actor, timestamp, old value, new value, reason). Audit-visibility in the case-history UI is a separate follow-up (see B5).

---

### Scenario 8 — SLA State badge appears and progresses through states

**Acceptance criteria covered:** B3, B4, B6

**Setup**: Ensure there is a HIGH-priority case whose deadline is within the next 2 hours (or use test controls to shorten the deadline).

1. Open the case's **View Case** modal.
2. Leave the case unresolved and let time pass (or use an accelerated test clock, if available in UAT).
3. Refresh the case at intervals corresponding to 1/3, 2/3, and past the deadline.

**Expected result**
- At step 1:
  - A distinct **SLA State** badge is visible in the Case Details tab, separate from the Priority badge.
  - The badge reads **On Track** (green), **At Risk** (yellow), **Due Soon** (amber), or **Breached** (red) depending on the remaining time.
- Across steps 2–3:
  - The SLA State badge transitions **On Track → At Risk → Due Soon → Breached**.
  - The Priority badge remains unchanged throughout (still **High**).
  - No repeated hourly badge changes on any transition (idempotency of the underlying escalation-ledger entry is still observable — see Scenario 11).

**Notification behaviour (deferred — environment limitation)**
- Notification delivery is **not yet configured** on the UAT environment, although the emit code paths are complete. The following outcomes are therefore **deferred** and should be recorded as a known limitation rather than a defect:
  - On **AT_RISK** for an **unclaimed** case: `CASE_CLAIM_CHASE` would fire.
  - On **DUE_SOON** for an **owned** case: `CASE_SUPPORT_CHASE` would fire.
  - On **BREACHED**: `CASE_SLA_BREACHED` would fire.
  - **Known gap:** on **DUE_SOON** for an **unclaimed** case, no dedicated notification is emitted (design gap independent of the environment issue — flag as a possible follow-up).
- Re-verify these once notification channels are configured on UAT.

---

### Scenario 9 — Supervisor changes priority; SLA deadline re-anchors correctly  🚫 **Not Applicable (this release)**

**Acceptance criteria covered:** B3, B5

> **N/A reason:** all four sub-cases below require a supervisor to invoke the **Change Priority** action from the CMS UI. That control is not yet exposed in the frontend (see B5 and the Section 5 limitations note). The backend endpoint (`PATCH /cases/:caseId/priority`) is implemented, role-guarded, and re-anchors `sla_due_at` correctly, and can be exercised out-of-band via API for engineering verification, but this scenario is not executable through the product during UAT.
>
> *Sub-cases preserved below for reference and to be re-enabled once the UI control ships.*

#### 9a — Priority change on a case in Ready For Assignment (never reopened) *(deferred)*

1. ~~Identify a case whose `sla_started_at` is **48 hours** ago (i.e., it reached RFA 48 h ago and has not been reopened since), currently at **Priority = Medium** (deadline = `sla_started_at + 72 h`, so ~24 h remaining, SLA state On Track).~~
2. ~~Sign in as **Supervisor**. Open the case and use **Change Priority** to move to **High**. Provide a reason.~~
3. ~~Refresh the case.~~

**Expected result (deferred)**
- The SLA deadline is recomputed as `sla_started_at + 24 h` (the HIGH target). Because `sla_started_at` was 48 h ago, the recomputed deadline is already in the past.
- The SLA State badge immediately shows **Breached**.
- A single `CASE_SLA_BREACHED` group notification would fire to the supervisors candidate group — **notification delivery is currently unconfigured on the environment** (see Section 5 limitations); the emit code path exists but arrival cannot be confirmed until channels are wired up.
- An audit record for the priority change is created.

#### 9b — Priority change on a reopened case (anchor is the reopen moment, not creation) *(deferred)*

1. ~~Identify or prepare a case that has been **reopened** at some point after its original creation. Note the reopen timestamp — this is the case's current `sla_started_at`.~~
2. ~~Sign in as **Supervisor**. Change priority on this case.~~
3. ~~Refresh the case.~~

**Expected result (deferred)**
- The new deadline = `sla_started_at (reopen moment) + new-priority-target`. It is **not** anchored to the case's original creation date.
- SLA state updates accordingly. If the reopen moment is recent and the priority is bumped to HIGH, the case may still show On Track even though its original creation is old.

#### 9c — Priority change on a case that has never reached RFA (DRAFT / PENDING) *(deferred)*

1. ~~Identify a case still in **DRAFT** or **PENDING_CASE_CREATION_APPROVAL** (no `sla_due_at` yet).~~
2. ~~Sign in as **Supervisor**. Change priority on this case.~~
3. ~~Refresh the case.~~

**Expected result (deferred)**
- The action succeeds. `sla_due_at` remains **null** — nothing to recompute yet. No error, no SLA State badge (or a "not yet started" state, matching B4).
- The audit record is still created.

#### 9d — Non-supervisor is blocked *(deferred)*

1. ~~Attempt to change priority as an **Investigator** (non-supervisor) via the same action on any case.~~

**Expected result (deferred)**
- With no UI control present, there is nothing for an Investigator to click; when the control ships, the button must be hidden for non-supervisors and the underlying API returns **401 Unauthorized** to a non-supervisor JWT (per B5).

---

### Scenario 10 — Reports reconcile and buckets reflect severity, not age

**Acceptance criteria covered:** A6, B7, B8

1. Navigate to **Reports**.
2. Open the **Case Status Report**. Observe:
   - **Runtime check:** Total = Open + Closed. Closed = Confirmed + Refuted + Inconclusive (+ system auto-closures). Note any discrepancy.
3. Open the **Workload Report** (or the priority-breakdown chart).
   - Note the count of the **High Priority** bucket.
   - Drill into the bucket.
4. Verify the priority buckets show the new values (Low / Medium / High) — no legacy NEW / URGENT / CRITICAL / BREACH values in any legend or bar label.

**Expected result**
- **Guaranteed by code:** Every case in the High bucket has Priority = **High** (severity). The bucket is not populated by cases that are simply old.
- **Guaranteed by code:** Priority-bucket totals reconcile: `low + medium + high = totalCases` (all derived from the same underlying query).
- At step 4: only Low / Medium / High values appear across legends and bar labels; no legacy NEW / URGENT / CRITICAL / BREACH values are shown.

---

### Scenario 11 — Notification idempotency (no hourly noise)

**Acceptance criteria covered:** B6

> **Environment limitation:** notification delivery is not yet configured on UAT (see Section 5). The idempotency guarantee lives in the escalation-records ledger (`(case_id, sla_state)` unique constraint), which is still observable — the "inbox" portion of this scenario is deferred.

1. Identify a case that is currently **Breached**.
2. Note the timestamp of the most recent escalation-ledger entry for that case (via BIAR / DB view of `SLA Escalation Records` if accessible, or ask engineering to confirm).
3. Wait for **at least two SLA cron cycles** (typically 2 hours).
4. Re-check the escalation-ledger for the same case.

**Expected result**
- **No additional `CASE_SLA_BREACHED` ledger row** has been written for that same case in the interim. The breach entry is a one-shot per `(case, SLA state)` pair.
- *Deferred:* once notification channels are configured, the assignee/supervisor notifications inbox should likewise show a single breach notification, not repeated hourly copies.

---

### Scenario 12 — Cross-service check (BIAR lakehouse, optional)

**Acceptance criteria covered:** downstream data integrity for A1, A6, B1, B4

If the tester has access to the BIAR dashboard:

1. Open the case-related dashboards.
2. Confirm that the total case count for the test period matches the CMS Reports (no triple-counting of FRAUD_AND_AML alerts).
3. Confirm that priority breakdowns show Low / Medium / High buckets only.
4. Confirm the presence of SLA-related tables in the lakehouse (Investigation Groups, SLA Policies, SLA Escalation Records) — evidence that BIAR is receiving the new schema correctly.

---

### Scenario 13 — Sub-rule reference is visible on the alert (and band / exit-condition reason are queryable in BIAR)

**Acceptance criteria covered:** C1, C2, C3

1. Sign in as **Investigator A**. Navigate to **Alerts Dashboard**.
2. Locate an alert produced by a rule that is known to have sub-rules.
3. Open the alert detail view (or the alert visualisation panel).
4. Inspect each firing rule listed on the alert.
5. Open a second alert whose rules do **not** have sub-rules.

**Expected result (CMS UI)**
- At step 4: each firing rule shows a **sub-rule reference** ("Sub-ref: ...") where the alert payload includes one. Band and exit-condition reason are **not expected to appear in the CMS view** in this release (see C2, C3) — they are emitted to the BIAR lakehouse only.
- At step 5: the alert renders cleanly with the sub-rule reference simply absent. No error, no misleading empty placeholder.

**Cross-check (BIAR side, for C2 and C3)**
- If the tester has BIAR access: query `alerts_nav_rules` for the alert ID and confirm that `matched_band_reason`, `band_reasons_json`, and `matched_exit_condition_reason` columns are populated as expected for the rules on that alert.

---

### Scenario 14 — Investigation notes flow through to the lakehouse

**Acceptance criteria covered:** C4

1. Sign in as **Investigator A**. Open a case with at least one task.
2. Open the task and add an **investigation note** (for example: "Verified against KYC record on 13 July 2026 — inconsistency in beneficiary address").
3. Save the note.
4. Wait for the next lakehouse refresh cycle (or ask a system administrator to run the Tasks ETL).
5. In the BIAR dashboard (or a Jupyter notebook against the `gold/tasks` table), locate the task by ID and inspect the `investigationNotes` column.
6. Return to the CMS, edit the same note, and save.
7. After the next refresh, re-check the lakehouse.

**Expected result**
- After step 5: the note text is present, complete, and matches what was entered in the CMS.
- After step 7: the updated content is reflected. No duplication.

---

### Scenario 15 — Alert Navigator shows correct transaction lineage

**Acceptance criteria covered:** C5

1. Sign in as **Investigator A**. Navigate to the **Alert Navigator** (or the CMS view that surfaces transaction lineage for an alert).
2. Open an alert whose underlying transaction status, ID, amount, and currency are known (from operational records).
3. Inspect the transaction lineage fields.

**Expected result**
- **Transaction status** matches the known value on the transaction record.
- **Transaction ID** matches.
- **Transaction amount** and **currency** match.
- **Instructing agent** and **instructed agent** continue to display correctly (regression check).

---

### Scenario 16 — Backward compatibility of task status representation

**Acceptance criteria covered:** C6

If the tester has access to BIAR dashboards / notebooks:

1. Open any pre-existing dashboard that reads `gold/tasks.status`.
2. Open a notebook or dashboard that reads `gold/tasks.status_raw`.

**Expected result**
- At step 1: the dashboard continues to render task statuses in **upper-case normalised form** as it did before.
- At step 2: the raw source value is populated and available for consumers that need it.

---

## 6. Regression Areas to Watch

While testing the above, the tester should also confirm that the following pre-existing capabilities continue to work unchanged:

- **Single-type cases** (pure FRAUD, pure AML) open, assign, close, and report exactly as before this release.
- **Case reassignment** across investigators still works.
- **Case suspension and reopen** flows still work.
- **Auto-close of stale unassigned cases** (STATUS_71 / STATUS_72) still work and are reflected in reports.
- **Existing dashboards and reports** (outside the specific numbers that changed) render without errors.
- **Login, session, and role-based navigation** are unaffected.

---

## 7. Definition of Done for This Release

The Product Owner may accept this release when:

- [ ] All Scenarios 1–16 have been executed and passed (scenarios marked **Not Applicable** are skipped, not counted as failures).
- [ ] Every acceptance criterion in Section 3 has been observed as true at least once (subject to the environment limitations called out in Section 5).
- [ ] Regression areas in Section 6 have been spot-checked without discovering new issues.

---

## 8. Out of Scope for This Release

The following items are **not** part of this development and should not block acceptance:

- The two external Tazama review sub-issues (paysys-pmo #834 and #833) — these are review-tracker items with no further code change required from our side.
- Any changes to the alert-ingestion pipeline itself. Alert content and detection logic are unchanged.
- Changes to authentication, tenancy, or role-management screens.
- Any BIAR-side dashboard redesign — only the lakehouse schema and ingestion are updated to keep parity with CMS.

### Known gaps / possible follow-ups (surfaced during code fact-check)

These are not scope of this delivery, but the tester should confirm the Product Owner's intent on each:

- **A1 — latent AI-triage container gap:** the AI-triage code path in `triage.service.ts` contains a FRAUD_AND_AML branch that would leave the originating case with `case_type = FRAUD_AND_AML` alongside FRAUD and AML sibling cases. This branch is not reachable today because the AI/ML prediction component is not implemented (`predictAlert` returns a hardcoded false-positive) and `getAllCases` does not apply the `NON_CONTAINER_CASE_FILTER`. Address both — either strip the FRAUD_AND_AML branch or apply the container-exclusion filter to the Cases dashboard listing — **before** the AI-triage feature is enabled.
- **B5 — priority-change audit visibility in case history:** the priority-change domain event has no listener writing to `caseHistory`. Audit records exist in the audit store; whether they render in the case-history / task-log tab is a separate wiring question to confirm at runtime.
- **B6 — unclaimed DUE_SOON notifications:** no dedicated notification currently fires when an unclaimed case reaches DUE_SOON. Decide whether such a nudge is expected.
- **C2 / C3 — CMS-side rendering of band and exit-condition reason:** these fields are emitted to the BIAR lakehouse (`alerts_nav_rules`) but the CMS alert view does not render them. If the Product Owner requires investigator-facing display in CMS, raise as a follow-up scope item.

---

*Prepared for paysys-pmo #814 / #823 and BIAR alert-enrichment work. See the linked CMS and BIAR pull requests in the header for implementation detail.*
