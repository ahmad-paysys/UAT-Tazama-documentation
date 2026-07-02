# Per-Issue Analysis Report — Beta Testing Board
**Board:** [tazama-lf/projects/20](https://github.com/orgs/tazama-lf/projects/20/views/1)
**Report Date:** 2026-06-29
**Scope:** All open CMS and BIAR issues. Code reviewed from `case-management-system` and `biar` repos (dev branch).

---

## Legend

| Effort | Meaning |
|---|---|
| 🟢 Small | < 1 day. Isolated change, known location. |
| 🟡 Medium | 1–2 days. Multiple files/layers, some coordination needed. |
| 🔴 Large | 3+ days. Cross-system, new infrastructure, or architectural change. |

---

## In Progress

---

### [#146](https://github.com/tazama-lf/case-management-system/issues/146) — Alert navigator visualization not showing alerted typologies/rules
**Repo:** CMS · **Assignee:** ibadkhan088 · **Status:** In Progress

**Description:** The alert navigator visualization is not rendering alerted typologies and rules. A related request in comments also asks to remove meaningless bottom metrics (typologies triggered, rules passed, avg score).

**Code Analysis:**
The alert navigator data pipeline is fully in place. The backend [`alerts-lakehouse.service.ts`](case-management-system/backend/src/modules/gold-lakehouse/alerts-lakehouse.service.ts) queries `alert_navigator_rules` and `alert_navigator_typologies`, joins them, and returns structured `typologies[]` with `rules[]` nested inside each one. The SQL already filters `rule_weight > 0 OR rule_id = 'EFRuP@1.0.0'`, so only triggered rules are returned.

The issue is on the **frontend rendering side**: the visualization component is not correctly consuming the `typologies` array or mapping `rules` per typology. The metrics to be removed (typologies triggered, rules passed, avg score) are likely computed as `statistics.totalTypologies` and `statistics.totalRules` in the response and rendered as separate UI widgets.

**Comments note:** Data comes from the Lakehouse (not the evaluation table directly), so testing must compare against Lakehouse responses, not raw evaluation data.

**What needs to change:**
- Frontend: fix the alert navigator visualization component to correctly render each typology and its triggered rules from the response
- Frontend: remove the 3 bottom metric widgets (typologies triggered, rules passed, avg score)
- No backend changes required — the data is already correct

**Effort:** 🟡 Medium (1–2 days frontend work)

---

### [#158](https://github.com/tazama-lf/case-management-system/issues/158) — Transactions History cumulative value graph has no data
**Repo:** CMS · **Assignee:** sobia-rizwan1567 · **Status:** In Progress (End Date: 2026-06-26, overdue)

**Description:** The cumulative value graph in the Transactions History visualization shows no data.

**Code Analysis:**
The backend [`transaction-lakehouse.service.ts`](case-management-system/backend/src/modules/gold-lakehouse/transaction-lakehouse.service.ts) queries the `transaction_history` table for `cum_tx_amount` and `cum_tx_count` per event row. These are mapped to a `cumulative[]` array at line ~286–290:
```ts
cumulativeAmount: parseFloat(e.cum_tx_amount) || 0,
cumulativeCount: parseInt(e.cum_tx_count, 10) || 0,
```
The SQL `WHERE th.row_type = 'EVENT'` filters for only event rows and joins `transaction_detail` to get timestamps.

The issue is most likely that `cum_tx_amount` is `null` or not populated in the `transaction_history` table for the Tazama tenant's data — the `|| 0` fallback would silently zero the values. It could also be a frontend charting issue where an empty or all-zero `cumulative` array is not rendered as "no data" but instead as a flat zero line that looks blank.

**What needs to change:**
- Investigate whether `cum_tx_amount` / `cum_tx_count` are populated in the Lakehouse `transaction_history` table for the test tenant
- If data is missing: the ETL/ingestion pipeline (BIAR side) needs to be checked
- If data is present but charting fails: fix the frontend chart component to correctly bind the `cumulative` array
- No structural backend code change expected

**Effort:** 🟡 Medium (investigation + either ETL fix or frontend fix)

---

### [#165](https://github.com/tazama-lf/case-management-system/issues/165) — The "score" is not the sum of the rule weights
**Repo:** CMS · **Assignee:** MuneebPaysys · **Status:** In Progress

**Description:** The typology score displayed in the CMS does not equal the sum of the triggered rule weights.

**Code Analysis:**
In [`alerts-lakehouse.service.ts`](case-management-system/backend/src/modules/gold-lakehouse/alerts-lakehouse.service.ts), `typologyScore` is populated directly from `ant.typology_score` fetched from the `alert_navigator_typologies` Lakehouse table — it is **not** computed as the sum of rule weights in the CMS code. The developer comment in #165 confirms this: *"the typology score is not manually calculated; it is fetched from the Lakehouse API via the typology_score field"*.

This means the score is whatever the TMS/FRMS evaluation engine stored in the Lakehouse. If it doesn't match the sum of rule weights, the bug is either:
1. In how TMS computes and stores `typology_score` (upstream, not CMS)
2. In how the Lakehouse ETL (`biar/automation-orchestrator/Table_ETLs/EvaluationETL.py`) transforms the field

**What needs to change:**
- Verify what `typology_score` value TMS stores in the evaluation result vs. what the Lakehouse ETL persists
- If the ETL is wrong: fix `EvaluationETL.py` in BIAR
- If TMS stores the score correctly but CMS needs to recalculate it from rule weights as a cross-check: add that computation in the backend service
- CMS frontend likely needs no change

**Effort:** 🔴 Large (requires cross-system investigation between TMS output, BIAR ETL, and CMS display)

---

### [#163](https://github.com/tazama-lf/case-management-system/issues/163) — Score value displayed when no rules are triggered
**Repo:** CMS · **Assignee:** MuneebPaysys · **Status:** In Progress

**Description:** A score value is displayed in the alert navigator even when no rules were triggered for a typology.

**Code Analysis:**
In [`alerts-lakehouse.service.ts`](case-management-system/backend/src/modules/gold-lakehouse/alerts-lakehouse.service.ts) the query filters rules with `rule_weight > 0 OR rule_id = 'EFRuP@1.0.0'` — so non-triggered rules are excluded from the `rules[]` list. However, `typologyScore` is still populated from `ant.typology_score` for every typology row, regardless of whether it has triggered rules. A typology with zero triggered rules will still show its stored score.

**What needs to change:**
- Backend: in `getAlertNavigatorData()`, after mapping typologies, if `triggeredRulesData.length === 0` (and no EFRuP), set `typologyScore` to `0` or `null`
- Or: frontend — conditionally display the score only when `rules` array is non-empty after parsing
- Small targeted fix in one function

**Effort:** 🟢 Small (1–2 hours, one-line guard in the backend mapping or frontend render)

---

### [#82](https://github.com/tazama-lf/biar/issues/82) — DLH BUG: tx_status incorrectly used as EFRUP block/override outcome
**Repo:** BIAR · **Assignee:** MuneebPaysys · **Status:** In Progress (End Date: 2026-06-24, overdue)

**Description:** The `tx_status` field is incorrectly being used as the EFRUP block/override outcome in the Data Lakehouse.

**Code Analysis:**
This is entirely in the BIAR automation-orchestrator ETL layer. The relevant file is likely [`ALertsETL.py`](biar/automation-orchestrator/Table_ETLs/ALertsETL.py) or [`EvaluationETL.py`](biar/automation-orchestrator/Table_ETLs/EvaluationETL.py). The EFRUP block/override outcome should come from the evaluation result's `block_or_override_status` field, not from `tx_status`. `tx_status` represents the payment transaction status (e.g. ACCEPTED/REJECTED), which is a different concept.

**What needs to change:**
- Fix the BIAR ETL to source EFRUP outcome from the correct field in the evaluation result JSON
- Update the relevant Lakehouse table (`alert_navigator_header.block_or_override_status`) with correct values
- CMS already reads `block_or_override_status` from the Lakehouse, so no CMS code change needed once the ETL is fixed
- Historical data in the Lakehouse may need reprocessing

**Effort:** 🟡 Medium (ETL fix is small, but Lakehouse re-ingestion of historical data adds time)

---

### [#80](https://github.com/tazama-lf/biar/issues/80) — DLH BUG: TMS metrics incorrectly calculated
**Repo:** BIAR · **Assignee:** MuneebPaysys · **Status:** In Progress (End Date: 2026-06-24, overdue)

**Description:** TMS metrics in the BIAR dashboards are incorrectly calculated.

**Code Analysis:**
Comment from MuneebPaysys confirms the fix will be done in **JupyterHub notebooks** (notebook-specific metric calculation approach), not in the ETL pipeline. This means the calculation logic in the Jupyter dashboard notebooks needs to be corrected. The BIAR repo's `JupyterHub/` folder contains the hub config, but the actual notebook files are deployed at runtime.

**What needs to change:**
- Update the relevant JupyterHub notebook(s) that calculate TMS metrics
- No CMS code change needed
- MuneebPaysys noted "PR will be given tomorrow with rest of the Dashboard notebooks" — PR may already exist or be in review

**Effort:** 🟡 Medium (notebook fix + deploy + retest; data reprocessing may be needed)

---

### [#81](https://github.com/tazama-lf/biar/issues/81) — TMS Metrics - DLH requirements
**Repo:** BIAR · **Assignee:** MuneebPaysys · **Status:** In Progress (End Date: 2026-06-24, overdue)

**Description:** DLH requirements for TMS Metrics — this appears to be the feature/requirement ticket linked to #80.

**Code Analysis:**
Same as #80: fix is in JupyterHub notebooks per standup decision. MuneebPaysys confirmed completion and that a PR is pending. This is closely coupled with #80 and may resolve together.

**What needs to change:**
- Same notebook changes as #80
- Verify that the DLH requirements (specific metric definitions, formulas) are implemented correctly in the notebooks

**Effort:** 🟡 Medium (same scope as #80 — likely resolved in the same PR)

---

### [#71](https://github.com/tazama-lf/biar/issues/71) — BIAR Bug: Case Management Trend Dashboard - no investigator data displayed
**Repo:** BIAR · **Assignee:** MuneebPaysys · **Status:** In Progress (End Date: 2026-06-24, overdue)

**Description:** The Case Management Trend Dashboard shows no investigator data. Root cause identified in comments: CMS fetches user names from Keycloak at runtime but does **not** persist usernames to the database, so the Lakehouse has no investigator name — only IDs.

**Code Analysis:**
Confirmed by `arif-paysyslabs` in comments: *"we aren't saving the user name anywhere in the CMS ODS for a single source of truth"*. The solution outlined requires:
1. A new `users` table in the CMS Postgres database storing `user_id`, `user_name`, `created_at`
2. CMS application logic to persist user details on login
3. A new ETL in BIAR to sync the new `users` table to the Lakehouse

There is currently **no** `user_name` or equivalent field stored anywhere in the backend (confirmed by grep — only `userName` appears as a parameter in `alert.service.ts` and `triage.service.ts` for log messages, never persisted).

**What needs to change:**
- Backend (CMS): create DB migration to add a `users` table
- Backend (CMS): on login in `auth.service.ts`, upsert user record (id + name) to the new table
- BIAR: create a new ETL to ingest the `users` table into the Lakehouse
- BIAR: update dashboard notebook to JOIN on user ID → user name

**Effort:** 🔴 Large (new table, migration, login hook, new ETL, notebook update — 3–5 days across both repos)

---

### [#195](https://github.com/tazama-lf/case-management-system/issues/195) — Multiple issues with Alert Dashboard search and filtering functionality
**Repo:** CMS · **Assignee:** ibadkhan088 · **Status:** In Progress (End Date: 2026-06-29, today)

**Description:** Multiple issues with Alert Dashboard search and filtering (no detail in the issue body itself).

**Code Analysis:**
The filter system exists end-to-end: [`AlertsSearchAndFilters.tsx`](case-management-system/frontend/src/features/alerts/components/AlertsSearchAndFilters.tsx) handles UI state, `filterService` calls the backend, and [`filter.service.ts`](case-management-system/backend/src/modules/filter/filter.service.ts) + [`filter.repository.ts`](case-management-system/backend/src/modules/filter/filter.repository.ts) handle persistence. The save logic works but applying saved filters uses `setTimeout` staggering (0ms, 10ms, 20ms, 30ms, 40ms) to sequence React state updates — a known fragile pattern. The underlying issues are likely:
- Saved filters not being applied correctly (related to #208)
- Filter appearing only after refresh (related to #207)
- Search query behavior with filters active

**What needs to change:**
- Fix the `setTimeout` stagger in `handleSavedFilterSelect` — replace with a single state object update or `useReducer`
- Ensure the `fetchSavedFilters` callback is called after save (it is on the Cases side but not on the Alerts side — `handleSaveCurrentFilters` does NOT call `fetchSavedFilters()` after success in `AlertsSearchAndFilters.tsx`)
- This is the parent ticket for #207 and #208

**Effort:** 🟡 Medium (1–2 days to fix all filter-related issues across both alert and case filter components)

---

### [#207](https://github.com/tazama-lf/case-management-system/issues/207) — Newly Saved Filter Does Not Appear Until Page Refresh
**Repo:** CMS · **Assignee:** ibadkhan088 · **Status:** In Progress (End Date: 2026-06-29, today)

**Description:** When a filter is saved, it does not appear in the saved filters dropdown until the page is refreshed.

**Code Analysis:**
In [`AlertsSearchAndFilters.tsx`](case-management-system/frontend/src/features/alerts/components/AlertsSearchAndFilters.tsx), the `handleSaveCurrentFilters` function calls `filterService.createFilter(payload)` and shows a success toast, but **does not call `fetchSavedFilters()` afterwards**. So the `savedFilters` state is never updated in the current session.

Compare this to [`CaseFilters.tsx`](case-management-system/frontend/src/features/cases/components/CaseFilters.tsx) line ~294 which correctly calls `await fetchSavedFilters()` after saving — that's the fix that needs to be applied to the Alerts component.

**What needs to change:**
- In `AlertsSearchAndFilters.tsx`, add `await fetchSavedFilters()` inside `handleSaveCurrentFilters` after the `success()` toast call (same pattern as `CaseFilters.tsx`)
- One line fix

**Effort:** 🟢 Small (< 30 minutes, one-line fix)

---

### [#165 duplicate note — see above]

---

## Retest

---

### [#142](https://github.com/tazama-lf/case-management-system/issues/142) — Tazama tenant user names not displaying correctly (showing user ID)
**Repo:** CMS · **Assignee:** Sandy-at-Tazama · **Status:** Retest

**Description:** User names are shown as IDs in multiple views including case timeline, case aging report, and investigator workload report. A PR was merged but Sandy confirmed the issue persists in all three views.

**Code Analysis:**
This is the same root cause as #71: CMS does not persist user names to the database — they are fetched live from Keycloak. For report views and the case timeline, the backend either has no Keycloak lookup or is passing raw user IDs. The `arif-paysyslabs` plan (new `users` table, persist on login) is the correct fix.

This issue is **blocked on** the same infrastructure work as #71. Until a `users` table exists and is populated, any fix is superficial (e.g., adding a Keycloak lookup per-request on report endpoints — possible but expensive).

**What needs to change:**
- Shared fix with #71: persist user names in a `users` table on login
- Backend report endpoints need to JOIN user IDs against the `users` table to resolve names
- Affects: case timeline (`caseHistory.service.ts`), aging report, workload report queries

**Effort:** 🔴 Large (blocked on same infrastructure work as #71; ~3–5 days)

---

### [#169](https://github.com/tazama-lf/case-management-system/issues/169) — Display the subruleref for the ERFUP rule
**Repo:** CMS · **Assignee:** Sandy-at-Tazama, ibadkhan088 · **Status:** Retest

**Description:** The EFRUP subruleref should be displayed near the triggered typologies heading. Multiple EFRUP rules may exist but they share the same subruleref value.

**Code Analysis:**
The backend already extracts `flowProcessorData` (the EFRUP subruleref):
```ts
const flowProcessorRule = rulesData.find((r) => r.rule_id === 'EFRuP@1.0.0');
const flowProcessorData = flowProcessorRule?.rule_sub_ref ?? undefined;
```
This is returned as `typology.flowProcessorData` in the `AlertNavigatorDataResponse`. The fix has been implemented in the backend. The issue is whether the **frontend** alert navigator visualization renders `flowProcessorData` from the response. Sandy's retest comment (with a screenshot) suggests the frontend may not yet display it, or it is showing in the wrong position (should be near "Triggered Typologies" heading, not per-typology).

**What needs to change:**
- Frontend: verify the alert navigator visualization reads `flowProcessorData` from the typology object and renders it next to the "Triggered Typologies" heading
- May require confirming that WajahatAhmed's screenshot in comments represents the fixed or unfixed state

**Effort:** 🟢 Small (frontend rendering verification/fix, < 1 day)

---

### [#177](https://github.com/tazama-lf/case-management-system/issues/177) — Independent Variable shows 0 for some typologies
**Repo:** CMS · **Assignee:** RubaZehra · **Status:** Retest

**Description:** The independent variable result displays 0 for some typologies.

**Code Analysis:**
In [`alerts-lakehouse.service.ts`](case-management-system/backend/src/modules/gold-lakehouse/alerts-lakehouse.service.ts), `rule_independent_variable` is fetched from `alert_navigator_rules` and mapped to `independentVariable` in the rules JSON string. Sandy's comment notes: *"An independent variable result of 0 could be valid — does it correspond to the value in the evaluation result?"* This is likely a **data validation issue** not a code bug — 0 may be the correct value for some rule evaluations.

This issue is also blocked on #146 being resolved (Sandy: *"the alert dashboard is still not showing all typologies in the evaluation report so this is still difficult to test"*).

**What needs to change:**
- Confirm with product/data team whether 0 is a valid independent variable
- If it is valid: close the issue
- If not valid: trace the 0 back through the ETL to the TMS evaluation result — likely a BIAR data issue
- No CMS code change expected

**Effort:** 🟢 Small (data validation check, likely no code change)

---

### [#188](https://github.com/tazama-lf/case-management-system/issues/188) — Typologies Displayed Improperly in CMS Alert Details Modal
**Repo:** CMS · **Assignee:** RubaZehra · **Status:** Retest

**Description:** Previously only one typology was shown even if multiple were triggered. Fix was implemented to show multiple. Sandy's retest comment says this is blocked on #146 (alert navigator must be fixed first to validate this properly).

**Code Analysis:**
The display of multiple typologies in the `AlertsDetailModal.tsx` component was fixed per RubaZehra's comment (showing multiple triggered typologies from the response). Sandy's remaining concern is about consistency between the "alert details" view and the "alert navigator visualization" view — they should show the same typologies. This cross-view consistency depends on #146 being resolved first.

**What needs to change:**
- Verify after #146 is fixed that both views (alert details modal and alert navigator) show the same typology set
- No additional code change likely needed beyond what's already been done

**Effort:** 🟢 Small (verification only after #146 is resolved)

---

## Internal Review

---

### [#202](https://github.com/tazama-lf/case-management-system/issues/202) — Investigator column shows ID instead of name in Workload Report
**Repo:** CMS · **Assignee:** WajahatAhmed-paysys · **Status:** Internal Review (End Date: 2026-06-30)

**Description:** The Investigator column in the Investigation Workload Report shows user IDs instead of user names.

**Code Analysis:**
sobia-rizwan1567 commented that this is resolved in [PR #194](https://github.com/tazama-lf/case-management-system/pull/194), but WajahatAhmed's screenshot shows the issue still visible. The fix in PR #194 may resolve the CMS-side query but the Keycloak user lookup for report views is a known systemic gap (same root cause as #142, #71).

**What needs to change:**
- Verify PR #194 is merged and correctly resolves the issue on dev branch
- If the PR uses Keycloak lookups to resolve names: this is a per-request approach that works but is not scalable
- Long-term fix still requires the `users` table from #71

**Effort:** 🟢 Small (verify PR #194 result; likely just needs retest confirmation)

---

### [#201](https://github.com/tazama-lf/case-management-system/issues/201) — User ID shown instead of name in Ageing Report exports
**Repo:** CMS · **Assignee:** WajahatAhmed-paysys · **Status:** Internal Review (End Date: 2026-06-30)

**Description:** Same as #202 but in the Ageing Report exports.

**Code Analysis:**
Same root cause as #202. Both are noted as resolved in PR #194 per sobia's comment. WajahatAhmed's screenshot confirms it still appeared post-PR. The export functionality (likely PDF or CSV generation in the report service) may be pulling raw DB fields without Keycloak resolution.

**What needs to change:**
- Same as #202: verify PR #194 covers the export path (not just the UI view)
- Check [`report.service.ts`](case-management-system/backend/src/modules/report/report.service.ts) for the export generation to ensure user name resolution is applied there too

**Effort:** 🟢 Small (verification + potential one-line fix in report export service)

---

### [#204](https://github.com/tazama-lf/case-management-system/issues/204) — Missing tenant isolation in the `reference_id` table exposes data across tenants
**Repo:** CMS · **Assignee:** WajahatAhmed-paysys · **Status:** Internal Review (End Date: 2026-06-30)

**Description:** ⚠️ **Security issue.** The `reference_id` table does not enforce tenant isolation, allowing data from one tenant to be visible to another.

**Code Analysis:**
Looking at [`admin.repository.ts`](case-management-system/backend/src/modules/repository/admin.repository.ts), both `registerReferenceId` and `getReferenceId` correctly use `tenant_id` as a filter:
- `registerReferenceId`: sets `tenant_id: tenantId` on create
- `getReferenceId`: `findMany({ where: { tenant_id: tenantId } })`

The code as-on-disk (dev branch) **appears to have tenant isolation** implemented. However, sobia's comment says this is resolved in PR #194 — meaning the fix may not have been on dev when the issue was filed, or there is a different code path.

**Possible remaining gaps:**
- Check if the Prisma schema has `tenant_id` as a non-nullable column on the `ReferenceId` model
- Check if any other endpoint (e.g., a global admin list endpoint) bypasses tenant filtering
- PR #194 should be confirmed as merged and tested

**What needs to change:**
- Confirm PR #194 is merged and deployed to dev
- Audit all `referenceId` queries to ensure no cross-tenant read paths exist
- Verify Prisma schema enforces `tenant_id` at the DB level (unique constraint or RLS)

**Effort:** 🟡 Medium (security audit + schema verification; 1 day)

---

## Todo

---

### [#205](https://github.com/tazama-lf/case-management-system/issues/205) — Investigations bug: conditions view not displaying conditions for entity/account
**Repo:** CMS · **Assignee:** ibadkhan088 · **Status:** Todo

**Description:** The conditions view in the Investigations panel is not displaying conditions for an entity or account. Sandy pointed to additional screenshots in org discussion #167.

**Code Analysis:**
The backend [`condition-lakehouse.service.ts`](case-management-system/backend/src/modules/gold-lakehouse/condition-lakehouse.service.ts) has full implementations for:
- `getConditionsByEntity()` — queries `conditions` table by `account_id` or `entity_id`
- `getConditionsSummaryByAccount()` — queries `conditions` table
- `getConditionsListByAccount()` — queries `conditions_timeline` table

The `getConditionsByEntity` method first looks up accounts via `account_holder` WHERE `source = entityId`. If no accounts are found, it returns an empty response. The issue is likely:
1. The `account_holder` table doesn't have records linking entity IDs to accounts for the test data, OR
2. The entity ID format expected differs (note in `getConditionsContextByTransaction`: the code uses `${entityId}TAZAMA_EID` as the enhanced entity ID for `account_holder` lookups, but `getConditionsByEntity` uses the plain `entityId`)

**What needs to change:**
- Verify the `account_holder` table has data for the Tazama tenant entities
- Check if `getConditionsByEntity` should also use the `TAZAMA_EID` suffix format
- Possibly also check the frontend component to ensure it is passing the correct entity/account ID format
- If the conditions table itself has no data: it's a data/ETL issue in BIAR

**Effort:** 🟡 Medium (investigation needed across ETL data and backend query logic; 1–2 days)

---

### [#209](https://github.com/tazama-lf/case-management-system/issues/209) — CMS Dashboard Review and Fixes
**Repo:** CMS · **Assignee:** sobia-rizwan1567 · **Status:** Todo

**Description:** General dashboard review and fixes — no specific details in the issue body.

**Code Analysis:**
Without specific details it's hard to scope precisely. The CMS dashboard has:
- Case summary statistics (counts by status/priority) — in `case-query.service.ts`
- Alert summary statistics — in `alert.statistics.service.ts`
- Jupyter notebook embedded visualizations via the voila proxy

Given the other open issues (#140, #141 closed, #133 closed), known dashboard problems include: case counts not matching detailed views, aging report data scope issues, and user name display.

**What needs to change:**
- Requires clarification of which specific elements need fixing
- Likely overlaps with #140 (aging report scope), #142 (user names), and #211 (assignee column)
- Cannot fully scope without more details from sobia or Sandy

**Effort:** 🟡 Medium (estimated 1–3 days depending on exact scope — needs clarification)

---

### [#215](https://github.com/tazama-lf/case-management-system/issues/215) — CMS Bug: Network analysis visualization errors
**Repo:** CMS · **Assignee:** sobia-rizwan1567 · **Status:** Todo

**Description:** Network analysis visualization has errors.

**Code Analysis:**
Network analysis is served via the [`TransactionLakehouseService`](case-management-system/backend/src/modules/gold-lakehouse/transaction-lakehouse.service.ts) which builds `TransactionNetworkResponseDto` and `CounterpartyNetworkResponseDto`. The frontend likely uses a graph library (D3 or similar) to render these. Common failure modes:
- Lakehouse query returns empty data (missing `network_map` data for the tenant)
- Response shape mismatch between backend DTO and frontend expectation
- Frontend graph rendering errors when nodes/edges are empty

The Jupyter notebooks also cover network analysis (`account-network.ipynb`, `counterparty-network.ipynb`, `transaction-network.ipynb`) — these may be the "visualizations" referred to (voila-rendered notebooks) rather than the React-side network graph.

**What needs to change:**
- Identify whether the error is in the React network graph component or in the voila notebook
- If notebook: fix the relevant `.ipynb` in CMS `notebooks/` directory
- If React: fix the frontend network graph component with null/empty data handling
- If data: check BIAR ETL for network map data

**Effort:** 🟡 Medium (1–2 days; scope depends on whether it's notebook vs React)

---

### [#210](https://github.com/tazama-lf/case-management-system/issues/210) — CMS bug: add visualizations tab to the case details
**Repo:** CMS · **Assignee:** sobia-rizwan1567 · **Status:** Todo

**Description:** A visualizations tab needs to be added to the case details view.

**Code Analysis:**
Looking at the frontend, the case details view likely has tabs (e.g. Overview, Tasks, Comments, History). Adding a new "Visualizations" tab requires:
- Adding a new tab in the case detail page component
- Embedding the appropriate voila-proxy visualizations (the CMS already has a voila-proxy setup via [`voila-proxy.service.ts`](case-management-system/backend/src/modules/voila-proxy/voila-proxy.service.ts) and `jupyter-proxy`)
- The visualizations themselves (notebooks) already exist in `notebooks/`
- Tab routing/state management change

**What needs to change:**
- Frontend: add "Visualizations" tab to case details view component
- Frontend: embed voila notebook frames (account-network, transaction-network, etc.) in the tab
- Backend: likely no change — voila proxy already exists

**Effort:** 🟡 Medium (1–2 days frontend work to add tab + embed notebooks correctly)

---

### [#140](https://github.com/tazama-lf/case-management-system/issues/140) — CMS bug: investigator case aging report shows entire tenant data instead of user-scoped data
**Repo:** CMS · **Assignee:** sobia-rizwan1567 · **Status:** Todo

**Description:** When logged in as an investigator, the case aging report shows data for the entire tenant (262 cases) instead of just the investigator's cases (38 cases). Supervisor view also shows cross-tenant data (262 vs expected 70 for Tazama tenant).

**Code Analysis:**
In [`case-query.service.ts`](case-management-system/backend/src/modules/case/services/case-query.service.ts), `getUserCases` uses `case_owner_user_id` and `tasks.assigned_user_id` as user scoping. However, the aging report is separate — it's likely a Jupyter notebook (voila) that queries the Lakehouse directly without the same user-scoping logic applied. If the report queries the Lakehouse `cases` table without filtering by `case_owner_user_id`, it will return all tenant cases.

Sandy's screenshots show 262 cases for both investigator and supervisor, suggesting the report is **not applying user-level filtering at all**, only (possibly) tenant-level filtering.

**What needs to change:**
- The aging report notebook or backend query needs to accept `userId` as a parameter
- Apply `case_owner_user_id = {userId}` filter when the viewer is an investigator role
- Apply `tenant_id` filter when the viewer is a supervisor (to scope to their tenant only, not all tenants)
- May require updating the voila notebook to receive the JWT/userId via a parameter

**Effort:** 🔴 Large (requires parameterized notebook execution with user context; involves voila, JWT forwarding, and Lakehouse query changes — 2–3 days)

---

### [#167](https://github.com/tazama-lf/case-management-system/issues/167) — Display subruleref reason and rule description next to sub-ref/rule id fields
**Repo:** CMS · **Assignee:** ibadkhan088 · **Status:** Todo

**Description:** In the alert navigator, the subruleref reason from rule configuration and the rule description should be displayed next to the respective fields.

**Code Analysis:**
Comments confirmed that `subRuleRef reason` and `rule description` are **not sent downstream** to CMS in the evaluation result. They exist in the TMS rule configuration database. Sandy confirmed they should be sourced from the rule configuration in the DLH (configuration DB must be available in the Lakehouse).

The current data model in the Lakehouse (`alert_navigator_rules`) includes `rule_sub_ref` but not `rule_description` or `subruleref_reason`. These fields would need to be:
1. Added to the Lakehouse rule configuration table (BIAR ETL)
2. Joined in the `getAlertNavigatorData()` SQL query
3. Returned in the `rules` object
4. Displayed in the frontend

**What needs to change:**
- BIAR: ETL to sync TMS rule configuration (including `subruleref_reason` and `rule_description`) to the Lakehouse
- CMS Backend: update the `getAlertNavigatorData()` SQL to JOIN against the rule config table
- CMS Backend: add `ruleDescription` and `subRefReason` to the rules response type
- CMS Frontend: render the new fields in the alert navigator UI

**Effort:** 🔴 Large (cross-system: BIAR ETL + CMS backend SQL + frontend — 3–4 days)

---

### [#152](https://github.com/tazama-lf/case-management-system/issues/152) — CMS feature: dashboard links to cases assigned to you
**Repo:** CMS · **Assignee:** Sandy-at-Tazama · **Status:** Todo

**Description:** Dashboard summary cards should link to a filtered view showing only cases assigned to the logged-in investigator.

**Code Analysis:**
RubaZehra confirmed that filters are not currently persisted in CMS in a way that makes this easy. However, the Cases Dashboard already supports filtering by `case_owner_user_id` and `assigned_user_id` via the `CaseQueryService`. The required change is purely frontend: make the dashboard summary count cards clickable links that navigate to the cases list with a pre-applied filter (e.g. `/cases?owner={userId}`).

**What needs to change:**
- Frontend: wrap dashboard count cards in `<Link>` or add `onClick` navigation with URL params
- Frontend: ensure the CasesDashboard reads `owner` URL param and applies it as a filter on load
- Backend: no change needed — the filter already works

**Effort:** 🟡 Medium (frontend navigation + URL param filter wiring; ~1 day)

---

### [#153](https://github.com/tazama-lf/case-management-system/issues/153) — CMS feature: dashboard links to high priority alerts
**Repo:** CMS · **Assignee:** Sandy-at-Tazama · **Status:** Todo

**Description:** Dashboard summary cards should link to a filtered view showing high priority alerts.

**Code Analysis:**
Same pattern as #152 but for the Alerts Dashboard. The alerts dashboard already supports priority filtering via `AlertsSearchAndFilters`. The change is: make the dashboard summary/count cards for high-priority alerts into clickable links that navigate to `/alerts?priority=CRITICAL` or similar.

**What needs to change:**
- Frontend: add navigation links from alert priority count cards to the alerts dashboard with pre-set priority filter
- Same as #152: read URL param on mount and apply filter
- Backend: no change

**Effort:** 🟡 Medium (same scope as #152; ~1 day)

---

### [#90](https://github.com/tazama-lf/biar/issues/90) — Alert History displays Unknown for valid alert types
**Repo:** BIAR · **Assignee:** MuneebPaysys · **Status:** Todo

**Description:** The Alert History view shows "Unknown" for valid alert types.

**Code Analysis:**
In [`alerts-lakehouse.service.ts`](case-management-system/backend/src/modules/gold-lakehouse/alerts-lakehouse.service.ts) the `getAlertHistoryAlerts` method maps `r.type ?? 'Unknown'` from the `alert_type_norm` Lakehouse column. If `alert_type_norm` is null or not populated in the Lakehouse for the Tazama tenant's alerts, it falls back to `'Unknown'`.

This is a **BIAR data/ETL issue** — the `alert_type_norm` field is not being correctly populated in the `alerts` Lakehouse table. The fix is in the BIAR ETL (likely [`ALertsETL.py`](biar/automation-orchestrator/Table_ETLs/ALertsETL.py)).

**What needs to change:**
- BIAR: fix `ALertsETL.py` to correctly map/normalize alert type to `alert_type_norm`
- CMS: no code change needed; the null-coalescing `?? 'Unknown'` is appropriate defensive code

**Effort:** 🟢 Small (one field mapping fix in the ETL; < 1 day — but requires Lakehouse re-ingestion)

---

### [#208](https://github.com/tazama-lf/case-management-system/issues/208) — Selected Saved Filter Is Not Applied
**Repo:** CMS · **Assignee:** ibadkhan088 · **Status:** Todo

**Description:** When a user selects a saved filter from the dropdown, the filter is not applied to the alerts view.

**Code Analysis:**
In [`AlertsSearchAndFilters.tsx`](case-management-system/frontend/src/features/alerts/components/AlertsSearchAndFilters.tsx), `handleSavedFilterSelect` applies filters via a sequence of `setTimeout` calls (0ms through 40ms). This is a fragile pattern — if React batches or schedules state updates differently, some `onFilterChange` calls may overwrite each other or fire before the parent component's state has settled. React 18's automatic batching makes this even more unreliable.

Compare to `CaseFilters.tsx` which uses direct sequential calls without setTimeout and works correctly.

**What needs to change:**
- Replace the `setTimeout` chain in `handleSavedFilterSelect` (AlertsSearchAndFilters) with a single `useReducer` dispatch or pass all filter values at once to a combined `onFiltersChange(filters: AlertsSearchFilters)` callback
- Requires refactoring the parent's filter state management to accept a bulk update

**Effort:** 🟡 Medium (refactor of filter state management in the Alerts components; ~1 day)

---

### [#211](https://github.com/tazama-lf/case-management-system/issues/211) — CMS feature: Cases Dashboard - Add Assignee Column and Filter by Assignee
**Repo:** CMS · **Assignee:** RubaZehra · **Status:** Todo

**Description:** Add an Assignee column to the Cases Dashboard table and allow filtering by assignee.

**Code Analysis:**
The backend `CaseQueryService.getUserCases()` already returns `case_owner_user_id` and `assigned_user_id` (via tasks). The `GetAllCasesQueryDto` has `ownerId` and `unassignedOnly` filters. However, the assignee field holds a **user ID** (UUID), not a name — which runs into the same user name problem as #142/#71. Displaying an ID as an assignee column is not useful.

**What needs to change:**
- Frontend: add an "Assignee" column to `CasesTable.tsx` — this is straightforward
- Frontend: add assignee filter to `CaseFilters.tsx` using existing backend query params
- The display name problem means either: show the ID for now (quick), or block this on #71 (correct long-term)
- Backend: no structural change needed; the filter exists

**Effort:** 🟡 Medium (~1 day frontend; but display quality depends on #71 being resolved)

---

### [#212](https://github.com/tazama-lf/case-management-system/issues/212) — CMS bug: Search Parameters for Manual Case Creation
**Repo:** CMS · **Assignee:** RubaZehra · **Status:** Todo

**Description:** Search parameters for manual case creation have issues (no detail in body).

**Code Analysis:**
In [`CreateCaseModal.tsx`](case-management-system/frontend/src/features/alerts/components/AlertsDetailModal.tsx), alert search uses `triageService.getNALTAlerts(alertSearchTerm, ...)` to find alerts to link to a case. The search fires when `alertSearchTerm.length >= 1`. The backend `manual-case-create.dto.ts` and `case-creation.service.ts` handle the actual case creation.

Without more details it's unclear whether the issue is: (a) the search query not returning expected results, (b) incorrect search field being used (e.g. searching by wrong identifier type), or (c) the search results not being properly filtered/displayed.

**What needs to change:**
- Needs clarification from RubaZehra on exact reproduction steps
- Likely a frontend search query issue in `CreateCaseModal.tsx` or `LinkExistingAlerts.tsx`
- Possibly the search needs to support multiple identifier types (alert ID, transaction ID, account ID)

**Effort:** 🟡 Medium (1–2 days, but needs requirements clarification first)

---

### [#213](https://github.com/tazama-lf/case-management-system/issues/213) — CMS feature: Auto-Refresh of the Cases Dashboard
**Repo:** CMS · **Assignee:** RubaZehra · **Status:** Todo

**Description:** The Cases Dashboard should automatically refresh at a configurable interval.

**Code Analysis:**
The Cases Dashboard currently has no polling mechanism. The data fetching hooks (e.g. the cases list query) are invoked on mount and on filter changes only. There is no `refetchInterval` or `setInterval` pattern in any case dashboard hook (confirmed by grep — no `refetchInterval`, `setInterval`, or `useInterval` found in `features/cases/`).

**What needs to change:**
- Frontend: add a `refetchInterval` option (if using React Query) or a `setInterval` + `cleanup` pattern to the cases list data hook
- Frontend: optionally add a UI control for the refresh interval (30s / 1min / 5min or off)
- Backend: no change needed

**Effort:** 🟢 Small (1–2 hours if using React Query `refetchInterval`; ~half day if adding UI toggle)

---

## Summary Table

| Issue | Title (short) | Repo | Assignee | Status | Effort | Blocked By |
|---|---|---|---|---|---|---|
| [#146](https://github.com/tazama-lf/case-management-system/issues/146) | Alert navigator not showing typologies/rules | CMS | ibadkhan088 | In Progress | 🟡 Medium | — |
| [#158](https://github.com/tazama-lf/case-management-system/issues/158) | Tx History cumulative graph no data | CMS | sobia-rizwan1567 | In Progress | 🟡 Medium | Lakehouse data |
| [#165](https://github.com/tazama-lf/case-management-system/issues/165) | Score ≠ sum of rule weights | CMS | MuneebPaysys | In Progress | 🔴 Large | TMS/BIAR ETL |
| [#163](https://github.com/tazama-lf/case-management-system/issues/163) | Score shown when no rules triggered | CMS | MuneebPaysys | In Progress | 🟢 Small | — |
| [#82](https://github.com/tazama-lf/biar/issues/82) | tx_status used as EFRUP outcome | BIAR | MuneebPaysys | In Progress | 🟡 Medium | Lakehouse re-ingest |
| [#80](https://github.com/tazama-lf/biar/issues/80) | TMS metrics incorrectly calculated | BIAR | MuneebPaysys | In Progress | 🟡 Medium | Notebook deploy |
| [#81](https://github.com/tazama-lf/biar/issues/81) | TMS Metrics - DLH requirements | BIAR | MuneebPaysys | In Progress | 🟡 Medium | Same as #80 |
| [#71](https://github.com/tazama-lf/biar/issues/71) | Case Mgmt Trend - no investigator data | BIAR | MuneebPaysys | In Progress | 🔴 Large | New users table |
| [#195](https://github.com/tazama-lf/case-management-system/issues/195) | Alert dashboard search/filter issues | CMS | ibadkhan088 | In Progress | 🟡 Medium | — |
| [#207](https://github.com/tazama-lf/case-management-system/issues/207) | Saved filter not appearing until refresh | CMS | ibadkhan088 | In Progress | 🟢 Small | — |
| [#142](https://github.com/tazama-lf/case-management-system/issues/142) | User names showing as IDs | CMS | Sandy-at-Tazama | Retest | 🔴 Large | #71 (users table) |
| [#169](https://github.com/tazama-lf/case-management-system/issues/169) | Display EFRUP subruleref | CMS | ibadkhan088 | Retest | 🟢 Small | — |
| [#177](https://github.com/tazama-lf/case-management-system/issues/177) | Independent Variable shows 0 | CMS | RubaZehra | Retest | 🟢 Small | #146 |
| [#188](https://github.com/tazama-lf/case-management-system/issues/188) | Typologies displayed improperly | CMS | RubaZehra | Retest | 🟢 Small | #146 |
| [#202](https://github.com/tazama-lf/case-management-system/issues/202) | Investigator ID in Workload Report | CMS | WajahatAhmed-paysys | Internal Review | 🟢 Small | PR #194 |
| [#201](https://github.com/tazama-lf/case-management-system/issues/201) | User ID in Ageing Report exports | CMS | WajahatAhmed-paysys | Internal Review | 🟢 Small | PR #194 |
| [#204](https://github.com/tazama-lf/case-management-system/issues/204) | Tenant isolation in reference_id table | CMS | WajahatAhmed-paysys | Internal Review | 🟡 Medium | PR #194 |
| [#205](https://github.com/tazama-lf/case-management-system/issues/205) | Conditions view not showing data | CMS | ibadkhan088 | Todo | 🟡 Medium | Lakehouse data |
| [#209](https://github.com/tazama-lf/case-management-system/issues/209) | CMS Dashboard Review and Fixes | CMS | sobia-rizwan1567 | Todo | 🟡 Medium | Needs clarification |
| [#215](https://github.com/tazama-lf/case-management-system/issues/215) | Network analysis visualization errors | CMS | sobia-rizwan1567 | Todo | 🟡 Medium | Lakehouse data |
| [#210](https://github.com/tazama-lf/case-management-system/issues/210) | Add visualizations tab to case details | CMS | sobia-rizwan1567 | Todo | 🟡 Medium | — |
| [#140](https://github.com/tazama-lf/case-management-system/issues/140) | Aging report shows all-tenant data | CMS | sobia-rizwan1567 | Todo | 🔴 Large | Notebook parameterization |
| [#167](https://github.com/tazama-lf/case-management-system/issues/167) | Display subruleref reason + rule description | CMS | ibadkhan088 | Todo | 🔴 Large | BIAR ETL + CMS backend |
| [#152](https://github.com/tazama-lf/case-management-system/issues/152) | Dashboard links to assigned cases | CMS | Sandy-at-Tazama | Todo | 🟡 Medium | — |
| [#153](https://github.com/tazama-lf/case-management-system/issues/153) | Dashboard links to high priority alerts | CMS | Sandy-at-Tazama | Todo | 🟡 Medium | — |
| [#90](https://github.com/tazama-lf/biar/issues/90) | Alert History shows Unknown type | BIAR | MuneebPaysys | Todo | 🟢 Small | BIAR ETL re-ingest |
| [#208](https://github.com/tazama-lf/case-management-system/issues/208) | Selected saved filter not applied | CMS | ibadkhan088 | Todo | 🟡 Medium | — |
| [#211](https://github.com/tazama-lf/case-management-system/issues/211) | Add Assignee column + filter | CMS | RubaZehra | Todo | 🟡 Medium | #71 for display |
| [#212](https://github.com/tazama-lf/case-management-system/issues/212) | Search params for manual case creation | CMS | RubaZehra | Todo | 🟡 Medium | Needs clarification |
| [#213](https://github.com/tazama-lf/case-management-system/issues/213) | Auto-refresh Cases Dashboard | CMS | RubaZehra | Todo | 🟢 Small | — |

---

## Key Systemic Blockers

These two issues are blockers or root causes for multiple other issues:

### 1. User Name Persistence (#71, #142, #201, #202)
**No user names are stored in the CMS database.** They are fetched live from Keycloak but never persisted. This means report views, the BIAR dashboards, and any feature requiring a user's display name either shows a raw UUID or calls Keycloak on every request. Until a `users` table is created and populated on login, all four of these issues cannot be properly closed.

### 2. Alert Navigator Visualization (#146, #177, #188)
The alert navigator frontend rendering is broken — this blocks the retest of #177 and #188, which both depend on being able to see the full typology/rule data correctly.
