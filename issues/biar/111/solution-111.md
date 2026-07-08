# Complete Fix — Exact Code Changes

Track A can technically merge as a small PR, but it must be part of the atomic #110+#111 release train — merging path renames before #110 breaks all references. Track B is the full consumer sweep including the short-term view-layer `LEFT JOIN gold/pacs008` for pacs-only fields.

---

### Track A — Path renames (PR 1 of 2, or bundled with #110)

#### File: `automation-orchestrator/Table_ETLs/views_orchestrator.py`

**BEFORE** ([lines 70–116](../../../repos/biar/automation-orchestrator/Table_ETLs/views_orchestrator.py)):

```python
if self._hudi_ready(f"{self.warehouse_root}/bronze/transactions"):
    self._run_view(TransactionDetailViewETL, "vw_transaction_detail")
    self._run_view(TransactionHistoryViewETL, "vw_transaction_history")
...
if (self._hudi_ready(f"{self.warehouse_root}/bronze/transactions")
    and self._hudi_ready(f"{self.warehouse_root}/gold/alerts")
    ...):
    self._run_view(NetworkNavigatorViewETL, "network_navigator")
...
if (self._hudi_ready(f"{self.warehouse_root}/gold/conditions")
    and self._hudi_ready(f"{self.warehouse_root}/gold/transactions")
    and self._hudi_ready(f"{self.warehouse_root}/gold/alerts")):
    self._run_view(ConditionsTimelineViewETL, "conditions_timeline")
```

**AFTER:** Three word swaps — `bronze/transactions` → `bronze/transaction`, `gold/transactions` → `gold/transaction`. `gold/conditions` unchanged.

Small edit, load-bearing consequence: without it, the four transaction-derived view ETLs never trigger after #110.

#### File: `automation-orchestrator/Table_ETLs/ConditionsTimelineView.py`

**BEFORE** ([line 98](../../../repos/biar/automation-orchestrator/Table_ETLs/ConditionsTimelineView.py)):

```python
t0 = self.spark.read.format("hudi").load(f"{self.warehouse_root}/gold/transactions").alias("t")
```

**AFTER:**

```python
t0 = self.spark.read.format("hudi").load(f"{self.warehouse_root}/gold/transaction").alias("t")
```

Consumer uses only structural fields (`transaction_id`, `end_to_end_id`, `tenant_id`, `event_ts`, `tx_type`, `tx_amount`, `tx_ccy`); no pacs join needed.

#### File: `automation-orchestrator/Table_ETLs/alert_navigator.py`

**BEFORE** ([line 396](../../../repos/biar/automation-orchestrator/Table_ETLs/alert_navigator.py)):

```python
transactions_gold_path = f"{self.warehouse_root}/gold/transactions"
```

**AFTER:**

```python
transactions_gold_path = f"{self.warehouse_root}/gold/transaction"
```

Reads only TMS-core columns — no pacs join needed.

#### File: `datalakehouse-api/lakehouse_query_api.py`

**BEFORE** ([lines 139, 166](../../../repos/biar/datalakehouse-api/lakehouse_query_api.py)):

```python
transactions_gold_path  = f"{WAREHOUSE_ROOT}/gold/transactions"
...
GOLD_PATHS = {
    ...
    "transactions":                    transactions_gold_path,
    ...
}
```

**AFTER:**

```python
transaction_gold_path  = f"{WAREHOUSE_ROOT}/gold/transaction"
...
GOLD_PATHS = {
    ...
    "transaction":                     transaction_gold_path,
    "transactions":                    transaction_gold_path,  # temporary alias — remove after one release
    ...
}
```

Alias retained for one release to give API consumers time to migrate the URL query param.

#### File: `JupyterHub/notebooks/Case_Management_Trend_Dashboard.ipynb`

**BEFORE** (cell at line 153 in the source): `transactions_path = f"{WAREHOUSE_ROOT}/gold/transactions"`

**AFTER:** `transactions_path = f"{WAREHOUSE_ROOT}/gold/transaction"`

Path rename only.

#### File: `JupyterHub/notebooks/Anomaly_Detection_And_Rule_Calibration.ipynb` (path rename only — see Track B for schema decision)

**BEFORE** (cells 7 and 14): `transactions_path = f"{WAREHOUSE_ROOT}/gold/transactions"`

**AFTER:** `transactions_path = f"{WAREHOUSE_ROOT}/gold/transaction"`

