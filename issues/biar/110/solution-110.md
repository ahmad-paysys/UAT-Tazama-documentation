# Complete Fix — Exact Code Changes

Track B must land as a single coordinated release. Track A is not applicable (no meaningful minimal fix exists). Consumer sweep is in [solution-111.md](../111/solution-111.md); view-builder candidate list hardening is in [solution-108.md](../108/solution-108.md) and folded into the same PR.

---

### Track A — Not applicable

There is no partial change that provides value without breaking something else. If the team wants a documentation-only change to record intent before the rewrite lands, add a `# TODO(#110): wrong source — see issues/biar/110` comment above `TransactionsETL.bronze()` at [line 108](../../../repos/biar/automation-orchestrator/Table_ETLs/TransactionsETL.py). No other files touched.

---

### Track B — The rewrite (PR 1 of 1)

#### Step B1 — `TransactionsETL.py` — rewrite `bronze`

**File:** `automation-orchestrator/Table_ETLs/TransactionsETL.py`

**BEFORE** (lines 36–46, 108–166):

```python
@property
def bronze_path(self) -> str:  return f"{self.warehouse_root}/bronze/transactions"
@property
def silver_path(self) -> str:  return f"{self.warehouse_root}/silver/transactions"
@property
def gold_path(self) -> str:    return f"{self.warehouse_root}/gold/transactions"

def bronze(self, source_path: str, mode: str = "from_pacs") -> str:
    pacs008_bronze_path = f"{self.warehouse_root}/bronze/pacs008"
    pacs002_bronze_path = f"{self.warehouse_root}/bronze/pacs002"
    df_pacs008 = self.spark.read.format("hudi").load(pacs008_bronze_path)
    df_pacs002 = self.spark.read.format("hudi").load(pacs002_bronze_path)
    if mode == "from_pacs":
        raw = (self._transactions_from_pacs(df_pacs008, "pacs008")
               .unionByName(self._transactions_from_pacs(df_pacs002, "pacs002")))
    else:
        raw = self._join_mode(source_path, df_pacs008, df_pacs002)
    bronze = (raw
        .withColumn("createdAt",       F.col("createdAt").cast("long"))
        .withColumn("endToEndId",      F.col("endToEndId").cast("string"))
        .withColumn("tenantId",        F.col("tenantId").cast("string"))
        .withColumn("transactionData", F.col("transactionData").cast("string"))
        ...)
```

**AFTER:**

```python
@property
def bronze_path(self) -> str:  return f"{self.warehouse_root}/bronze/transaction"   # singular
@property
def silver_path(self) -> str:  return f"{self.warehouse_root}/silver/transaction"
@property
def gold_path(self) -> str:    return f"{self.warehouse_root}/gold/transaction"

def bronze(self, source_path: str) -> str:
    """
    Read a batch of event_history.transaction rows landed at source_path
    (Ozone JSON dropped by the NiFi Transaction processor).

    Schema of each row (per Full-Stack-Docker-Tazama 00-CREATE.sql):
      source, destination, transaction (jsonb), endToEndId, amt, ccy, msgId,
      creDtTm, txTp, txSts, tenantId
    """
    df = self.spark.read.json(source_path)

    bronze = (df
        .withColumn("source",       F.col("source").cast("string"))
        .withColumn("destination",  F.col("destination").cast("string"))
        .withColumn("transaction",  F.to_json(F.col("transaction")))   # keep as JSON string for downstream
        .withColumn("end_to_end_id", F.col("endToEndId").cast("string"))
        .withColumn("tx_type",      F.col("txTp").cast("string"))
        .withColumn("tx_status",    F.col("txSts").cast("string"))
        .withColumn("tx_amount",    F.col("amt").cast("double"))
        .withColumn("tx_ccy",       F.col("ccy").cast("string"))
        .withColumn("tx_msg_id",    F.col("msgId").cast("string"))
        .withColumn("cre_dt_tm",    F.col("creDtTm").cast("string"))
        .withColumn("tenant_id",    F.col("tenantId").cast("string"))
        .withColumn("created_at_ts", F.current_timestamp())
        .withColumn("source_file_path", F.lit(source_path))
        # Composite PK carried through Bronze → Silver → Gold
        .withColumn("transaction_id",
                    F.concat_ws("||",
                                F.coalesce(F.col("tx_type"), F.lit("")),
                                F.coalesce(F.col("end_to_end_id"), F.lit(""))))
        .withColumn("record_hash",
                    F.sha2(F.concat_ws("||",
                                       F.col("end_to_end_id"),
                                       F.col("tenant_id"),
                                       F.col("cre_dt_tm"),
                                       F.col("transaction")), 256))
    )

    self.write_hudi(bronze, self.bronze_path,
                    self.hudi_opts("transaction", "transaction_id", "created_at_ts"))
    print(f"[TransactionsETL] Bronze written → {self.bronze_path}")
    return self.bronze_path
```

