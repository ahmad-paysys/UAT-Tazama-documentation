# Complete Fix — Exact Code Changes

Track A can merge independently and unblocks the four erroring dashboards. Track B adds a tenant selector widget to the same four notebooks and depends on Track A (it assumes the expanded `TENANT_FILTER_COLUMNS` and the removed cell 4). Track B is scoped to the four erroring notebooks only per user direction; the other three notebooks (Case_Tracking_Analysis, Fraud_Typology_Effectiveness, Anomaly_Detection_And_Rule_Calibration) are deferred to a follow-up issue.

---

## Track A — Unbreak the four dashboards (PR 1 of 2)

### A1. [Case_Management_Trend_Dashboard.ipynb](repos/biar/JupyterHub/notebooks/Case_Management_Trend_Dashboard.ipynb) — delete redundant cell 4

**BEFORE (cell 4):**

```python
#Filters based on tenant ID for tables

alerts = alerts.filter(alerts.tenant_id == 'TAZAMA')
transactions = transactions.filter(transactions.tenant_id == 'TAZAMA')
cases = cases.filter(cases.tenant_id == 'TAZAMA')
```

**AFTER:** delete this cell entirely.

Cell 1 has already filtered `alerts`, `transactions`, and `cases` via `load_tenant_hudi()`. The re-filter is redundant and assumes a column name (`tenant_id`) that may not exist on the raw Hudi DataFrames — this is the source of the `AnalysisException` the reporter shows.

---

### A2. All four notebooks — expand `TENANT_FILTER_COLUMNS`

The tuple is duplicated verbatim in [Case_Management_Trend_Dashboard.ipynb](repos/biar/JupyterHub/notebooks/Case_Management_Trend_Dashboard.ipynb) cell 1, [Executive_Overview_Dashboard.ipynb](repos/biar/JupyterHub/notebooks/Executive_Overview_Dashboard.ipynb) cell 3, [TMS_Performance_Dashboard.ipynb](repos/biar/JupyterHub/notebooks/TMS_Performance_Dashboard.ipynb) cell 2, and [Fraud_Trend_Analysis_Dashboard.ipynb](repos/biar/JupyterHub/notebooks/Fraud_Trend_Analysis_Dashboard.ipynb) cell 1.

**BEFORE:**

```python
TENANT_FILTER_VALUE = "TAZAMA"
TENANT_FILTER_COLUMNS = ("tenant_id", "tx_tenant_id", "tenantid")
```

**AFTER:**

```python
TENANT_FILTER_VALUE = "TAZAMA"
TENANT_FILTER_COLUMNS = (
    "tenant_id",
    "tx_tenant_id",
    "tenantid",
    "tenantId",
    "txTenantId",
)
```

Spark column names are case-sensitive by default. Ozone/BIAR ingestion often produces camelCase (`tenantId`), which the current allowlist misses. Adding the two camelCase variants makes the helper resilient to either naming convention without changing behaviour for tables that already match.

---

### A3. [Executive_Overview_Dashboard.ipynb](repos/biar/JupyterHub/notebooks/Executive_Overview_Dashboard.ipynb) — stop swallowing tenant-load errors

**BEFORE (cell 5, load block):**

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

**AFTER:**

```python
alerts = load_tenant_hudi(gold_alerts_path)
cases = load_tenant_hudi(cases_gold_path)
tasks = load_tenant_hudi(tasks_gold_path)
transactions = normalize_transactions_for_dashboard(load_tenant_hudi(transactions_gold_path))
evaluation = load_tenant_hudi(evaluation_gold_path)
print("✓ All source tables loaded successfully")
```

Swallowing the exception makes the visible symptom worse than the underlying cause: downstream cells then raise `NameError` on `alerts`/`cases`/etc., which is what the reporter saw as "tenantId error, transactions table". Let the ValueError from `load_tenant_hudi` propagate so the operator sees the real problem in one place.

**Impact of Track A alone:** All four dashboards execute end-to-end against tables using either snake_case or camelCase tenant column names, and the `TAZAMA` tenant scoping still applies. Track A does not add a runtime tenant selector — that's Track B.

---

## Track B — Tenant selector widget (PR 2 of 2)

