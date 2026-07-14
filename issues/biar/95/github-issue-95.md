# GitHub Issue & PR Templates — Issue #95

---

## GitHub Issue Body

### Problem

Several BIAR Jupyter dashboards under `JupyterHub/notebooks/` either fail to execute or show data across every tenant, in contradiction of the interim multi-tenancy requirement:

- **Executive Overview Dashboard** raises a `tenantId` error and mentions the `transactions` table.
- **Case Management Trend Dashboard**, **TMS Performance Dashboard**, and **Fraud Trend Analysis Dashboard** all fail with a similar tenant-column error.
- **Case Tracking Analysis** and **Fraud Typology Effectiveness** screenshots in the beta sandbox render counts spanning every tenant.

The stated interim behaviour is that every dashboard should be scoped to a single tenant selected by the operator, until the full multi-tenancy scheme lands.

### Root Cause

Every dashboard notebook defines the same helper `load_tenant_hudi()` that filters Hudi tables by a tenant column whose name is drawn from a fixed allowlist: `("tenant_id", "tx_tenant_id", "tenantid")`. The allowlist misses the camelCase `tenantId` that some Hudi tables actually use, so `load_tenant_hudi()` raises `ValueError`. In `Executive_Overview_Dashboard.ipynb` that exception is swallowed by a `try/except` and downstream cells then die with `NameError`. In `Case_Management_Trend_Dashboard.ipynb` cell 4 additionally re-filters with a hardcoded `df.tenant_id == 'TAZAMA'` literal, which throws `AnalysisException` when the underlying column isn't named `tenant_id`. The `TENANT_FILTER_VALUE = "TAZAMA"` string is also hardcoded in all seven notebooks, so an operator can't switch tenants without editing every notebook.

### Impact

- Executive Overview, Case Management Trend, TMS Performance, and Fraud Trend Analysis dashboards are unusable in the beta sandbox.
- Where the filter is a silent no-op (Case Tracking Analysis, Fraud Typology Effectiveness), users see cross-tenant data — an interim multi-tenancy violation.
- Executive dashboards for the beta review are blocked.
- The reporter's "Was `transactions` renamed to `transaction`?" is a misread of the error string; no table rename occurred.

### Proposed Fix

Two tracks, sequenced.

**Track A — Unbreak the four erroring dashboards** *(interim, safe to ship alone)*

- Files: `Case_Management_Trend_Dashboard.ipynb`, `Executive_Overview_Dashboard.ipynb`, `TMS_Performance_Dashboard.ipynb`, `Fraud_Trend_Analysis_Dashboard.ipynb`.
- Delete redundant cell 4 in Case Management Trend; extend `TENANT_FILTER_COLUMNS` to include `"tenantId"` and `"txTenantId"`; remove the `try/except` in Executive Overview so tenant-load errors surface.
- No schema migration. No frontend. ~10 lines net change.
- Estimated effort: ~1 hour.

**Track B — Tenant selector widget** *(interim UX, follows Track A)*

- Same four notebooks.
- Replace the hardcoded `TENANT_FILTER_VALUE = "TAZAMA"` with an `ipywidgets.Dropdown` populated from a distinct-values scan of the tenant column at kernel startup; thread the widget value into every `load_tenant_hudi` call site.
- No schema migration. No frontend (JupyterHub renders `ipywidgets` natively).
- Estimated effort: ~3 hours.
- Rollout to the remaining three notebooks (Case_Tracking_Analysis, Fraud_Typology_Effectiveness, Anomaly_Detection_And_Rule_Calibration) is a follow-up ticket.

### Acceptance Criteria

- [ ] Case_Management_Trend_Dashboard.ipynb cell 4 removed
- [ ] `TENANT_FILTER_COLUMNS` in all four notebooks includes `"tenantId"` and `"txTenantId"`
- [ ] Executive_Overview_Dashboard.ipynb cell 5 no longer swallows exceptions
- [ ] All four notebooks execute end-to-end against the beta sandbox with no tenant errors
- [ ] Notebook KPIs match an independent SQL query for the `TAZAMA` tenant
- [ ] (Track B) Each of the four notebooks exposes an `ipywidgets.Dropdown` populated with distinct tenant values
- [ ] (Track B) Changing the dropdown and re-running downstream cells recomputes metrics for the selected tenant
- [ ] (Track B) Default value is `"TAZAMA"`; no hardcoded `"TAZAMA"` literals remain in filter expressions in the four notebooks
- [ ] Follow-up issue opened for Case_Tracking_Analysis, Fraud_Typology_Effectiveness, and Anomaly_Detection_And_Rule_Calibration

---

## PR Descriptions

### Track A PR

**PR Title:** fix: unbreak four BIAR dashboards blocked by tenant filter (#95)

**PR Body:**

- **Summary**
  - Delete redundant cell 4 in `Case_Management_Trend_Dashboard.ipynb` that re-filters with a hardcoded literal against a column that may not exist.
  - Extend `TENANT_FILTER_COLUMNS` in all four erroring notebooks to include camelCase `"tenantId"` and `"txTenantId"` so `load_tenant_hudi()` matches tables that use camelCase.
  - Remove the `try/except` around the tenant load in `Executive_Overview_Dashboard.ipynb` cell 5 so ValueErrors surface immediately instead of turning into downstream `NameError`s.
  - No schema migration. No frontend. No downtime.

- **Test Plan**
  - [ ] Open each of the four notebooks in the beta sandbox JupyterHub and run "Restart & Run All"
  - [ ] Confirm no `AnalysisException`, no `NameError`, no swallowed `"✗ Error loading tables"` print
  - [ ] Verify total case count on the Executive Overview matches `SELECT count(*) FROM gold_cases WHERE <tenant-col> = 'TAZAMA'`
  - [ ] Confirm Case Management Trend renders trend charts end-to-end

- **Related Issue:** https://github.com/tazama-lf/biar/issues/95

### Track B PR

**PR Title:** feat: add tenant selector widget to four BIAR dashboards (#95)

**PR Body:**

- **Summary**
  - Replace hardcoded `TENANT_FILTER_VALUE = "TAZAMA"` with an `ipywidgets.Dropdown` in the four Track A notebooks.
  - Populate the dropdown from a distinct-values scan of the tenant column on `gold/cases` at kernel startup, with `"TAZAMA"` as the default option.
  - `load_tenant_hudi()` grows a `tenant` kwarg; all call sites thread `tenant=tenant_widget.value`.
  - Notebook markdown documents that this is a display-only control, not a security boundary.

- **Test Plan**
  - [ ] Restart & Run All each notebook with the default `TAZAMA` selection; confirm results match the Track A baseline
  - [ ] Select a second tenant from the dropdown, re-run all cells; confirm counts change and match `SELECT count(*) ... WHERE <tenant-col> = '<selected>'`
  - [ ] Select a tenant with zero rows; confirm the dashboard renders with visible "no data" indicators, not a crash
  - [ ] Verify markdown warning next to the widget is visible

- **Related Issue:** https://github.com/tazama-lf/biar/issues/95

- **Migration:** None. No schema changes.

- **Sequencing Note:** Requires the Track A PR to be merged first. Track A adds `"tenantId"` / `"txTenantId"` to the allowlist and removes the redundant Cell 4 that Track B otherwise conflicts with.