Path rename alone is **not sufficient** — see Track B for the `instg_mmb_id`/`instd_mmb_id`/`charge_count` schema gap.

**Impact of Track A alone:** the four view ETLs correctly build (via `views_orchestrator`); `ConditionsTimelineView`, `alert_navigator`, and the two notebooks locate their upstream. Anomaly Detection still fails on missing columns — Track B addresses.

---

### Track B — View rewrites + pacs joins + notebook decision

#### Step B1 — `transaction_detail_view.py` — switch to `gold/transaction` + pacs join

**File:** `automation-orchestrator/Table_ETLs/transaction_detail_view.py`

**BEFORE** (schematic — see [lines 25, 53–125](../../../repos/biar/automation-orchestrator/Table_ETLs/transaction_detail_view.py)):

```python
self.transactions_bronze_path = f"{self.warehouse_root}/bronze/transactions"
...
def _resolve_json_column(self, df):
    candidates = ["transactionData", "transaction_data", "transaction", "payload", …]
    …
    return df.withColumn("transaction_data", F.col(json_col).cast("string"))

def _extract_pacs_fields(self, df):
    dbtr_name = F.get_json_object("transaction_data", "$.FIToFICstmrCdtTrf.CdtTrfTxInf.Dbtr.Nm")
    cdtr_name = F.get_json_object("transaction_data", "$.FIToFICstmrCdtTrf.CdtTrfTxInf.Cdtr.Nm")
    intrbk_amt = F.get_json_object("transaction_data", "$.FIToFICstmrCdtTrf.CdtTrfTxInf.IntrBkSttlmAmt.Amt.Amt").cast("double")
    xchg_rate = F.get_json_object("transaction_data", "$.FIToFICstmrCdtTrf.CdtTrfTxInf.XchgRate").cast("double")
    …
```

**AFTER:**

```python
self.transactions_gold_path = f"{self.warehouse_root}/gold/transaction"
self.pacs008_gold_path      = f"{self.warehouse_root}/gold/pacs008"
...
# _resolve_json_column removed (candidate list hardening — see #108)

def run(self, source_path=None):
    tx  = self.spark.read.format("hudi").load(self.transactions_gold_path).alias("tx")
    p8  = self.spark.read.format("hudi").load(self.pacs008_gold_path).alias("p8")

    joined = (tx.join(p8,
                      (F.col("tx.end_to_end_id") == F.col("p8.end_to_end_id")) &
                      (F.col("tx.tenant_id")    == F.col("p8.tenant_id")),
                      how="left")
              .select(
                  F.col("tx.transaction_id"),
                  F.col("tx.end_to_end_id"),
                  F.col("tx.tenant_id"),
                  F.col("tx.tx_type"),
                  F.col("tx.tx_status"),
                  F.col("tx.tx_amount"),
                  F.col("tx.tx_ccy"),
                  F.col("tx.tx_msg_id"),
                  F.col("tx.source_account_id").alias("debtor_account_id"),
                  F.col("tx.destination_account_id").alias("creditor_account_id"),
                  F.col("tx.debtor_entity_id").alias("debtor_id"),
                  F.col("tx.creditor_entity_id").alias("creditor_id"),
                  F.col("tx.event_ts").alias("tx_event_ts"),
                  F.col("tx.event_date"),
                  # pacs-only fields (short-term view-layer join)
                  F.col("p8.dbtr_name").alias("debtor_name"),
                  F.col("p8.cdtr_name").alias("creditor_name"),
                  F.col("p8.intrbk_sttlm_amt").alias("intrbk_sttlm_amt"),
                  F.col("p8.intrbk_sttlm_ccy").alias("intrbk_sttlm_ccy"),
                  F.col("p8.xchg_rate").alias("exchange_rate"),
                  F.col("p8.instg_mmb_id"),
                  F.col("p8.instd_mmb_id"),
                  F.col("p8.charge_count"),
              ))

    self.write_hudi(joined, self.view_path, self.hudi_opts("vw_transaction_detail", "transaction_id", "tx_event_ts"))
    return self.view_path
```

Assumes `gold/pacs008` exposes `dbtr_name`, `cdtr_name`, `intrbk_sttlm_amt`, `intrbk_sttlm_ccy`, `xchg_rate`, `instg_mmb_id`, `instd_mmb_id`, `charge_count` at Gold. Verify against `Pacs008ETL.gold` output; extend if any of these fields are not currently projected.