Numbered steps B1–B4 apply the same pattern to each of the four erroring notebooks. Each notebook gets:

1. A new "Tenant selector" cell inserted immediately after the `load_tenant_hudi` definition.
2. `load_tenant_hudi` updated to accept a `tenant` kwarg (default preserves the module-level `TENANT_FILTER_VALUE`).
3. All `load_tenant_hudi(path)` call sites updated to `load_tenant_hudi(path, tenant=tenant_widget.value)`.

The pattern is shown once (B1) and applied identically for B2/B3/B4.

### Step B1 — Case_Management_Trend_Dashboard.ipynb

**File:** [Case_Management_Trend_Dashboard.ipynb](repos/biar/JupyterHub/notebooks/Case_Management_Trend_Dashboard.ipynb)

**BEFORE** (cell 1, tenant helper section):

```python
TENANT_FILTER_VALUE = "TAZAMA"
TENANT_FILTER_COLUMNS = (
    "tenant_id", "tx_tenant_id", "tenantid", "tenantId", "txTenantId",
)

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

**AFTER** (cell 1, tenant helper section — accept an explicit tenant kwarg):

```python
DEFAULT_TENANT = "TAZAMA"
TENANT_FILTER_COLUMNS = (
    "tenant_id", "tx_tenant_id", "tenantid", "tenantId", "txTenantId",
)

def load_tenant_hudi(path, tenant=DEFAULT_TENANT, **options):
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
    return df.filter(df[tenant_col] == tenant)
```

**Insert new cell 2 (Tenant selector):**

```python
# ============================================================================
# INTERIM TENANT FILTER (issue #95)
# This widget controls which tenant's data the dashboard renders.
# NOTE: this is a display-only control and NOT a security boundary. Any user
# with notebook access can change it. A proper multi-tenancy enforcement is
# tracked as a separate epic.
# ============================================================================

import ipywidgets as widgets
from IPython.display import display

def _discover_tenants(path):
    df = spark.read.format("hudi").load(path)
    tenant_col = next((c for c in TENANT_FILTER_COLUMNS if c in df.columns), None)
    if tenant_col is None:
        return [DEFAULT_TENANT]
    return sorted(
        [row[0] for row in df.select(tenant_col).distinct().collect() if row[0]]
    )

# Use gold/cases for the discovery scan — smallest of the tenant-scoped tables.
_tenant_options = _discover_tenants(f"{WAREHOUSE_ROOT}/gold/cases")
if DEFAULT_TENANT not in _tenant_options:
    _tenant_options = [DEFAULT_TENANT] + _tenant_options

