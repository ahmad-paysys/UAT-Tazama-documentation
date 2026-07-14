# Impact Study — Issue #95: BIAR bug: multi-tenancy interim fix - add tenant filter to each Jupyter notebook dashboard

**Issue:** [#95](https://github.com/tazama-lf/biar/issues/95)
**Study Date:** 2026-07-13
**Author:** Ahmad Khalid
**Repository:** tazama-lf/biar

---

## Summary

Four BIAR Jupyter dashboards (Executive Overview, Case Management Trend, TMS Performance, Fraud Trend Analysis) either error out or silently render cross-tenant data because the interim tenant filter helper `load_tenant_hudi()` matches a fixed snake_case allowlist and Case Management Trend duplicates the filter with a hardcoded literal that fails when the tenant column isn't named `tenant_id`. Track A deletes the redundant filter, expands the allowlist to include camelCase variants, and stops swallowing the `load_tenant_hudi` ValueError in Executive Overview. Track B replaces the hardcoded `TENANT_FILTER_VALUE = "TAZAMA"` with an `ipywidgets` dropdown across the four erroring notebooks so operators can pick a tenant at runtime.

---

## Confirmed Root Cause

- `TENANT_FILTER_COLUMNS = ("tenant_id", "tx_tenant_id", "tenantid")` is defined identically in seven notebooks and does not include the camelCase `tenantId` — confirmed in [Executive_Overview_Dashboard.ipynb](repos/biar/JupyterHub/notebooks/Executive_Overview_Dashboard.ipynb) cell 3, [Case_Management_Trend_Dashboard.ipynb](repos/biar/JupyterHub/notebooks/Case_Management_Trend_Dashboard.ipynb) cell 1, [TMS_Performance_Dashboard.ipynb](repos/biar/JupyterHub/notebooks/TMS_Performance_Dashboard.ipynb) cell 2, [Fraud_Trend_Analysis_Dashboard.ipynb](repos/biar/JupyterHub/notebooks/Fraud_Trend_Analysis_Dashboard.ipynb) cell 1.
- Case Management Trend cell 4 re-applies `.filter(df.tenant_id == 'TAZAMA')` to `alerts`, `transactions`, `cases` — the DataFrames were already filtered in cell 1, and the `alerts` / `cases` variables carry the raw Hudi column name which may not be `tenant_id`.
- Executive Overview cell 5 wraps the tenant load in `try/except Exception as e: print(...)` — the exception is swallowed, so downstream cells raise `NameError` on `alerts`, `cases`, `transactions`.
- Every notebook resolves the transactions table as `f"{WAREHOUSE_ROOT}/gold/transactions"` (plural). No notebook references `gold/transaction` singular. The reporter's "Was it renamed `transaction`" comment is a misinterpretation of the surfaced ValueError string, not a real table rename.
- `TENANT_FILTER_VALUE = "TAZAMA"` is hardcoded in all seven notebooks; there is no widget, environment variable, or config entry.

---

## Track A — Minimal fix: unbreak the four dashboards

### What Changes

Four notebooks. Delete one cell in one notebook, extend one tuple in all four, unwrap one `try/except` in one notebook. ~10 lines of net change.

### Impact

| Aspect | Value |
|---|---|
| Files changed | 4 |
| Schema migration required | No |
| Frontend changes required | No |
| Downtime required | No (notebook re-run only) |
| Risk of regression | Low — filter allowlist is additive; deleted cell 4 is provably redundant |
| Reversibility | Trivial (revert commit) |

What Track A fixes immediately:
- Executive Overview, Case Management Trend, TMS Performance, Fraud Trend Analysis all execute end-to-end.
- Tenant filter succeeds against Hudi tables that use camelCase (`tenantId`) column naming.
- Tenant-load failures no longer silently propagate as `NameError` downstream.

What Track A does not fix:
- `TENANT_FILTER_VALUE` remains hardcoded to `"TAZAMA"` — operators still cannot switch tenants without editing the notebook.
- Case_Tracking_Analysis, Fraud_Typology_Effectiveness, and Anomaly_Detection_And_Rule_Calibration are unchanged (they share the same hardcoding but are not in the four-notebook fix scope per user direction).
- The filter remains a display-only control; it is not a security boundary and can be trivially edited by a notebook user.

Track A is safe to ship in isolation.

---

## Track B — Tenant selector widget across the four erroring notebooks

### What Changes

Replace the hardcoded `TENANT_FILTER_VALUE = "TAZAMA"` with an `ipywidgets.Dropdown` (populated from a distinct-values scan of the tenant column) in each of the four erroring notebooks. `load_tenant_hudi()` gains a `tenant` kwarg so the value is threaded explicitly.

### Schema Impact

None. No new tables, no new columns, no migrations.

### Backend Code Impact

| File | Change |
|---|---|
| [Case_Management_Trend_Dashboard.ipynb](repos/biar/JupyterHub/notebooks/Case_Management_Trend_Dashboard.ipynb) | Add tenant widget cell after cell 1; pass widget value into `load_tenant_hudi`. |
| [Executive_Overview_Dashboard.ipynb](repos/biar/JupyterHub/notebooks/Executive_Overview_Dashboard.ipynb) | Add tenant widget cell after cell 3; pass widget value into `load_tenant_hudi`. |
| [TMS_Performance_Dashboard.ipynb](repos/biar/JupyterHub/notebooks/TMS_Performance_Dashboard.ipynb) | Add tenant widget cell after cell 2; pass widget value into `load_tenant_hudi`. |
| [Fraud_Trend_Analysis_Dashboard.ipynb](repos/biar/JupyterHub/notebooks/Fraud_Trend_Analysis_Dashboard.ipynb) | Add tenant widget cell after cell 1; pass widget value into `load_tenant_hudi`. |

No new files. No enum sweeps. No reference sweeps.

### Frontend Code Impact

None. `ipywidgets` are rendered natively by JupyterHub — the notebooks are the "frontend" here.

---

## Side Effects and Risks

### Risks of Track A Alone

| Risk | Likelihood | Mitigation |
|---|---|---|
| Removing Case_Management_Trend cell 4 leaves a stale markdown reference | Very Low | Grep the notebook for "Filters based on tenant ID" — nothing references it |
| Unwrapping try/except in Executive Overview causes the notebook to hard-fail if the tenant column is missing | Low (that's the intended behaviour) | The failure is now visible and the operator can extend `TENANT_FILTER_COLUMNS` or investigate; silent failure was worse |
| Adding `"tenantId"` to the allowlist could match an unrelated column in an unexpected table | Very Low | The allowlist match is per-table at load time; each Hudi table only has one tenant column |

### Risks of Track B

| Risk | Likelihood | Mitigation |
|---|---|---|
| Distinct-values scan of tenant column is expensive on large tables | Medium | Cache the distinct list against the smallest table (`gold/cases`, usually the smallest); document expected latency |
| Users forget to re-execute downstream cells after changing the dropdown | High (Jupyter UX) | Add a markdown warning next to the widget; consider `interact()`-style rebind for a future iteration |
| Widget value collision with the same `TENANT_FILTER_VALUE` name in seven notebooks in the same JupyterHub session | Low | Each notebook has its own Python kernel; no cross-contamination |
| Operator selects a tenant with zero rows and misreads empty dashboards as broken | Medium | Show row-count summary immediately after load; document expected behaviour |

### Cross-Issue Dependencies

Scanned the [issues/biar/](issues/biar/) directory. Existing entries `108`, `109`, `110`, `111` were reviewed for overlap. None of them touch [JupyterHub/notebooks/](repos/biar/JupyterHub/notebooks/) — they cover other subsystems within biar (see `impact-108-111.md`). No merge conflict risk with any known open issue.

---

## Effort Estimate

| Track | Files | Effort |
|---|---|---|
| Track A | 4 notebooks | ~1 hour (including smoke tests) |
| Track B | 4 notebooks | ~3 hours (including end-to-end two-tenant test) |
| **Total** | **4 notebooks touched** | **~4 hours** |

---

## Acceptance Criteria (Verification Checklist)

### Track A

- [ ] [Case_Management_Trend_Dashboard.ipynb](repos/biar/JupyterHub/notebooks/Case_Management_Trend_Dashboard.ipynb) cell 4 removed
- [ ] `TENANT_FILTER_COLUMNS` in all four notebooks includes `"tenantId"` and `"txTenantId"`
- [ ] [Executive_Overview_Dashboard.ipynb](repos/biar/JupyterHub/notebooks/Executive_Overview_Dashboard.ipynb) cell 5 no longer swallows exceptions
- [ ] All four notebooks execute end-to-end against the beta sandbox without tenant errors
- [ ] Counts on the Executive Overview and Case Management Trend dashboards equal the row count for `TAZAMA` in `gold/cases` (verified with a separate SQL query)

### Track B

- [ ] Each of the four notebooks exposes an `ipywidgets.Dropdown` populated with distinct tenant values
- [ ] Changing the dropdown and re-running downstream cells recomputes metrics for the new tenant
- [ ] Default value is `"TAZAMA"`
- [ ] No hardcoded `"TAZAMA"` literals remain in filter expressions in the four notebooks

---

## Recommended Sequencing

1. Ship Track A as one PR against the biar `dev` branch. It has no dependencies and unblocks the four erroring dashboards immediately.
2. Once Track A is merged and verified in the sandbox, ship Track B as a second PR against `dev`.
3. Open a follow-up issue for Case_Tracking_Analysis, Fraud_Typology_Effectiveness, and Anomaly_Detection_And_Rule_Calibration — same widget rollout, deferred per user direction.
4. Longer-term: replace the interim filter with a proper JupyterHub per-user-tenant authentication scheme (out of scope here; belongs to the full multi-tenancy epic).
