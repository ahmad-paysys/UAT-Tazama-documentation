# Issue #95 — BIAR bug: multi-tenancy interim fix - add tenant filter to each Jupyter notebook dashboard

**Repository:** tazama-lf/biar
**Issue:** [BIAR bug: multi-tenancy interim fix - add tenant filter to each Jupyter notebook dashboard](https://github.com/tazama-lf/biar/issues/95)
**Author:** Sandy-at-Tazama
**State:** Open
**Report Date:** 2026-07-13

---

## Executive Summary

The BIAR Jupyter notebook dashboards under `JupyterHub/notebooks/` are supposed to be scoped to a single tenant as an interim multi-tenancy control. A shared helper `load_tenant_hudi()` already exists in every dashboard notebook to filter Hudi tables by tenant column at load time, using a hardcoded `TENANT_FILTER_VALUE = "TAZAMA"`. In practice, three symptoms are visible in the beta sandbox:

1. **Executive Overview Dashboard** raises a `tenantId` error and mentions the `transactions` table (screenshot from the issue). This is caused by the load block being wrapped in `try/except` that swallows the exception; downstream cells then run against undefined symbols and crash later, and the reporter interprets the surfaced message as "tenantId error, transactions table".
2. **Case Management Trend Dashboard, TMS Performance Dashboard, Fraud Trend Analysis Dashboard** all display an `AnalysisException`-style error. In Case Management Trend the culprit is a leftover Cell 4 that re-filters with a hardcoded literal (`alerts.tenant_id == 'TAZAMA'`) after Cell 1 has already applied `load_tenant_hudi()`. When the underlying Hudi table's tenant column name differs from `tenant_id` (e.g. `tx_tenant_id` or `tenantid`), this line throws because `tenant_id` isn't a column of the DataFrame post-load. The other two dashboards fail for the analogous case-sensitivity / column-name mismatch inside `load_tenant_hudi()` itself.
3. **Case Tracking Analysis, Executive Overview and Fraud Typology Effectiveness** screenshots show counts across all tenants — this is the same class of bug: the tenant filter silently degrades (either the wrong column name is picked up or the filter value doesn't match the actual tenant id string stored in the data).

Track A is a one-line delete plus a column-set expansion inside the existing helper — safe to ship immediately and unblocks all four erroring dashboards. Track B replaces the hardcoded tenant with an `ipywidgets` dropdown so the interim filter is user-selectable (per the issue's stated requirement), scoped to the four erroring notebooks per user direction.

---

## How It Works Today — Confirmed in Code

### 1. Shared `load_tenant_hudi()` helper

Every dashboard notebook defines the same helper, with the same tenant-column allowlist. Example from [Executive_Overview_Dashboard.ipynb](repos/biar/JupyterHub/notebooks/Executive_Overview_Dashboard.ipynb) cell 3:

```python
TENANT_FILTER_VALUE = "TAZAMA"
TENANT_FILTER_COLUMNS = ("tenant_id", "tx_tenant_id", "tenantid")

def load_tenant_hudi(path, **options):
    reader = spark.read.format("hudi")
    for key, value in options.items():
        reader = reader.option(key, value)
    df = reader.load(path)
    tenant_col = next((c for c in TENANT_FILTER_COLUMNS if c in df.columns), None)
    if tenant_col is None:
        raise ValueError(
            f"Hudi table at {path} has no tenant filter column; "
            f"expected one of {TENANT_FILTER_COLUMNS}"
        )
    return df.filter(df[tenant_col] == TENANT_FILTER_VALUE)
```

Confirmed identical (verbatim) in:
- [Case_Management_Trend_Dashboard.ipynb](repos/biar/JupyterHub/notebooks/Case_Management_Trend_Dashboard.ipynb) cell 1 (inline with the SparkSession)
- [Case_Tracking_Analysis_Dashboard.ipynb](repos/biar/JupyterHub/notebooks/Case_Tracking_Analysis_Dashboard.ipynb) cell 2
- [Executive_Overview_Dashboard.ipynb](repos/biar/JupyterHub/notebooks/Executive_Overview_Dashboard.ipynb) cell 3
- [Fraud_Trend_Analysis_Dashboard.ipynb](repos/biar/JupyterHub/notebooks/Fraud_Trend_Analysis_Dashboard.ipynb) cell 1
- [Fraud_Typology_Effectiveness_Dashboard.ipynb](repos/biar/JupyterHub/notebooks/Fraud_Typology_Effectiveness_Dashboard.ipynb) cell 2
- [TMS_Performance_Dashboard.ipynb](repos/biar/JupyterHub/notebooks/TMS_Performance_Dashboard.ipynb) cell 2
- [Anomaly_Detection_And_Rule_Calibration.ipynb](repos/biar/JupyterHub/notebooks/Anomaly_Detection_And_Rule_Calibration.ipynb) cell 4

**The `TENANT_FILTER_COLUMNS` tuple only contains snake_case / lowercase variants.** It does **not** include camelCase `tenantId`, which is the natural Ozone/BIAR ingestion column casing (see `normalize_transactions_for_dashboard` in the same file where the alias explicitly maps `tenantid` → `tenant_id`, implying the raw column is `tenantid` in some tables but may be `tenantId` in others). Spark column names are case-sensitive by default; a table where the column is spelled `tenantId` will not match any of the three allowed names, and `load_tenant_hudi()` raises `ValueError(...)`.

### 2. Case Management Trend Dashboard — redundant Cell 4

[Case_Management_Trend_Dashboard.ipynb](repos/biar/JupyterHub/notebooks/Case_Management_Trend_Dashboard.ipynb) cell 4:

```python
#Filters based on tenant ID for tables

alerts = alerts.filter(alerts.tenant_id == 'TAZAMA')
transactions = transactions.filter(transactions.tenant_id == 'TAZAMA')
cases = cases.filter(cases.tenant_id == 'TAZAMA')
```

- The three DataFrames referenced are already tenant-filtered in cell 1 via `load_tenant_hudi()`.
- `transactions` in cell 1 is passed through `normalize_transactions_for_dashboard()` which always yields a column named `tenant_id`, so `transactions.tenant_id` succeeds.
- `alerts` and `cases` in cell 1 are the *raw* Hudi DataFrames — whatever tenant column they carry, it may or may not be named `tenant_id`. If the alerts Hudi table uses `tenantid` (or `tenantId`), then `alerts.tenant_id` raises `AnalysisException: cannot resolve 'tenant_id'` at cell 4. This matches the reporter's screenshot showing the Case Management Trend dashboard "not currently executing due to an error".

### 3. Executive Overview Dashboard — try/except that swallows tenant errors

[Executive_Overview_Dashboard.ipynb](repos/biar/JupyterHub/notebooks/Executive_Overview_Dashboard.ipynb) cell 5:

```python
try:
    alerts = load_tenant_hudi(gold_alerts_path)
    cases = load_tenant_hudi(cases_gold_path)
    tasks = load_tenant_hudi(tasks_gold_path)
    transactions = normalize_transactions_for_dashboard(load_tenant_hudi(transactions_gold_path))
    evaluation = load_tenant_hudi(evaluation_gold_path)
    print("✓ All source tables loaded successfully")
except Exception as e:
    print(f"✗ Error loading tables: {e}")
```

- If `load_tenant_hudi()` raises `ValueError("Hudi table at .../gold/transactions has no tenant filter column; expected one of ('tenant_id', 'tx_tenant_id', 'tenantid')")`, the `except` prints it and moves on — `alerts`, `cases`, `transactions`, etc. are then `NameError` in every subsequent cell. The reporter's first screenshot shows a `tenantId` error and mentions `transactions`; this is that surfaced ValueError.

### 4. Other dashboards — same helper, no downstream re-filter

TMS Performance Dashboard, Fraud Trend Analysis Dashboard, Case Tracking Analysis Dashboard, Fraud Typology Effectiveness Dashboard all rely on `load_tenant_hudi()` alone. If the helper raises (column-name mismatch), the notebook stops there. If the helper returns an empty DataFrame (right column, wrong value — the actual tenant id is `"tenant-001"` and not `"TAZAMA"`), the dashboard runs cleanly but with zero rows, which contradicts the reporter's screenshots showing all-tenant counts. That leaves one plausible remaining failure mode: the tenant column on some tables carries a raw value that happens to match `"TAZAMA"` for every row (e.g. every row was ingested under a placeholder tenant), so the filter is a no-op — which is exactly the "all tenants visible" symptom.

The `TENANT_FILTER_VALUE = "TAZAMA"` string is hardcoded in seven places. There is no widget, environment variable, or config file that lets an operator change it without editing every notebook.

### 5. `transactions` vs `transaction` table name

Every notebook reads the same path: `f"{WAREHOUSE_ROOT}/gold/transactions"` (plural). No notebook references `/gold/transaction` (singular). The reporter's comment "Was it renamed `transaction`" is a red herring — the error text likely contains the string `transactions` because that is the table path in the ValueError message, not because the table was renamed.

---

## Root Cause Analysis

The tenant interim filter is best-effort, not enforced. `load_tenant_hudi()` matches a fixed allowlist of column names and applies a hardcoded literal, then Case_Management_Trend duplicates the filter with an even narrower assumption (`tenant_id` column exists on the raw DataFrame). When the actual Hudi tables use a column name outside the allowlist (`tenantId` in camelCase, most likely), the helper raises and — in Executive Overview — that exception is swallowed by a `try/except`, causing downstream `NameError`s. Where the tenant column *is* present but every row already has the same value, the filter becomes a no-op and the dashboards render all-tenant counts.

Downstream symptoms:
- Executive Overview: reported `tenantId` error and unresolved `transactions` symbol → swallowed `load_tenant_hudi` ValueError.
- Case Management Trend, TMS Performance, Fraud Trend: `AnalysisException` when the redundant `.filter(df.tenant_id == 'TAZAMA')` is evaluated on a DataFrame that doesn't carry `tenant_id`.
- Case Tracking Analysis, Fraud Typology Effectiveness (screenshots show all-tenant counts): filter succeeds but is a no-op because every row's tenant column already equals the literal.

---

## Blast Radius — Full File Inventory

### Files that must change (Track A + Track B)

| File | What's affected | Lines / cells |
|---|---|---|
| [Case_Management_Trend_Dashboard.ipynb](repos/biar/JupyterHub/notebooks/Case_Management_Trend_Dashboard.ipynb) | Redundant re-filter in cell 4; `TENANT_FILTER_COLUMNS` missing camelCase variants in cell 1 | cell 1, cell 4 |
| [Executive_Overview_Dashboard.ipynb](repos/biar/JupyterHub/notebooks/Executive_Overview_Dashboard.ipynb) | try/except silently swallows `load_tenant_hudi` failures; `TENANT_FILTER_COLUMNS` missing camelCase | cell 3, cell 5 |
| [TMS_Performance_Dashboard.ipynb](repos/biar/JupyterHub/notebooks/TMS_Performance_Dashboard.ipynb) | `TENANT_FILTER_COLUMNS` missing camelCase | cell 2 |
| [Fraud_Trend_Analysis_Dashboard.ipynb](repos/biar/JupyterHub/notebooks/Fraud_Trend_Analysis_Dashboard.ipynb) | `TENANT_FILTER_COLUMNS` missing camelCase | cell 1 |

### Files indirectly affected (Track B widget rollout — deferred per user)

| File | What's affected | Lines / cells |
|---|---|---|
| [Case_Tracking_Analysis_Dashboard.ipynb](repos/biar/JupyterHub/notebooks/Case_Tracking_Analysis_Dashboard.ipynb) | Same hardcoded `TENANT_FILTER_VALUE`; not part of the four erroring dashboards but shares the shortcoming | cell 2 |
| [Fraud_Typology_Effectiveness_Dashboard.ipynb](repos/biar/JupyterHub/notebooks/Fraud_Typology_Effectiveness_Dashboard.ipynb) | Same hardcoded `TENANT_FILTER_VALUE` | cell 2 |
| [Anomaly_Detection_And_Rule_Calibration.ipynb](repos/biar/JupyterHub/notebooks/Anomaly_Detection_And_Rule_Calibration.ipynb) | Same hardcoded `TENANT_FILTER_VALUE` | cell 4 |

---

## Side-Effect Map

| Symptom if left unfixed | Consequence |
|---|---|
| Executive Overview dashboard raises tenantId error | Executive users cannot view the primary case management KPIs |
| Case Management Trend dashboard cell 4 AnalysisException | Trend analytics unavailable for supervisors; screenshots in the issue confirm the failure |
| TMS Performance / Fraud Trend Analysis dashboards fail identically | TMS SLA reporting and fraud trend evaluation are blocked |
| Where the filter is a silent no-op | Users see cross-tenant data — a multi-tenancy compliance violation for the interim period |
| Hardcoded `TENANT_FILTER_VALUE = "TAZAMA"` in every notebook | Operators cannot switch tenants without editing each `.ipynb` — no way to demo alternate tenants in beta sandbox |

---

## Effort Assessment

### Track A — Minimal fix: unbreak the four dashboards

- Delete cell 4 in [Case_Management_Trend_Dashboard.ipynb](repos/biar/JupyterHub/notebooks/Case_Management_Trend_Dashboard.ipynb) (redundant re-filter that duplicates cell 1's filter and asserts a column name that may not exist).
- Expand `TENANT_FILTER_COLUMNS` in all four erroring notebooks to include the camelCase variants: `("tenant_id", "tx_tenant_id", "tenantid", "tenantId", "txTenantId")`.
- Remove the `try/except` wrapper in Executive Overview cell 5 so tenant-load failures surface loudly instead of leaving downstream cells with undefined names.
- No schema migration. No frontend changes.
- ~4 files touched, ~10 lines of net change.

| Step | File | Effort |
|---|---|---|
| A1 | Case_Management_Trend_Dashboard.ipynb — delete cell 4 | 5 min |
| A2 | Case_Management_Trend_Dashboard.ipynb — extend TENANT_FILTER_COLUMNS | 5 min |
| A3 | Executive_Overview_Dashboard.ipynb — extend TENANT_FILTER_COLUMNS; unwrap try/except | 10 min |
| A4 | TMS_Performance_Dashboard.ipynb — extend TENANT_FILTER_COLUMNS | 5 min |
| A5 | Fraud_Trend_Analysis_Dashboard.ipynb — extend TENANT_FILTER_COLUMNS | 5 min |
| A6 | Manual smoke test each notebook in JupyterHub | 30 min |

Track A total: ~1 hour.

### Track B — Tenant selector widget (four erroring notebooks only, per user)

- Introduce an `ipywidgets.Dropdown` (or `Text`) tenant selector in a new "config" cell placed immediately after the `load_tenant_hudi` definition.
- `TENANT_FILTER_VALUE` becomes the widget's `.value` and is passed into `load_tenant_hudi` as a parameter (helper signature grows a `tenant` kwarg).
- Tenant options populated from a distinct-values scan of the tenant column at notebook startup, with `"TAZAMA"` as the default.
- Applies to: Case_Management_Trend, Executive_Overview, TMS_Performance, Fraud_Trend_Analysis. The other three notebooks are deferred to a follow-up.
- No schema migration. No frontend changes (JupyterHub renders `ipywidgets` natively).

| Step | File | Effort |
|---|---|---|
| B1 | Case_Management_Trend_Dashboard.ipynb — add widget, thread tenant kwarg | 30 min |
| B2 | Executive_Overview_Dashboard.ipynb — add widget, thread tenant kwarg | 30 min |
| B3 | TMS_Performance_Dashboard.ipynb — add widget, thread tenant kwarg | 30 min |
| B4 | Fraud_Trend_Analysis_Dashboard.ipynb — add widget, thread tenant kwarg | 30 min |
| B5 | End-to-end test with two distinct tenants in the sandbox | 1 hour |

Track B total: ~3 hours (four erroring notebooks only).

### Recommended sequencing

1. **Track A merges first** — it unblocks all four dashboards independently, has essentially no risk, and does not depend on Track B.
2. **Track B follows once A is verified** — the widget requires a tenant column of consistent naming, which Track A guarantees via the expanded allowlist.
3. Extending Track B to the remaining three notebooks (Case_Tracking_Analysis, Fraud_Typology_Effectiveness, Anomaly_Detection_And_Rule_Calibration) is a follow-up ticket, not part of this fix.

---

## Acceptance Criteria

### Track A

- [ ] Cell 4 in Case_Management_Trend_Dashboard.ipynb removed
- [ ] `TENANT_FILTER_COLUMNS` in all four notebooks includes `"tenantId"` and `"txTenantId"` in addition to the existing snake_case variants
- [ ] Executive_Overview_Dashboard.ipynb cell 5 no longer wraps the tenant load in `try/except` (or the except re-raises after logging)
- [ ] All four notebooks execute end-to-end against the beta sandbox without `tenantId` / `AnalysisException` errors
- [ ] Counts in the Executive Overview and Case Management Trend dashboards match the count of rows for the `TAZAMA` tenant only (verified with a separate SQL query against the Hudi warehouse)

### Track B (four erroring notebooks only)

- [ ] Each of the four notebooks exposes an `ipywidgets.Dropdown` with tenant options loaded from the data
- [ ] Selecting a different tenant re-executes downstream cells against that tenant's data
- [ ] Default value is `"TAZAMA"` to preserve existing behaviour
- [ ] Notebook cells downstream of the widget receive the selected tenant (no hardcoded literals left)
- [ ] Documented in the notebook markdown that the filter is a display-only interim control, not a security boundary

---