Rewrite reads from Ozone JSON files landed by the live NiFi `Transaction` processor. All pacs-specific bronze reads removed. Column names normalised to snake_case, `transaction` column preserved as a JSON string for downstream inspection.

#### Step B2 — `TransactionsETL.py` — rewrite `silver` with entity-join enrichment

**BEFORE** (lines 222–292): silver reads bronze, extracts ISO 20022 paths.

**AFTER:**

```python
def silver(self) -> str:
    b = self.spark.read.format("hudi").load(self.bronze_path)

    # Join account_holder to derive entity ids for source (debtor) and destination (creditor).
    account_holder_path = f"{self.warehouse_root}/bronze/account_holder"
    ah = self.spark.read.format("hudi").load(account_holder_path).select(
        F.col("source").alias("ah_source"),
        F.col("destination").alias("ah_destination"),
        F.col("tenantId").alias("ah_tenant_id"),
    )

    # transaction.source = account.id → account_holder.destination → account_holder.source = entity.id
    s = (b
         .join(ah,
               (b["source"] == ah["ah_destination"]) & (b["tenant_id"] == ah["ah_tenant_id"]),
               how="left")
         .withColumnRenamed("ah_source", "debtor_entity_id")
         .drop("ah_destination", "ah_tenant_id")
    )
    ah2 = ah.selectExpr(
        "ah_source as ah_source2",
        "ah_destination as ah_destination2",
        "ah_tenant_id as ah_tenant_id2",
    )
    s = (s
         .join(ah2,
               (s["destination"] == ah2["ah_destination2"]) & (s["tenant_id"] == ah2["ah_tenant_id2"]),
               how="left")
         .withColumnRenamed("ah_source2", "creditor_entity_id")
         .drop("ah_destination2", "ah_tenant_id2")
    )

    s = (s
         .withColumn("event_ts",   F.to_timestamp(F.col("cre_dt_tm")))
         .withColumn("event_date", F.to_date("event_ts"))
    )

    # Dedup — keep latest per composite PK
    w = Window.partitionBy("transaction_id").orderBy(F.col("created_at_ts").desc())
    silver = s.withColumn("_rn", F.row_number().over(w)).filter("_rn = 1").drop("_rn")

    self.write_hudi(silver, self.silver_path,
                    self.hudi_opts("silver_transaction", "transaction_id", "created_at_ts"))
    print(f"[TransactionsETL] Silver written → {self.silver_path}")
    return self.silver_path
```

Entity-join chain follows Design Requirement 7. `debtor_entity_id` / `creditor_entity_id` present for **all rows** because the ODS enforces FKs on `source`/`destination` on the transaction table and both entities exist by the time a transaction lands. No pacs joins yet — those are deferred.

#### Step B3 — `TransactionsETL.py` — rewrite `gold`

**BEFORE** (lines 298–342).

**AFTER:**

```python
def gold(self) -> str:
    s = self.spark.read.format("hudi").load(self.silver_path)

    gold = (s.select(
        F.col("transaction_id"),
        F.col("end_to_end_id"),
        F.col("tenant_id"),
        F.col("tx_type"),
        F.col("tx_status"),
        F.col("tx_amount"),
        F.col("tx_ccy"),
        F.col("tx_msg_id"),
        F.col("source").alias("source_account_id"),
        F.col("destination").alias("destination_account_id"),
        F.col("debtor_entity_id"),
        F.col("creditor_entity_id"),
        F.col("event_ts"),
        F.col("event_date"),
        F.col("created_at_ts").alias("ingested_at_ts"),
        F.col("source_file_path"),
        F.col("record_hash"),
    ))

    bad = [c for c, t in gold.dtypes if t.startswith(("array", "struct"))]
    if bad:
        raise RuntimeError(f"[TransactionsETL] Gold contains non-scalar cols: {bad}")

    gold_opts = {
        **self.hudi_opts("transaction", "transaction_id", "ingested_at_ts"),
        "hoodie.datasource.write.payload.class":
            "org.apache.hudi.common.model.OverwriteWithLatestAvroPayload",
    }
    self.write_hudi(gold, self.gold_path, gold_opts)
    print(f"[TransactionsETL] Gold written → {self.gold_path}")
    return self.gold_path
```