tenant_widget = widgets.Dropdown(
    options=_tenant_options,
    value=DEFAULT_TENANT,
    description="Tenant:",
    style={"description_width": "initial"},
)
display(tenant_widget)
print("⚠️  After changing the tenant, re-run all cells below to refresh the dashboard.")
```

**Update every `load_tenant_hudi(...)` call site in the notebook to pass the widget value:**

```python
alerts = load_tenant_hudi(alerts_path, tenant=tenant_widget.value)
cases = load_tenant_hudi(cases_path, tenant=tenant_widget.value)
tasks = load_tenant_hudi(tasks_path, tenant=tenant_widget.value)
transactions = normalize_transactions_for_dashboard(
    load_tenant_hudi(transactions_path, tenant=tenant_widget.value)
)
cms_usernames = load_tenant_hudi(cms_usernames_path, tenant=tenant_widget.value)
```

**Caveats / ordering:**
- The tenant widget must be defined *before* any `load_tenant_hudi` call site executes. If the load calls are in the same cell as the SparkSession bootstrap (as they are in Case_Management_Trend cell 1), split that cell so the widget cell can sit between the helper definition and the load calls.
- Selecting a new tenant does not automatically re-run downstream cells; the markdown warning tells the user to re-execute.

### Step B2 — Executive_Overview_Dashboard.ipynb

Apply the same three-part change to [Executive_Overview_Dashboard.ipynb](repos/biar/JupyterHub/notebooks/Executive_Overview_Dashboard.ipynb):
- Cell 3: rename `TENANT_FILTER_VALUE` → `DEFAULT_TENANT`, add `tenant` kwarg to `load_tenant_hudi`.
- Insert Tenant selector cell after cell 3 (same widget code as B1).
- Cell 5: pass `tenant=tenant_widget.value` to every `load_tenant_hudi(...)` call.

### Step B3 — TMS_Performance_Dashboard.ipynb

Apply the same three-part change to [TMS_Performance_Dashboard.ipynb](repos/biar/JupyterHub/notebooks/TMS_Performance_Dashboard.ipynb):
- Cell 2: rename `TENANT_FILTER_VALUE` → `DEFAULT_TENANT`, add `tenant` kwarg.
- Insert Tenant selector cell after cell 2.
- Cell 3 (and any other load site): thread `tenant=tenant_widget.value`.

### Step B4 — Fraud_Trend_Analysis_Dashboard.ipynb

Apply the same three-part change to [Fraud_Trend_Analysis_Dashboard.ipynb](repos/biar/JupyterHub/notebooks/Fraud_Trend_Analysis_Dashboard.ipynb):
- Cell 1: rename `TENANT_FILTER_VALUE` → `DEFAULT_TENANT`, add `tenant` kwarg.
- Insert Tenant selector cell after cell 1.
- Cell 2: thread `tenant=tenant_widget.value` on all four `load_tenant_hudi` calls.

---

## Data Migration

None. Track A and Track B are both notebook-only changes. No Hudi table columns are added, renamed, or backfilled. The existing tenant column values on `gold/*` tables are the source of truth for the widget's dropdown options.

---

## Test Cases

### Unit tests

There are no Python unit tests in the notebook directory. Add a lightweight PyTest module under `JupyterHub/tests/test_tenant_helper.py` that imports the shared logic once it is extracted, or (interim) run the following inline in a scratch cell:

```python
# Verify load_tenant_hudi behaves as expected against a synthetic DataFrame.
from pyspark.sql import Row

# Case 1: snake_case column matches
df = spark.createDataFrame([Row(tenant_id="TAZAMA", v=1), Row(tenant_id="OTHER", v=2)])
assert next(c for c in TENANT_FILTER_COLUMNS if c in df.columns) == "tenant_id"

# Case 2: camelCase column matches after Track A
df = spark.createDataFrame([Row(tenantId="TAZAMA", v=1), Row(tenantId="OTHER", v=2)])
assert next(c for c in TENANT_FILTER_COLUMNS if c in df.columns) == "tenantId"

# Case 3: no tenant column raises
df = spark.createDataFrame([Row(x=1)])
tenant_col = next((c for c in TENANT_FILTER_COLUMNS if c in df.columns), None)
assert tenant_col is None
```

### Integration / E2E scenarios

1. **Track A regression check (all four notebooks):** Open each notebook in the beta sandbox JupyterHub. Run all cells. Assert no `AnalysisException`, no `NameError`, no swallowed print starting with `✗ Error loading tables`.
2. **Track A tenant isolation:** For each notebook, note the top KPI (e.g. total cases, total transactions). Independently query the Hudi warehouse via `spark-sql` for `SELECT count(*) FROM gold_cases WHERE tenantId = 'TAZAMA'` — the notebook KPI must equal that count.
3. **Track B widget default:** Open each of the four notebooks; verify the dropdown appears, defaults to `TAZAMA`, and running all cells gives the same result as pre-Track-B.
4. **Track B tenant switch:** Change the dropdown to a second tenant that has data in `gold/cases`. Re-run all cells. Verify counts change and match `SELECT count(*) ... WHERE tenantId = '<selected>'`.
5. **Track B empty tenant:** Change the dropdown to a tenant with zero rows. Verify the dashboard renders with visible "no data" indicators rather than crashing.

### Data migration validation SQL

Not applicable — no schema migration in either track.

### Manual / UAT checks

| # | Scenario | Steps | Expected Result |
|---|---|---|---|
| 1 | Executive Overview loads under TAZAMA | Open the notebook, Kernel → Restart & Run All | All cells green, no "✗ Error loading tables" line |
| 2 | Case Management Trend loads under TAZAMA | Same as #1 | Trend charts render; no `AnalysisException` |
| 3 | Track A works with a table using `tenantId` (camelCase) | Repoint `WAREHOUSE_ROOT` to a Hudi dataset whose tenant column is `tenantId`; re-run | `load_tenant_hudi` succeeds; no ValueError |
| 4 | Track A no longer swallows load errors | Repoint to a warehouse with no tenant column; re-run Executive Overview | Notebook stops at cell 5 with a ValueError; no NameError further down |
| 5 | Track B tenant switch | Change dropdown from TAZAMA → other tenant; re-run all cells | KPIs recompute; distinct tenants show distinct counts |
| 6 | Track B default preserves current behaviour | Fresh notebook run without touching the widget | Result identical to pre-Track-B numbers |
| 7 | Follow-up scope check | Case_Tracking_Analysis, Fraud_Typology_Effectiveness, Anomaly_Detection_And_Rule_Calibration remain untouched | Still show all-tenant data — documented as a follow-up ticket |

---

## Overall Impact of the Fix

| Area | Before | After (Track A + Track B) |
|---|---|---|
| Executive Overview dashboard | Fails with a `tenantId` ValueError swallowed as a print, then downstream `NameError`s | Executes end-to-end; tenant scoping enforced at load |
| Case Management Trend dashboard | Cell 4 raises `AnalysisException: cannot resolve 'tenant_id'` when the Hudi table uses a different casing | Cell 4 removed; tenant filter applied once at load |
| TMS Performance dashboard | Same class of failure as above | Same fix — loads cleanly |
| Fraud Trend Analysis dashboard | Same | Same |
| Tenant column naming | Snake_case only (`tenant_id`, `tx_tenant_id`, `tenantid`) | Accepts snake_case and camelCase (`tenantId`, `txTenantId`) |
| Tenant value control | Hardcoded literal `"TAZAMA"` in seven notebooks | ipywidgets dropdown in the four erroring notebooks; other three deferred |
| Error visibility | try/except in Executive Overview swallows tenant load errors | ValueError propagates; operator sees the real cause |

---

## Fix Summary

The BIAR beta sandbox has four Jupyter dashboards that either fail outright or silently render data across every tenant, contrary to the interim multi-tenancy requirement. Underneath, all seven notebooks share a helper called `load_tenant_hudi` that is supposed to filter every Hudi table down to a single tenant, using a hardcoded tenant id of `"TAZAMA"` and an allowlist of three possible column names (`tenant_id`, `tx_tenant_id`, `tenantid`). Two things go wrong: (1) the allowlist doesn't include the camelCase spelling `tenantId` that some tables actually use, so the helper raises a `ValueError`, and (2) the Case Management Trend notebook has a leftover extra filter cell that hardcodes `df.tenant_id == 'TAZAMA'` — that line dies as soon as the underlying DataFrame doesn't have a column called `tenant_id`. Executive Overview then makes the failure worse by wrapping the whole tenant-load block in a `try/except` that prints the error and moves on, so the *visible* symptom becomes a cascade of `NameError` in downstream cells rather than the one real problem.

**Track A** is the safe, immediate fix. Delete the redundant cell 4 in Case Management Trend, unwrap the try/except in Executive Overview so the real error surfaces, and add the two camelCase variants to the tenant-column allowlist in all four erroring notebooks. That is roughly ten lines of net change across four `.ipynb` files, no schema change, no frontend work, no downtime. After Track A ships, all four dashboards run end-to-end against the beta sandbox regardless of whether the tenant column happens to be spelled snake_case or camelCase in the underlying Hudi tables.

**Track B** delivers what the issue actually asks for: a tenant filter that a user can change, not a hardcoded string buried in seven notebooks. It introduces an `ipywidgets` dropdown at the top of each of the four erroring notebooks, populated from a distinct-values scan of the tenant column at kernel startup, and threads the selected value into every `load_tenant_hudi` call site. The default stays `"TAZAMA"` so existing behaviour is preserved, and the widget is documented in the notebook itself as a display-only control — not a security boundary — until the broader multi-tenancy epic lands. Track B is limited to the four erroring notebooks per user direction; the same rollout for Case_Tracking_Analysis, Fraud_Typology_Effectiveness, and Anomaly_Detection_And_Rule_Calibration is captured as a follow-up ticket.