#### Step B2 — `transaction_history_view.py` — same treatment

**File:** `automation-orchestrator/Table_ETLs/transaction_history_view.py`

Same pattern: switch to `gold/transaction`; add composite-key `LEFT JOIN gold/pacs008` for the `Dbtr.Nm`/`Cdtr.Nm` and `IntrBkSttlm` fields the current view exposes.

#### Step B3 — `network_navigator_view.py` — switch without pacs join

**File:** `automation-orchestrator/Table_ETLs/network_navigator_view.py`

**BEFORE** ([lines 25, 55–102](../../../repos/biar/automation-orchestrator/Table_ETLs/network_navigator_view.py)):

```python
self.transactions_bronze_path = f"{self.warehouse_root}/bronze/transactions"
...
tx = self.spark.read.format("hudi").load(self.transactions_bronze_path)
...
candidates = ["transactionData", "transaction_data", "transaction", "payload", "raw_payload"]
json_col = next((c for c in candidates if c in tx.columns), None)
tx = tx.withColumn("transaction_data", F.col(json_col).cast("string"))
dbtr_id = F.get_json_object("transaction_data", "$.FIToFICstmrCdtTrf.CdtTrfTxInf.Dbtr.Id.PrvtId.Othr[0].Id")
cdtr_id = F.get_json_object("transaction_data", "$.FIToFICstmrCdtTrf.CdtTrfTxInf.Cdtr.Id.PrvtId.Othr[0].Id")
dbtr_acct = F.get_json_object("transaction_data", "$.FIToFICstmrCdtTrf.CdtTrfTxInf.DbtrAcct.Id.Othr[0].Id")
cdtr_acct = F.get_json_object("transaction_data", "$.FIToFICstmrCdtTrf.CdtTrfTxInf.CdtrAcct.Id.Othr[0].Id")
```

**AFTER:**

```python
self.transactions_gold_path = f"{self.warehouse_root}/gold/transaction"
...
tx = self.spark.read.format("hudi").load(self.transactions_gold_path)
# Structural fields available directly:
#   source_account_id, destination_account_id  → debtor/creditor account IDs
#   debtor_entity_id, creditor_entity_id       → debtor/creditor entity IDs
# No ISO path extractions; no pacs join.
```

Remove the candidate-list block entirely.

#### Step B4 — CMS backend — verify column contract

**Files:** `case-management-system/backend/src/modules/gold-lakehouse/transaction-lakehouse.service.ts`, `case-management-system/backend/src/modules/alert/alert.service.ts`

No code changes required — the SQL joins against `transaction_detail` are unchanged. Add integration tests to confirm `debtor_name`/`creditor_name` populate for pacs.008-based rows after the view rewrite.

#### Step B5 — Anomaly Detection notebook — decision

**File:** `biar/JupyterHub/notebooks/Anomaly_Detection_And_Rule_Calibration.ipynb`

Choose one:

**Option A — Add pacs join at the notebook layer.** After `transactions = spark.read.format("hudi").load(transactions_path)`, add a Spark join to `gold/pacs008` on `(end_to_end_id, tenant_id)` producing `instg_mmb_id`, `instd_mmb_id`, `charge_count`. Preserves the notebook's current feature-engineering cells verbatim.

**Option B — Read from a view builder that already joins.** Introduce a new view builder `anomaly_features_view.py` that provides the joined output; notebook reads from that view.

**Option C — Block the notebook on post-MVP.** Mark the notebook as blocked pending #110 Design Requirement 6; do not run it against the new Gold table until then.

Recommendation: **Option A** in the interim. It is fastest and keeps the notebook logic in one place; the join is retired when post-MVP Gold enrichment lands.

#### Step B6 — DLH Query API URL contract

Coordinate with API consumers. Track A retains `"transactions"` as an alias key; announce a deprecation. Add a `Deprecation:` HTTP response header or a warning in the API's OpenAPI spec. Remove the alias after one release.

---

### Data migration section

None. All changes are query-layer / view-layer / notebook.

---

### Test cases

#### Unit tests to add

- `test_transaction_detail_view_joins_pacs008_on_composite_key`: mock `gold/transaction` and `gold/pacs008`; assert the view produces expected `debtor_name` / `creditor_name` / `xchg_rate` for matching rows and nulls for non-matching.
- `test_network_navigator_view_no_pacs_join`: assert the view does not read `gold/pacs008`.
- `test_dlh_query_api_transaction_alias`: `?table=transactions` and `?table=transaction` both return data.