MVP Gold. No pacs enrichment. Consumers requiring `intr_bk_sttlm_amt` / `xchg_rate` / `instg_mmb_id` / `instd_mmb_id` / `charge_count` add a short-term view-layer `LEFT JOIN gold/pacs008` (see #111).

#### Step B4 — `TransactionsETL.py` — remove `_join_mode` and `run` signature

Delete `_transactions_from_pacs`, `_join_mode`, and simplify `run`:

```python
def run(self, source_path: str) -> str:
    print(f"[TransactionsETL] Starting Bronze → Silver → Gold from {source_path}")
    self.bronze(source_path)
    self.silver()
    self.gold()
    print("[TransactionsETL] ETL complete.")
    return self.gold_path
```

#### Step B5 — `lakehouse_automation_pipeline.py` — registry and skip-guard

**File:** `automation-orchestrator/lakehouse_automation_pipeline.py`

**BEFORE** (lines 12, 51–68, 150–176):

```python
from Table_ETLs.CombinedPacs import CombinedPacsETL
...
_ETL_REGISTRY: dict[str, type] = {
    "account": AccountETL,
    ...
    "evaluation": EvaluationETL,
}
...
def _route_etl(self, table, source_path, db_name=None):
    if table in ("pacs008", "pacs002"):
        CombinedPacsETL(self.spark, self.warehouse_root, table=table).run(source_path)
        return "All Done"
    if table in ("transaction", "transactions"):
        print("Skipping standalone Transactions ETL: ...")
        return "Skipped: transactions are derived from PACS gold"
    if table in self._ETL_REGISTRY:
        self._ETL_REGISTRY[table](self.spark, self.warehouse_root).run(source_path)
        return "All Done"
    ...
```

**AFTER:**

```python
from Table_ETLs.Pacs008ETL import Pacs008ETL
from Table_ETLs.Pacs002ETL import Pacs002ETL
from Table_ETLs.TransactionsETL import TransactionsETL
# (import of CombinedPacs removed)

_ETL_REGISTRY: dict[str, type] = {
    "account": AccountETL,
    "account_holder": AccountHolderETL,
    "alerts": AlertsETL,
    "cases": CasesETL,
    "conditions": ConditionsETL,
    "network_map": NetworkMapETL,
    "rule": RulesETL,
    "rules": RulesETL,
    "tasks": TasksETL,
    "typology": TypologiesETL,
    "typologies": TypologiesETL,
    "comment": CommentsETL,
    "comments": CommentsETL,
    "entity": EntityETL,
    "entities": EntityETL,
    "evaluation": EvaluationETL,
    "pacs008": Pacs008ETL,
    "pacs002": Pacs002ETL,
    "transaction":  TransactionsETL,
    "transactions": TransactionsETL,
}

def _route_etl(self, table, source_path, db_name=None):
    if table in self._ETL_REGISTRY:
        self._ETL_REGISTRY[table](self.spark, self.warehouse_root).run(source_path)
        return "All Done"
    # FALLBACK — unknown table → DynamicETL with db_name
    print(f"[FullETLOrchestrator] '{table}' not in ETL registry. Falling back to DynamicETL.")
    DynamicETL(self.spark, self.warehouse_root, db_name=db_name, table=table).run(source_path)
    return "All Done (DynamicETL fallback)"
```

#### Step B6 — Delete `automation-orchestrator/Table_ETLs/CombinedPacs.py`

`git rm automation-orchestrator/Table_ETLs/CombinedPacs.py`.

#### Step B7 — `views_orchestrator.py` — singular readiness paths

**File:** `automation-orchestrator/Table_ETLs/views_orchestrator.py`, [lines 70–116](../../../repos/biar/automation-orchestrator/Table_ETLs/views_orchestrator.py):

**BEFORE:**

```python
if self._hudi_ready(f"{self.warehouse_root}/bronze/transactions"):
    self._run_view(TransactionDetailViewETL, "vw_transaction_detail")
    self._run_view(TransactionHistoryViewETL, "vw_transaction_history")
else:
    print("[ViewsOrchestrator] Skipping transaction views (missing bronze/transactions)")

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

**AFTER:** `bronze/transactions` → `bronze/transaction`; `gold/transactions` → `gold/transaction`; `gold/conditions` unchanged (that path is not part of #110's rename). Textually one word ("transactions" → "transaction") on the three affected lines (71, 79, 108 in the current dev tree).

#### Step B8 — Migration commands (drop old + delete marker files)

Runtime, executed once on the orchestrator container or via a one-shot pod:

```bash
# Drop plural Hudi tables (Spark or hudi-cli):
spark-submit --conf … --class Cleaner ~/scripts/drop_hudi.py \
    /opt/biar/test_warehouse/bronze/transactions \
    /opt/biar/test_warehouse/silver/transactions \
    /opt/biar/test_warehouse/gold/transactions

# Remove marker files
rm -f /opt/biar/test_warehouse/.pipeline_state/pacs008_gold_done
rm -f /opt/biar/test_warehouse/.pipeline_state/pacs002_gold_done
rm -f /opt/biar/test_warehouse/.pipeline_state/transactions_trigger.lock
```

Ordering caveat: drop the plural Hudi tables **after** the new code is deployed and the singular tables have been created by at least one poll cycle; otherwise views_orchestrator may skip during the deploy window.

---

### Data migration section

No PostgreSQL schema changes. Hudi tables are rebuilt from the NiFi `Transaction` processor's watermark. NiFi has been polling `event_history.transaction` since deployment; on first run after the rewrite, `TransactionsETL` reads the accumulated Ozone JSON at `${bucket}/transaction/` and builds fresh Bronze/Silver/Gold Hudi tables.

The NiFi watermark on the `Transaction` processor is `credttm` (per live inspection). No state reset is required; NiFi's own state tracks the last-seen `credttm`. If a full backfill is desired, reset the NiFi processor's state (via NiFi UI → right-click → View State → Clear State) before enabling the new orchestrator behaviour.

---

### Test cases

#### Unit tests to add

- `test_bronze_reads_event_history_transaction_json`: assert bronze reads from `source_path` and produces expected column names (`source`, `destination`, `transaction`, `end_to_end_id`, `tx_type`, `tx_status`, etc.).
- `test_silver_joins_account_holder_for_debtor_and_creditor_entity`: given a mock `account_holder` with `source=E1, destination=A1` and `source=E2, destination=A2`, and a transaction with `source=A1, destination=A2`, silver produces `debtor_entity_id=E1, creditor_entity_id=E2`.
- `test_gold_excludes_pacs_specific_columns`: assert `gold/transaction` schema has no `intr_bk_sttlm_amt`, `xchg_rate`, `instg_mmb_id`, `instd_mmb_id`, `charge_count`, `debtor_name`, `creditor_name`.
- `test_route_etl_no_skip_guard_transaction`: given `table="transaction"`, `_route_etl` dispatches to `TransactionsETL`, not the skip guard.
- `test_route_etl_pacs008_direct`: given `table="pacs008"`, dispatches to `Pacs008ETL` directly, not `CombinedPacsETL`.

#### Integration / E2E scenarios

1. Send a synthetic pacs.008 message through the TMS pipeline. Verify `event_history.transaction` gets a row. Verify NiFi polls it and Ozone gets a JSON file at `${bucket}/transaction/`. Verify the orchestrator receives the payload with `table=transaction` and runs `TransactionsETL`. Verify `gold/transaction` gets a row with the expected `source`, `destination`, `debtor_entity_id`, `creditor_entity_id`, and TMS-flat fields.
2. Send a matching pacs.002 with `TxSts=ACCC`. Verify the `gold/transaction` row for that `endToEndId` reflects the updated `tx_status`. (Behavior depends on TMS `txSts`-update semantics — see risk table in impact-110.md.)
3. Send a pacs.008-only path — verify NO `bronze/transactions` (plural) rows appear, confirming `CombinedPacsETL` and skip-guard removal took effect.
4. Verify `views_orchestrator` runs `TransactionDetailViewETL` etc. once `bronze/transaction` is populated; verify view builders bind to the explicit `transaction` column.
5. Verify Anomaly Detection notebook (once #111 updates it) runs against `gold/transaction` and either the view-layer pacs join or a schema flag surfaces missing `instg_mmb_id`/`instd_mmb_id`.

#### Data migration validation SQL

None (no PG migration). Post-deploy sanity queries on Hudi via `spark.sql`:

```sql
-- schema sanity
SELECT * FROM gold_transaction LIMIT 5;

-- populated entity ids
SELECT COUNT(*) AS total,
       COUNT(debtor_entity_id) AS with_debtor,
       COUNT(creditor_entity_id) AS with_creditor
FROM gold_transaction;
-- expected: total == with_debtor == with_creditor (all rows have both, per ODS FK)

-- pacs.002 rows have tx_status populated
SELECT COUNT(*) FROM gold_transaction
WHERE tx_type LIKE 'pacs.002.%' AND tx_status IS NOT NULL;
```

#### Manual / UAT checks

| # | Scenario | Steps | Expected Result |
|---|---|---|---|
| 1 | End-to-end pacs.008 | Send synthetic pacs.008 to TMS; wait for NiFi poll cycle | New row in `gold/transaction` within one poll interval; `source`/`destination` populated |
| 2 | pacs.002 update | Send matching pacs.002; wait for NiFi poll cycle | `gold/transaction` row for same `endToEndId` reflects new `tx_status` |
| 3 | Skip guard removed | Log-tail orchestrator during a `table=transaction` poll | No "Skipped: transactions are derived from PACS gold" log; instead "TransactionsETL Bronze written…" |
| 4 | Anomaly Detection notebook | Run notebook cell 10 | Either succeeds (view-layer pacs join available) or fails cleanly on missing column (documented gap; not silent) |
| 5 | CMS Transaction Detail | Open a case; view transaction detail panel | `debtor_name` / `creditor_name` populated for pacs.008-based transactions (via view-layer pacs join); tolerated as null for pacs.002-only rows |

---

### Overall Impact of the Fix

| Area | Before | After |
|---|---|---|
| `gold/transactions` source | Raw pacs.008 + pacs.002 messages | `event_history.transaction` ODS |
| Row structure | One pacs message per row; Amt/Ccy null on pacs.002; TxSts null on pacs.008 | TMS-normalised flat record; Amt/Ccy/TxSts present per business logic |
| `source`/`destination` account IDs | Absent | Present for all rows (FK-validated by ODS) |
| Party identifiers | ISO paths on pacs.008 only | `debtor_entity_id`/`creditor_entity_id` via entity join for all rows |
| Pacs-specific fields at Gold | Present as columns | Absent at MVP; joinable from `gold/pacs008` via view layer; post-MVP: enriched at Gold |
| Naming collision (#108) | 5 vectors | 1 vector (CMS Prisma model — unambiguously CMS-scoped) |
| Skip guard | Discards every ODS batch | Removed |
| `CombinedPacsETL` | Coupled pacs008/pacs002 pipelines | Deleted |
| Table names | `*/transactions` (plural) | `*/transaction` (singular, matching ODS name and NiFi routing attribute) |

---

### Fix Summary

Today, `gold/transactions` is fed by raw pacs.008 and pacs.002 messages via `TransactionsETL(mode="from_pacs")`, orchestrated by `CombinedPacsETL` on marker files. The correct upstream — the TMS-normalised `event_history.transaction` table — is being polled by a live NiFi processor, landed into Ozone, and then explicitly discarded by a skip guard in the automation orchestrator's `_route_etl`. This is a wrong-source decision, and every downstream artefact (naming collision, missing FK columns, null-on-half-the-rows fields, chain of consumer joins to `gold/pacs002`) is a symptom.

Track B rewrites `TransactionsETL` to read from the NiFi-landed Ozone JSON at `${bucket}/transaction/`. It produces singular-named Hudi tables (`bronze/transaction`, `silver/transaction`, `gold/transaction`) with TMS-flat scalar fields (`end_to_end_id`, `tx_type`, `tx_status`, `tx_amount`, `tx_ccy`, `tx_msg_id`) plus the two structural columns that only exist in the ODS layer: `source` and `destination` FKs to the `account` table. Party identifiers are derived by joining `account_holder` and `entity`; the join produces `debtor_entity_id` / `creditor_entity_id` for every row, pacs.008 and pacs.002 alike. `CombinedPacsETL` is deleted; the skip guard is removed; `_ETL_REGISTRY` gains direct entries for `pacs008`, `pacs002`, `transaction`, `transactions`.

Track B has no independent Track A. The change is atomic by nature — you cannot remove the skip guard without also rewriting the ETL, and you cannot delete `CombinedPacsETL` without also giving `Pacs008ETL` / `Pacs002ETL` their own dispatch. The view-builder hardening from [#108](../108/) folds into the same PR. The 14 consumer updates in [#111](../111/) land immediately after. Pacs-specific enrichment at the Gold layer (`intr_bk_sttlm_amt`, `xchg_rate`, `instg_mmb_id`, `instd_mmb_id`, `charge_count`) is deferred to a post-MVP follow-up; MVP consumers that need those fields add a short-term view-layer `LEFT JOIN gold/pacs008 ON (end_to_end_id, tenant_id)`.

One question that must be resolved with the TMS team **before** the ETL rewrite lands: whether `event_history.transaction.txSts` is updated in-place after pacs.002 receipt (upsert key `(endToEndId, tenantId)`) or a new row is inserted with `txTp='pacs.002.001.12'` (upsert key `(endToEndId, txTp, tenantId)`). The Hudi dedupe key selection depends on this.
