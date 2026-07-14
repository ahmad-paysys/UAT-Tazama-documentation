# Issue #95 — User Stories: BIAR Jupyter dashboards multi-tenancy interim fix

**Assigned developer:** 1 developer
**Tracks:** Track A (unbreak the four dashboards) → Track B (tenant selector widget)
**Merge order:** Track A ships independently. Track B follows after Track A is merged and verified.

---

## US-95-01 — Delete redundant tenant re-filter in Case Management Trend Dashboard

**Title:** Remove Case Management Trend cell 4 that hardcodes a tenant literal against a column that may not exist

**Body:**
`Case_Management_Trend_Dashboard.ipynb` cell 4 re-applies `df.tenant_id == 'TAZAMA'` on three DataFrames (`alerts`, `transactions`, `cases`) that cell 1 has already filtered via `load_tenant_hudi()`. When the underlying Hudi table uses a column named `tenantid` or `tenantId` instead of `tenant_id`, this line throws `AnalysisException: cannot resolve 'tenant_id'` and the dashboard stops. The fix is to delete cell 4 entirely — it's provably redundant.

**Scope (Track A):**
- File: `JupyterHub/notebooks/Case_Management_Trend_Dashboard.ipynb`
- Change: delete cell 4 (the three-line filter block starting `#Filters based on tenant ID for tables`)
- No schema migration, no frontend changes
- No downstream cells reference cell 4 output beyond the DataFrames it re-assigns

**Acceptance Criteria:**
- [ ] Cell 4 is removed from the notebook
- [ ] Notebook executes end-to-end without `AnalysisException`
- [ ] Case Management Trend charts render for the `TAZAMA` tenant
- [ ] Case counts match an independent SQL query for `TAZAMA`

**Testing:**
- Restart & Run All the notebook in the beta sandbox JupyterHub
- Compare the top-level case count against `SELECT count(*) FROM gold_cases WHERE <tenant-col> = 'TAZAMA'`

---

## US-95-02 — Extend the tenant-column allowlist to include camelCase variants

**Title:** Make `load_tenant_hudi()` resilient to camelCase tenant column names (`tenantId`, `txTenantId`)

**Body:**
Every dashboard defines `TENANT_FILTER_COLUMNS = ("tenant_id", "tx_tenant_id", "tenantid")`. Spark column names are case-sensitive, and some Hudi tables in the warehouse use camelCase (`tenantId`). When no allowlist entry matches, `load_tenant_hudi()` raises `ValueError`. Adding `"tenantId"` and `"txTenantId"` to the allowlist makes the helper robust to either naming convention without changing behaviour for tables that already match.

**Scope (Track A):**
- Files: `Case_Management_Trend_Dashboard.ipynb`, `Executive_Overview_Dashboard.ipynb`, `TMS_Performance_Dashboard.ipynb`, `Fraud_Trend_Analysis_Dashboard.ipynb`
- Change: extend `TENANT_FILTER_COLUMNS` to `("tenant_id", "tx_tenant_id", "tenantid", "tenantId", "txTenantId")` in the tenant-helper cell of each notebook
- No schema migration, no frontend changes

**Acceptance Criteria:**
- [ ] All four notebooks have the extended tuple
- [ ] `load_tenant_hudi()` succeeds against a table whose tenant column is `tenantId`
- [ ] No behaviour change for tables that already match snake_case

**Testing:**
- Point `WAREHOUSE_ROOT` at a Hudi dataset whose tenant column is `tenantId`; confirm all four notebooks load without ValueError
- Point at the standard beta-sandbox warehouse; confirm results unchanged

---

## US-95-03 — Stop swallowing tenant-load errors in Executive Overview

**Title:** Remove the `try/except` around the tenant load in Executive Overview so ValueErrors surface

**Body:**
`Executive_Overview_Dashboard.ipynb` cell 5 wraps the five `load_tenant_hudi()` calls in `try/except Exception as e: print(...)`. If the load raises (e.g. tenant column not in the allowlist), the exception becomes a print and downstream cells then die with `NameError` on `alerts`, `cases`, `transactions`, etc. — the visible symptom becomes worse than the real cause. Removing the `try/except` lets the ValueError propagate at the source, giving the operator a single clear failure to fix.

**Scope (Track A):**
- File: `Executive_Overview_Dashboard.ipynb`
- Change: unwrap the `try/except` block in cell 5 (keep the five load statements and the success print, drop the wrapping try/except)
- No schema migration, no frontend changes