#### Integration / E2E scenarios

1. Send synthetic pacs.008 to TMS → `gold/transaction` gets a row; `gold/pacs008` gets a row. Wait for `views_orchestrator` cycle. Verify `vw_transaction_detail` contains `debtor_name` and `creditor_name`.
2. Send matching pacs.002 → `gold/transaction`'s row's `tx_status` updates; `vw_transaction_detail` still has `debtor_name`/`creditor_name` (from pacs.008 join).
3. Run Anomaly Detection notebook cell 10 → `instg_mmb_id`/`instd_mmb_id`/`charge_count` present (Option A) or notebook errors cleanly (Option C).
4. CMS: open a case with a pacs.008-based transaction → Transaction Detail panel shows `debtor_name`/`creditor_name`.
5. DLH Query API: `GET /query?table=transactions` returns data (via alias); `GET /query?table=transaction` returns the same data.

#### Data migration validation SQL

Not applicable.

#### Manual / UAT checks

| # | Scenario | Steps | Expected Result |
|---|---|---|---|
| 1 | View builder cycle | Deploy #110+#111; wait one poll interval | All 4 transaction-derived views build without error |
| 2 | CMS Transaction Detail | Open a case linked to a pacs.008 message | `debtor_name` and `creditor_name` populated |
| 3 | CMS Alert transactional data | Query `getAlertTransactionalData(endToEndId)` | Returns `transaction_detail` row with names |
| 4 | Anomaly notebook | Run notebook cells 1–20 | Completes without missing-column error |
| 5 | Case Management Trend Dashboard | Load notebook | Loads `gold/transaction` correctly; visualisations render |
| 6 | DLH Query API deprecation | `GET /query?table=transactions` | Returns data + Deprecation header |
| 7 | Live NiFi | Verify Transaction processor still RUNNING and Ozone accumulates JSON | Unchanged from pre-deploy |

---

### Overall Impact of the Fix

| Area | Before | After |
|---|---|---|
| View builder inputs | `bronze/transactions` (raw pacs JSON) | `gold/transaction` (TMS-normalised) |
| Party names on pacs.002 rows | Null (pacs.002 doesn't carry `Dbtr.Nm`) | Non-null when a matching pacs.008 exists (via view join) |
| Anomaly Detection notebook | Uses pacs-only fields from `gold/transactions` | Uses same fields joined from `gold/pacs008` at notebook layer (Option A) |
| CMS UI transaction detail | Names + settlement amounts sourced from raw pacs JSON via view builder | Same data, cleaner path: TMS fields from Gold, pacs fields via composite join |
| DLH Query API | Key `"transactions"` | Key `"transaction"` (alias `"transactions"` for one release) |
| Path-rename consumers | Broken after #110 rename | Corrected |

---

### Fix Summary

Fourteen consumers reference `gold/transactions` (or bronze/silver equivalents) or read fields that #110 removes from the MVP Gold schema. Path renames alone fix seven of them cleanly (Track A). The remaining seven — the three transaction-derived view builders, the two CMS backend services depending on those views, the Anomaly Detection notebook, and (bookkeeping) the DLH Query API — need a short-term `LEFT JOIN gold/pacs008 ON (end_to_end_id, tenant_id)` at the view or notebook layer to surface the pacs-only fields (party names, settlement amounts, exchange rate, member IDs, charge count) that the MVP `gold/transaction` no longer carries directly.

The join is composite-key by design: `end_to_end_id` alone is not unique across tenants and would allow cross-tenant collisions. This short-term arrangement lasts until #110 Design Requirement 6 (post-MVP Gold enrichment) implements the join at the Gold layer, at which point the per-consumer joins are removed and this issue's residual work concludes.

Two verification findings shaped the delivery:
- `ConditionsTimelineView.py` was flagged missing from dev at initial accuracy review; it has since been merged and is on dev with a `gold/transactions` read at line 98 (path rename only).
- `views_orchestrator.py` was not in the original consumer list; it is a small path-rename change but critical (readiness gates control whether the view builders run).

Live JupyterHub inspection also revealed that the Anomaly Detection notebook uses `instg_mmb_id`/`instd_mmb_id` as **join keys**, not just as features, so path rename alone is not sufficient. Option A above (pacs join at notebook layer) is the recommended interim; Option C (block on post-MVP) is acceptable if the notebook is not on the MVP critical path.
