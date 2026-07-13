# Testing Report

## Case Management System — Investigation Model, SLA/Priority Overhaul & BIAR Alert Enrichment

**Prepared for:** Product Owner
**Date:** 13 July 2026
**Development scope:** paysys-pmo tracker issues **#814** and **#823**, plus BIAR alert-enrichment work
**Underlying CMS issues:** [CMS #214](https://github.com/tazama-lf/case-management-system/issues/214), [CMS #220](https://github.com/tazama-lf/case-management-system/issues/220)
**Implementation:** [CMS PR #233](https://github.com/tazama-lf/case-management-system/pull/233), [CMS PR #234](https://github.com/tazama-lf/case-management-system/pull/234), [CMS PR #240](https://github.com/tazama-lf/case-management-system/pull/240), [BIAR PR #107](https://github.com/tazama-lf/biar/pull/107), [BIAR PR #118](https://github.com/tazama-lf/biar/pull/118)

---

## 1. Executive Summary

This report defines the **User Acceptance Testing (UAT)** scope for three related improvements to the Case Management System (CMS) that were developed in parallel and are being released together. All three are visible in the CMS web application and, downstream, in the BIAR data-lakehouse dashboards.

| Track | Business Problem | What Changed |
|-------|------------------|--------------|
| **Track A — Investigation Modelling** (paysys-pmo #814 / CMS #214) | A single FRAUD_AND_AML alert generated **three** cases in the system — one synthetic "container" plus one FRAUD and one AML investigation — causing triple-counted reports, permission bypass, an unnatural "Completed" status, and forced supervisor closures. | The container case has been removed. A FRAUD_AND_AML alert now creates **exactly two cases** (FRAUD and AML), each with its own owner, its own lifecycle, and standard permissions. Reports and totals reconcile correctly. |
| **Track B — SLA / Priority Split** (paysys-pmo #823 / CMS #220) | A single `Priority` field was being silently overwritten every hour by a background job based on case age. Supervisors' manual escalations were reverted within an hour. Priority meant "how old", not "how serious." | Priority (severity) and SLA state (timing) are now two independent, orthogonal fields. Priority is stable (LOW / MEDIUM / HIGH) and only changed via an audited supervisor action. A new SLA State (ON_TRACK / AT_RISK / DUE_SOON / BREACHED) is derived live from the case's deadline. |
| **Track C — BIAR Alert Enrichment** (BIAR PR #107) | Alert visualisations in the CMS did not expose the **sub-rule references**, **band information**, or **exit-condition reasons** behind an alert. Investigators had to guess *why* a rule fired. Investigation notes captured on tasks did not flow to the downstream data lakehouse, and the Alert Navigator lineage for transaction status / ID / amount was inconsistent. | Alert visualisations now display **band, exit-condition reasons, and sub-rule references** for each rule on the alert. **Investigation notes** are now carried end-to-end into the lakehouse for downstream analytics. **Alert Navigator** transaction lineage (status, ID, amount, currency) now enriches correctly from the current transactions Gold schema. |

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

Priority calibrates the SLA clock: HIGH = 24-hour target, MEDIUM = 72-hour, LOW = 168-hour (tenant-configurable). If a supervisor bumps priority mid-investigation, the deadline is re-anchored on the case's **original creation date** — an old case cannot be gamed into "on track" by lowering its priority.

### 2.3 Track C — Why alert enrichment matters

Investigators reviewing an alert previously saw *that* a rule fired but not the finer detail behind it. Specifically:

- **Sub-rule references** — a rule can trigger for several underlying reasons (sub-rules). Without seeing which sub-rule fired, the investigator has to reconstruct the "why" manually.
- **Band** — the band captures the qualitative bucket of the rule's evaluation. It gives the investigator a quick "how bad" indicator per rule, independent of the raw score.
- **Exit-condition reasons** — the natural-language rationale for why the rule stopped where it did. Essential for evidence packs and audit review.
- **Investigation notes** — investigators capture notes on tasks during a case. Previously these did not travel into the BIAR lakehouse, so downstream analytics and management dashboards could not surface investigator context.
- **Alert Navigator transaction lineage** — the Alert Navigator view is used to trace an alert back to the underlying transaction (status, ID, amount, currency). Previously this enrichment was inconsistent because it drew from the wrong Gold-schema fields.

**After this release:** all three enrichments (band, exit-condition reasons, sub-rule references) appear on the alert visualisations that back the CMS alert view; investigation notes flow to the lakehouse for downstream reporting; and Alert Navigator lineage is accurate against the current transactions Gold schema.

---

## 3. Acceptance Criteria (Consolidated)

The following acceptance criteria define what must be true for the Product Owner to sign off on this release. Each criterion is written to be verifiable from the frontend by a non-technical tester.

### 3.1 Track A — Investigation Modelling

**A1. Case count and list**
- A FRAUD_AND_AML alert produces exactly **two** cases in every list, filter, and export (one FRAUD, one AML).
- No row anywhere in the application has case type `FRAUD_AND_AML`.
- The Case Type filter option `FRAUD_AND_AML` (if still shown as a filter value) returns zero results.

**A2. Close-Case modal**
- On any FRAUD or AML case, the Close-Case modal offers exactly three outcomes: **Closed Confirmed**, **Closed Refuted**, **Closed Inconclusive**.
- The **"Completed"** outcome (STATUS_84) does not appear anywhere in the application.
- No **"Sub-Cases Closure Status"** table appears in the Close-Case modal.
- No supervisor-only **"Close Case (AFTER report)"** button appears; a normal investigator can close FRAUD or AML independently.

**A3. Case Filters screen**
- The Status filter drop-down does **not** contain "Completed" (STATUS_84).
- No STATUS_84 badge colour appears anywhere in the UI (case rows, case detail, dashboards).

**A4. Permissions**
- A user who is neither owner nor assignee of a FRAUD or AML case receives a **403** on any attempt to access, close, or modify it.
- FRAUD_AND_AML no longer receives any permission special-case.

**A5. Independence of sibling cases**
- Assigning the FRAUD case does not change the AML case.
- Closing the FRAUD case does not change the AML case (and vice versa).
- Suspending or reopening one case has no effect on any other case.

**A6. Reports and dashboards**
- Total Cases counts each alert once (a single FRAUD_AND_AML alert contributes 2 to totals, not 3).
- Case Type breakdown shows **FRAUD** and **AML** buckets only — no `FRAUD_AND_AML` bucket.
- Closed Cases = Closed Confirmed + Closed Refuted + Closed Inconclusive (plus system auto-closures 71/72). Totals reconcile.
- Average Resolution Time is not distorted by instant container closures.
- Investigator Workload does not attribute the (former) ownerless container to any investigator.

**A7. Performance**
- Triage of a FRAUD_AND_AML alert completes within **5 seconds** from the user's perspective.

### 3.2 Track B — SLA / Priority Split

**B1. Priority values and colours**
- Every Priority drop-down, badge, filter, and export shows only **Low**, **Medium**, **High**.
- `NEW`, `URGENT`, `CRITICAL`, and `BREACH` do not appear anywhere.
- Badge colours: **Low** — blue / green; **Medium** — amber / yellow; **High** — red / orange.

**B2. Priority stability**
- A case whose priority has been set (whether by triage or by a supervisor) does **not** have that value changed by any background process, over any elapsed time.
- A supervisor-set priority persists across at least two hourly SLA-cron cycles.
- New cases still receive a priority automatically at triage, derived from the risk score.

**B3. SLA deadline**
- Every new case has a **deadline** attached at creation, derived from priority: HIGH = 24 h, MEDIUM = 72 h, LOW = 168 h (or the tenant-configured values).
- If a supervisor changes priority mid-case, the deadline is **recalculated from the case's original creation date**, not from the moment of the change.

**B4. SLA State badge**
- The case list and case detail views display an **SLA State** badge **independently** of the Priority badge.
- Values and colours: **On Track** (green), **At Risk** (yellow), **Due Soon** (amber), **Breached** (red).
- A case without a deadline shows no SLA badge and no error.

**B5. Priority-change permissions and audit**
- A non-supervisor attempting to change priority receives a **403**.
- Every priority change creates an **audit record** capturing actor, timestamp, old value, new value, and reason.
- Audit records are visible in the case history / task log tab.

**B6. Notifications**
- Each `(case, SLA-state)` transition fires **at most one** notification — no hourly re-notification on breached cases.
- Distinct notifications exist for: SLA At-Risk (unclaimed — "claim-chase"), SLA Due-Soon (owned — "support-chase"), SLA Due-Soon (unclaimed), and SLA Breached.
- The notification payload includes: case ID, case type, SLA state, time remaining, and assignee (where applicable).

**B7. Filtering**
- Filtering by **High** returns high-risk cases regardless of age.
- Filtering by **Low** returns low-risk cases regardless of age.
- The Priority filter drop-down lists exactly: Low, Medium, High.

**B8. Reports**
- The Workload report's High Priority bucket contains cases with Priority = HIGH (a severity measure), **not** old cases.
- Report totals reconcile: `totalCases = low + medium + high`.

**B9. Performance**
- SLA cron job completes within **60 seconds** for a caseload of 10,000 open cases (system-observable, not directly UI-visible).

### 3.3 Track C — BIAR Alert Enrichment

**C1. Sub-rule references on the alert view**
- On any alert whose rules have sub-rules, the alert detail view (or the alert visualisation the investigator uses) displays a **sub-rule reference** for each firing rule.
- Alerts whose rules have no sub-rules render cleanly with the field simply absent (no error, no empty placeholder that misleads).

**C2. Band displayed per rule**
- Each firing rule on an alert exposes its **band** value.
- The band value is displayed in a way that a non-technical investigator can interpret (a label or a colour-coded chip, matching the way other rule attributes are shown).

**C3. Exit-condition reason displayed per rule**
- Each firing rule on an alert exposes its **exit-condition reason** as human-readable text.
- The reason text appears alongside the rule outcome and remains visible when the investigator scrolls through multiple rules on the same alert.

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
   - **At least one alert triggered by a rule that carries sub-rules, band, and exit-condition reasons** (for Track C — sub-rule reference, band, and exit-condition reason display).
   - **At least one task on an existing case with an investigation note already recorded** (for Track C — lakehouse propagation check).
4. **BIAR dashboards** (optional cross-service check): access to the BIAR dashboard for confirming lakehouse-side counts. This is not the Product Owner's primary responsibility but is available for cross-verification.

---

## 5. Test Scenarios

Each scenario is a self-contained walkthrough. Steps assume the tester is signed into the CMS web app.

### Scenario 1 — A new FRAUD_AND_AML alert creates only two cases

**Acceptance criteria covered:** A1, A5, A7

1. Sign in as **Investigator A**.
2. Navigate to **Alerts Dashboard** from the main navigation.
3. Locate a fresh FRAUD_AND_AML alert (or ingest one). Click the alert row to open the **Alert Details** modal.
4. Trigger triage from the modal (use the manual triage flow).
5. Once triage completes, close the modal and navigate to the **Cases** area.
6. Filter or search by the alert reference / customer.

**Expected result**
- Exactly **two** case rows appear for this alert — one with case type **FRAUD** and one with case type **AML**.
- Neither case has type `FRAUD_AND_AML`.
- Both cases have an owner assigned (no ownerless "container" appears in the list).
- The triage-to-cases transition completes in under 5 seconds.

7. Open the FRAUD case. Under the **Linked Items** (or equivalent) area, confirm the AML sibling is discoverable via the shared investigation group. There is **no** "parent case" panel.

---

### Scenario 2 — Sibling cases are independent

**Acceptance criteria covered:** A5

1. Sign in as **Investigator A**. Open the FRAUD case created in Scenario 1.
2. Assign yourself as owner (or note the current owner).
3. Sign in (in a second browser session) as **Investigator B**. Open the sibling AML case.
4. On the AML case, change the assignee to Investigator B.
5. On the FRAUD case tab, refresh the page.

**Expected result**
- The FRAUD case's owner / assignee is unchanged by any action taken on the AML case.
- The AML case's owner / assignee is unchanged by any action taken on the FRAUD case.

6. As Investigator A, close the FRAUD case (see Scenario 3).
7. Refresh the AML case as Investigator B.

**Expected result**
- The AML case's status is unchanged. It remains open and independently closable.

---

### Scenario 3 — Close-Case modal offers only real outcomes

**Acceptance criteria covered:** A2

1. Sign in as **Investigator A**. Open any FRAUD or AML case ready to be closed.
2. Click **Close Case** in the Case Actions panel.
3. Inspect the **Final Outcome** drop-down.

**Expected result**
- Exactly three options appear: **Closed Confirmed**, **Closed Refuted**, **Closed Inconclusive**.
- **"Completed"** is **not** an option.
- No **"Sub-Cases Closure Status"** table is rendered inside the modal.
- No supervisor-only "Close Case (AFTER report)" secondary button appears.

4. Select an outcome, provide required notes, and confirm.

**Expected result**
- The case closes to the chosen status. No sibling case is affected. No supervisor approval is required.

---

### Scenario 4 — Filters and dashboards no longer expose the container concept

**Acceptance criteria covered:** A1, A3, A6

1. Sign in as any user with report access.
2. Navigate to **Cases** and open the **Filters** panel.
3. Inspect the **Status** filter drop-down.

**Expected result**
- No **"Completed"** (STATUS_84) option appears in the drop-down.

4. Inspect the **Case Type** filter. If `FRAUD_AND_AML` is still present as a legacy filter value, select it and apply the filter.

**Expected result**
- Zero rows return.

5. Navigate to **Reports → Case Status** (or equivalent).
6. Compare Total Cases with (Open Cases + Closed Cases). Compare Closed Cases with the sum of Closed Confirmed + Closed Refuted + Closed Inconclusive (+ system auto-closures).

**Expected result**
- Both identities balance exactly. No orphan "Completed" bucket exists.
- The **Case Type breakdown** chart shows only FRAUD and AML segments — no `FRAUD_AND_AML` segment.
- The **Investigator Workload** chart contains no unowned "container" entries.

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

**Expected result**
- Exactly three options: **Low**, **Medium**, **High**. No `NEW`, `URGENT`, `CRITICAL`, or `BREACH` value appears.

2. Filter by **High**. Inspect the returned rows.

**Expected result**
- The rows include recent HIGH-severity cases as well as older HIGH-severity cases. Age is not the filter criterion.

3. Repeat for **Low** and **Medium**.
4. Navigate to **Alerts Dashboard**. Open its priority filter.

**Expected result**
- Only Low / Medium / High are listed there too. Alert cards display the new colours (Low blue/green, Medium amber, High red/orange).

5. Open any case that existed before this release (a historically migrated case).

**Expected result**
- Its priority is displayed as **Low**, **Medium**, or **High** — the migration converted NEW → LOW, URGENT → MEDIUM, CRITICAL → HIGH, BREACH → HIGH.

---

### Scenario 7 — Priority is stable and no longer overwritten by the background job

**Acceptance criteria covered:** B2

1. Sign in as **Supervisor**. Open a case currently at **Priority = Low**.
2. Use the **Change Priority** action in the Case Actions panel. Change priority to **High** and provide a reason (e.g., "Manual escalation — new intelligence").
3. Confirm the change; verify the badge now shows **High**.
4. Wait at least **two hours** (so the SLA cron has run at least twice), or ask a system administrator to trigger the SLA job twice.
5. Return to the same case.

**Expected result**
- Priority remains **High**. No background job has reverted it.
- The case history / task log tab shows an audit entry: who changed the priority, when, from what to what, and the reason.

---

### Scenario 8 — SLA State badge appears and progresses through states

**Acceptance criteria covered:** B3, B4, B6

**Setup**: Ensure there is a HIGH-priority case whose deadline is within the next 2 hours (or use test controls to shorten the deadline).

1. Open the case's **View Case** modal.

**Expected result**
- A distinct **SLA State** badge is visible in the Case Details tab, separate from the Priority badge.
- The badge reads **On Track** (green), **At Risk** (yellow), **Due Soon** (amber), or **Breached** (red) depending on the remaining time.

2. Leave the case unresolved and let time pass (or use an accelerated test clock, if available in UAT).
3. Refresh the case at intervals corresponding to 1/3, 2/3, and past the deadline.

**Expected result**
- The SLA State badge transitions **On Track → At Risk → Due Soon → Breached**.
- The Priority badge remains unchanged throughout (still **High**).
- The assigned investigator (or supervisor for unclaimed cases) receives **one** in-app notification per transition — no repeated hourly alerts.

---

### Scenario 9 — Supervisor changes priority; SLA deadline re-anchors correctly

**Acceptance criteria covered:** B3, B5

1. Identify a case that has been open for **48 hours** currently at **Priority = Medium** (deadline was set at creation to +72 h; case is currently On Track with ~24 h to go).
2. Sign in as **Supervisor**. Open the case and use **Change Priority** to move to **High**. Provide a reason.
3. Refresh the case.

**Expected result**
- The SLA deadline is now recalculated based on the original creation timestamp + 24 h (the HIGH target). Since the case is already 48 h old, the deadline has already passed and the SLA State badge should immediately show **Breached**.
- A single Breach notification fires for the assignee.
- An audit record for the priority change is present in the case history.

4. Attempt to change priority as an **Investigator** (non-supervisor) via the same action.

**Expected result**
- The action is blocked (button hidden or a **403** response is returned).

---

### Scenario 10 — Reports reconcile and buckets reflect severity, not age

**Acceptance criteria covered:** A6, B7, B8

1. Navigate to **Reports**.
2. Open the **Case Status Report**. Confirm:
   - Total = Open + Closed.
   - Closed = Confirmed + Refuted + Inconclusive (+ system auto-closures).
3. Open the **Workload Report** (or the priority-breakdown chart).
   - Note the count of the **High Priority** bucket.
   - Drill into the bucket.

**Expected result**
- Every case in the High bucket has Priority = **High** (severity). The bucket is not populated by cases that are simply old.
- Totals reconcile: `low + medium + high = totalCases`.
4. Verify the priority buckets show the new values (Low / Medium / High) — no legacy NEW / URGENT / CRITICAL / BREACH values in any legend or bar label.

---

### Scenario 11 — Notification idempotency (no hourly noise)

**Acceptance criteria covered:** B6

1. Identify a case that is currently **Breached**.
2. Note the timestamp and content of the last SLA breach notification for that case.
3. Wait for **at least two SLA cron cycles** (typically 2 hours).
4. Check the notifications inbox for the assignee and supervisor.

**Expected result**
- **No additional Breach notification** has fired for that same case in the interim. The breach notification is a one-shot per `(case, SLA state)` pair.

---

### Scenario 12 — Cross-service check (BIAR lakehouse, optional)

**Acceptance criteria covered:** downstream data integrity for A1, A6, B1, B4

If the tester has access to the BIAR dashboard:

1. Open the case-related dashboards.
2. Confirm that the total case count for the test period matches the CMS Reports (no triple-counting of FRAUD_AND_AML alerts).
3. Confirm that priority breakdowns show Low / Medium / High buckets only.
4. Confirm the presence of SLA-related tables in the lakehouse (Investigation Groups, SLA Policies, SLA Escalation Records) — evidence that BIAR is receiving the new schema correctly.

---

### Scenario 13 — Sub-rule references, band, and exit-condition reasons are visible on the alert

**Acceptance criteria covered:** C1, C2, C3

1. Sign in as **Investigator A**. Navigate to **Alerts Dashboard**.
2. Locate an alert produced by a rule that is known to have sub-rules, banded evaluation, and exit-condition reasons.
3. Open the alert detail view (or the alert visualisation panel).
4. Inspect each firing rule listed on the alert.

**Expected result**
- Each firing rule shows a **sub-rule reference** (where applicable) — an identifier or short label linking to the specific sub-rule that triggered.
- Each firing rule shows a **band** value — presented as a label or colour-coded chip.
- Each firing rule shows a **human-readable exit-condition reason** — a short explanation of why the rule concluded as it did.

5. Open a second alert whose rules do **not** have sub-rules.

**Expected result**
- The alert renders cleanly with the sub-rule reference simply absent. No error, no misleading empty placeholder.

---

### Scenario 14 — Investigation notes flow through to the lakehouse

**Acceptance criteria covered:** C4

1. Sign in as **Investigator A**. Open a case with at least one task.
2. Open the task and add an **investigation note** (for example: "Verified against KYC record on 13 July 2026 — inconsistency in beneficiary address").
3. Save the note.
4. Wait for the next lakehouse refresh cycle (or ask a system administrator to run the Tasks ETL).
5. In the BIAR dashboard (or a Jupyter notebook against the `gold/tasks` table), locate the task by ID and inspect the `investigationNotes` column.

**Expected result**
- The note text is present, complete, and matches what was entered in the CMS.

6. Return to the CMS, edit the same note, and save.
7. After the next refresh, re-check the lakehouse.

**Expected result**
- The updated content is reflected. No duplication.

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

**Expected result**
- The dashboard continues to render task statuses in **upper-case normalised form** as it did before.

2. Open a notebook or dashboard that reads `gold/tasks.status_raw`.

**Expected result**
- The raw source value is populated and available for consumers that need it.

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

- [ ] All Scenarios 1–16 have been executed and passed.
- [ ] Every acceptance criterion in Section 3 has been observed as true at least once.
- [ ] Regression areas in Section 6 have been spot-checked without discovering new issues.
- [ ] Any defects raised during UAT are triaged and either fixed or explicitly deferred with Product Owner agreement.

---

## 8. Out of Scope for This Release

The following items are **not** part of this development and should not block acceptance:

- The two external Tazama review sub-issues (paysys-pmo #834 and #833) — these are review-tracker items with no further code change required from our side.
- Any changes to the alert-ingestion pipeline itself. Alert content and detection logic are unchanged.
- Changes to authentication, tenancy, or role-management screens.
- Any BIAR-side dashboard redesign — only the lakehouse schema and ingestion are updated to keep parity with CMS.

---

## 9. Sign-off

| Role | Name | Signature | Date |
|------|------|-----------|------|
| Product Owner | | | |
| QA Lead | | | |
| Engineering Lead | | | |

---

*Document prepared based on paysys-pmo tracker issues #814 and #823, CMS issues #214 and #220, CMS pull requests #233, #234, #240, and BIAR pull requests #107 and #118. For technical detail beyond the acceptance criteria in this document, refer to the impact and solution documents held in the UAT board repository under `issues/case-management-system/214/` and `issues/case-management-system/220/`.*