**Acceptance Criteria:**
- [ ] Cell 5 no longer swallows exceptions
- [ ] When `load_tenant_hudi` succeeds (normal case), the notebook still prints `✓ All source tables loaded successfully`
- [ ] When `load_tenant_hudi` raises, execution stops at cell 5 with a clear ValueError — downstream cells do not run

**Testing:**
- Restart & Run All against the normal beta-sandbox warehouse; confirm success path
- Temporarily point at a table with no tenant column; confirm the ValueError shows the tenant-column allowlist and downstream cells don't execute

---

## US-95-04 — Add a tenant selector widget to the four erroring dashboards

**Title:** Replace hardcoded `TENANT_FILTER_VALUE = "TAZAMA"` with an `ipywidgets.Dropdown`

**Body:**
The issue's stated requirement is that operators can pick a tenant per dashboard. Today, the tenant literal is hardcoded in every notebook, so switching tenants means editing seven `.ipynb` files. This story introduces an `ipywidgets.Dropdown` immediately after the `load_tenant_hudi` definition in each of the four erroring notebooks. The dropdown is populated from a distinct-values scan of the tenant column on `gold/cases` at kernel startup (smallest table, cheapest scan). `load_tenant_hudi()` grows a `tenant` kwarg (default `DEFAULT_TENANT = "TAZAMA"`) and every call site threads the widget value in. Notebook markdown documents that this is a display-only control, not a security boundary.

Follow-up: the same widget rollout for `Case_Tracking_Analysis`, `Fraud_Typology_Effectiveness`, and `Anomaly_Detection_And_Rule_Calibration` is a separate ticket.

**Scope (Track B):**
- Files: `Case_Management_Trend_Dashboard.ipynb`, `Executive_Overview_Dashboard.ipynb`, `TMS_Performance_Dashboard.ipynb`, `Fraud_Trend_Analysis_Dashboard.ipynb`
- Change per notebook:
  1. Rename `TENANT_FILTER_VALUE` → `DEFAULT_TENANT` in the helper cell; add `tenant=DEFAULT_TENANT` kwarg to `load_tenant_hudi()`.
  2. Insert a new "Tenant selector" cell after the helper cell — build `tenant_widget = widgets.Dropdown(...)` from a distinct-values scan of `${WAREHOUSE_ROOT}/gold/cases`.
  3. Update every `load_tenant_hudi(path)` call site to `load_tenant_hudi(path, tenant=tenant_widget.value)`.
- No schema migration, no frontend changes
- Depends on US-95-02 (allowlist expansion) landing first

**Acceptance Criteria:**
- [ ] Each of the four notebooks shows a Dropdown widget populated with distinct tenant values from `gold/cases`
- [ ] Default selected value is `"TAZAMA"`; default run reproduces Track A results exactly
- [ ] Changing the dropdown and re-running downstream cells recomputes metrics for the selected tenant
- [ ] Selected value matches an independent SQL query: `SELECT count(*) FROM gold_cases WHERE <tenant-col> = '<selected>'`
- [ ] No hardcoded `"TAZAMA"` string literals remain in filter expressions in the four notebooks
- [ ] Notebook markdown includes a warning that the widget is display-only and not a security boundary

**Testing:**
- Restart & Run All with the default `TAZAMA`; confirm identical counts to Track A baseline
- Switch to a second known tenant, re-run all cells; confirm counts match the independent SQL check
- Switch to a tenant with zero rows; confirm the notebook renders "no data" indicators cleanly (no NameError, no ZeroDivisionError)

---

## US-95-05 — Open a follow-up issue for the three deferred notebooks

**Title:** Track a follow-up for widget rollout to Case_Tracking_Analysis, Fraud_Typology_Effectiveness, and Anomaly_Detection_And_Rule_Calibration

**Body:**
Track B in this issue is scoped to the four erroring notebooks per user direction. The other three dashboards share the same hardcoded `TENANT_FILTER_VALUE = "TAZAMA"` and would benefit from the same widget rollout. Rather than expanding this issue's scope, open a follow-up ticket that references #95 and re-uses the exact same US-95-04 pattern for the remaining three notebooks.

**Scope:**
- Not a code change in this issue — administrative only.
- Create a new GitHub issue on `tazama-lf/biar` titled "Extend BIAR dashboard tenant selector widget to remaining three notebooks (follow-up to #95)".

**Acceptance Criteria:**
- [ ] Follow-up issue opened
- [ ] Follow-up issue references #95 and lists the three notebooks by name
- [ ] Follow-up issue re-uses the US-95-04 acceptance criteria template

**Testing:**
- N/A — administrative story.
