# Issue #110 — DLH BUG: event_history.transaction data is not the source for gold/transactions in DLH

**Repository:** tazama-lf/biar
**Issue:** [DLH BUG: event_history.transaction data is not the source for gold/transactions in DLH](https://github.com/tazama-lf/biar/issues/110)
**Author:** Sandy-at-Tazama
**State:** Open
**Report Date:** 2026-07-08
**Verified against:** biar `ef0c856` (dev), frms-coe-lib `513d91d` (dev), Full-Stack-Docker-Tazama `e218211` (dev), case-management-system `2b14c12f` (dev), live NiFi at `http://10.10.80.19:8088`, live JupyterHub at `http://10.10.80.19:8888`.

---

## Executive Summary

`gold/transactions` in the Data Lakehouse is currently built by `TransactionsETL` reading `bronze/pacs008` + `bronze/pacs002`, then hydrating pacs.008-only fields (`Amt`, `Ccy`, party names, exchange rate) and pacs.002-only fields (`TxSts`) via ISO 20022 JSON-path extractions. This is the wrong source. The correct source is the TMS-normalised `event_history.transaction` ODS table, which carries a flat record for every transaction (both pacs.008 and pacs.002 in one row) plus foreign-key columns `source` and `destination` referencing `event_history.account(id, tenantId)`.

Two mechanisms produce the wrong state and both need reversing:

1. The automation orchestrator's `_route_etl` at `lakehouse_automation_pipeline.py:158-163` explicitly skips every `transaction`/`transactions` batch that NiFi sends, returning `"Skipped: transactions are derived from PACS gold"`. NiFi's `Transaction` (singular) processor **is running live** and polling `event_history.transaction` correctly — every batch is being discarded by the orchestrator on receipt.
2. `CombinedPacsETL` uses marker files to detect when both pacs pipelines reach Gold, then invokes `TransactionsETL(mode="from_pacs")` — that is the only path that populates `bronze/transactions`. Pacs008ETL and Pacs002ETL have no other coupling to each other; the "combining" adds no independent value.

The rewrite: `TransactionsETL` reads from `{bucket}/transaction/` (Ozone landing from the live NiFi `Transaction` processor), removes `_join_mode`, extracts flat TMS fields, derives party identifiers via the `entity` join chain (`source → account → account_holder → entity.id` and symmetrically for `destination`), and drops-and-rebuilds `bronze/transaction` / `silver/transaction` / `gold/transaction` as singular-named tables. `CombinedPacsETL` is deleted; pacs008 and pacs002 are registered directly in `_ETL_REGISTRY`. The skip guard is removed and replaced by registry entries `"transaction": TransactionsETL`, `"transactions": TransactionsETL`. All references to pacs-specific settlement/exchange fields at the Gold layer are deferred: MVP `gold/transaction` has no pacs enrichment, and consumers that need `xchg_rate` / `intr_bk_sttlm_amt` / `instg_mmb_id` etc. add a short-term view-layer `LEFT JOIN gold/pacs008 ON (end_to_end_id, tenant_id)`. Post-MVP, `TransactionsETL` will do that join at the Gold layer.

---

## How It Works Today — Confirmed in Code

### 1. `TransactionsETL` sources from pacs bronze

[`TransactionsETL.py:108-119`](../../../repos/biar/automation-orchestrator/Table_ETLs/TransactionsETL.py):

```python
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
```

`_transactions_from_pacs` (lines 72–102) aliases the pacs `document`/`document_json` payload as `transactionData` — the source of #108's naming collision. Silver (lines 222–292) then extracts pacs-specific paths (`$.FIToFICstmrCdtTrf.…` and `$.FIToFIPmtSts.…`) into flat columns. Gold (lines 298–342) is a scalar-only selection.

### 2. Orchestrator skip guard

[`lakehouse_automation_pipeline.py:158-163`](../../../repos/biar/automation-orchestrator/lakehouse_automation_pipeline.py):

```python
if table in ("transaction", "transactions"):
    print(
        "Skipping standalone Transactions ETL: it is triggered only after "
        "pacs008 + pacs002 reach GOLD (via CombinedPacsETL)."
    )
    return "Skipped: transactions are derived from PACS gold"
```

Any FlowFile with `table=transaction` or `table=transactions` is discarded. NiFi receives HTTP 200 and never retries.

### 3. NiFi `Transaction` (singular) processor is LIVE

**Live cross-check at `http://10.10.80.19:8088/nifi-api/process-groups/root/processors`** (2026-07-08):

| Property | Value |
|---|---|
| Name | `QueryDatabaseTableRecord` |
| Table Name | `transaction` |
| Watermark | `credttm` |
| State | `RUNNING` |
| DBCP pool id | `5302abbb-b2b0-3f15-4821-b96045921265` |
| Pool name / URL | `event_history` / `jdbc:postgresql://10.10.80.18:15432/event_history` |

Downstream chain (confirmed from `/nifi-api/process-groups/root/connections`):

```
QueryDatabaseTableRecord(transaction, event_history)
   → UpdateAttribute 54e0b058-…   (sets table=transaction, bucket=#{pbucket}, object_key=…)
   → PutS3Object aa026213-…       (writes to s3a://${bucket}/transaction/…)
   → ReplaceText 81dd19c9-…       (Template A payload — no db_name field)
   → InvokeHTTP → orchestrator
```

**Payload template (Template A)** from `ReplaceText 81dd19c9-…`:

```json
{
  "raw_path": "s3a://${bucket}/${table}/${object_key}",
  "bucket": "${bucket}",
  "table": "${table}",
  "object_key": "${object_key}",
  "execute_notebook": true
}
```

No `db_name` in payload. The orchestrator receives `table=transaction` and hits the skip guard (§2 above) *before* `db_name` is inspected. So the batch is discarded cleanly, no crash. Ozone S3 accumulates JSON files at `${bucket}/transaction/` on every poll cycle that returns rows, and those files are read by nobody today.

### 4. `CombinedPacsETL` is the only path that triggers `TransactionsETL`

[`CombinedPacs.py:113-172`](../../../repos/biar/automation-orchestrator/Table_ETLs/CombinedPacs.py):

```python
def run(self, source_path: str) -> str:
    if self.table == "pacs008": Pacs008ETL(...).run(source_path)
    else:                       Pacs002ETL(...).run(source_path)
    # touch marker for self.table, check both markers
    pacs008_recent = self._is_recent_file(...)
    pacs002_recent = self._is_recent_file(...)
    if pacs008_recent and pacs002_recent:
        TransactionsETL(...).run(source_path, mode="from_pacs")
```

Nothing else calls `TransactionsETL`. `_route_etl` at `lakehouse_automation_pipeline.py:150-176` dispatches `pacs008` and `pacs002` through `CombinedPacsETL(table=…)`; and `transaction`/`transactions` through the skip guard. There is no path that would invoke `TransactionsETL(mode="join")` in current live use.

### 5. `event_history.transaction` — authoritative DDL

[`Full-Stack-Docker-Tazama/core/postgres/migration/base/00-CREATE.sql:162-197`](../../../repos/Full-Stack-Docker-Tazama/core/postgres/migration/base/00-CREATE.sql):

```sql
create table transaction (
    source varchar not null,
    destination varchar not null,
    transaction jsonb not null,
    endToEndId text     generated always as (transaction->>'EndToEndId') stored,
    amt        numeric(18, 2) generated always as ((transaction->>'Amt')::numeric(18, 2)) stored,
    ccy        varchar generated always as (transaction->>'Ccy') stored,
    msgId      varchar generated always as (transaction->>'MsgId') stored,
    creDtTm    text    generated always as (transaction->>'CreDtTm') stored,
    txTp       varchar generated always as (transaction->>'TxTp') stored,
    txSts      varchar generated always as (transaction->>'TxSts') stored,
    tenantId   text    generated always as (transaction->>'TenantId') stored,
    constraint unique_msgid unique (msgId, tenantId),
    foreign key (source, tenantId) references account (id, tenantId),
    foreign key (destination, tenantId) references account (id, tenantId),
    primary key (endToEndId, txTp, tenantId)
);
```

Confirms every field the issue claims. `source` and `destination` are FKs to `account(id, tenantId)` — TMS debtor and creditor account IDs. Amt/Ccy/TxSts are all present as generated-stored columns on the same row.

### 6. `event_history.entity` has no name column

[`00-CREATE.sql:83-88`](../../../repos/Full-Stack-Docker-Tazama/core/postgres/migration/base/00-CREATE.sql):

```sql
create table entity (
    id varchar not null,
    tenantId text not null,
    creDtTm timestamptz not null,
    primary key (id, tenantId)
);
```

`Dbtr.Nm` / `Cdtr.Nm` are transient pacs.008 message fields; they are not persisted to the ODS. `entity.id` is composed as `PrvtId.Othr[0].Id + SchmeNm.Prtry` per `getEntity` at `frms-coe-lib/src/builders/eventHistoryBuilder.ts:244-249`.

### 7. `event_history.account_holder` — the join fabric

[`00-CREATE.sql:90-98`](../../../repos/Full-Stack-Docker-Tazama/core/postgres/migration/base/00-CREATE.sql):

```sql
create table account_holder (
    source varchar not null,           -- entity.id
    destination varchar not null,       -- account.id
    tenantId text not null,
    creDtTm timestamptz not null,
    foreign key (source, tenantId) references entity (id, tenantId),
    foreign key (destination, tenantId) references account (id, tenantId),
    primary key (source, destination, tenantId)
);
```

So the join chain the issue proposes — `transaction.source → account.id → account_holder.destination → account_holder.source = entity.id` (and symmetrically for `transaction.destination`) — is exact.

---

## Root Cause Analysis

`gold/transactions` was designed under the assumption that pacs messages are the primary transaction source. The TMS ODS layer, which produces a normalised `event_history.transaction` record for every transaction (both pacs.008 and pacs.002 in one row, with FK-validated `source` / `destination` account IDs, and a JSONB payload that flat-projects all business fields including `TxSts`, `Amt`, `Ccy`, `EndToEndId`, `MsgId`, `CreDtTm`, `TxTp`, `TenantId`), was not wired in. The NiFi `Transaction` processor that would land the data was built and is running, but the orchestrator's `_route_etl` was configured to discard every batch. `CombinedPacsETL` was written to keep the pacs pipeline in charge of feeding `TransactionsETL`. All symptoms — the missing `source` / `destination` in `gold/transactions`, the reliance on pacs-only fields, the naming collision documented in #108 — flow from this single wrong-source decision.

---

## Blast Radius — Full File Inventory

### Files that must change (Track B — the rewrite)

| File | What's affected | Lines |
|---|---|---|
| `automation-orchestrator/Table_ETLs/TransactionsETL.py` | Rewrite `bronze`/`silver`/`gold` to source from `event_history.transaction`. Remove `_join_mode`. Populate `source`, `destination`, `debtor_entity_id`, `creditor_entity_id`. Rename paths to `bronze/transaction` (singular) etc. | Entire file |
| `automation-orchestrator/lakehouse_automation_pipeline.py` | Remove the `transaction`/`transactions` skip guard (lines 158–163). Add `"transaction": TransactionsETL`, `"transactions": TransactionsETL` to `_ETL_REGISTRY` (lines 51–68). Remove the `CombinedPacsETL` dispatch for pacs008/pacs002 and add direct `Pacs008ETL`/`Pacs002ETL` entries. | 51–68, 152–163 |
| `automation-orchestrator/Table_ETLs/CombinedPacs.py` | Delete the file | Entire file |
| `automation-orchestrator/Table_ETLs/views_orchestrator.py` | Change readiness checks from `bronze/transactions` to `bronze/transaction` (lines 71, 79) and `gold/transactions` to `gold/transaction` (line 108). | 71, 79, 108 |
| `data-lineage.md` | Update all references to plural `*/transactions` and to the pacs-fed pipeline | Entire "transaction" section |
| `.pipeline_state/pacs008_gold_done`, `.pipeline_state/pacs002_gold_done`, `.pipeline_state/transactions_trigger.lock` | Delete marker files (runtime state, not repo) | — |
| Existing Hudi tables `bronze/transactions`, `silver/transactions`, `gold/transactions` | Drop and let the new pipeline rebuild singular-named tables | Runtime state |

### Files consumed indirectly (Track B — but tracked in #111)

See [issues/biar/111](../111/) for the full 14-consumer inventory (including `ConditionsTimelineView.py` which was merged to dev in PR #92 after the initial accuracy review; see §"Cross-Issue Dependencies" below).

---

## Side-Effect Map

| Consequence if left unfixed | Impact |
|---|---|
| `gold/transactions` lacks `source` / `destination` — no way to link to `account` or `entity` cleanly; consumers reconstruct via ISO paths that don't work for pacs.002 rows | High — CMS network / counterparty graphs incomplete |
| Every pacs.002 row has null `Amt`/`Ccy`; every pacs.008 row has null `TxSts` | High — analytics queries silently produce nulls |
| Party names sourced from `Dbtr.Nm`/`Cdtr.Nm` are null on pacs.002 rows; CMS transaction detail view shows blank counterparty on payment-status messages | Medium-high |
| Every NiFi poll of `event_history.transaction` writes to Ozone and is immediately ignored | Wasted storage + wasted compute |
| `CombinedPacsETL` couples pacs008 and pacs002 through marker files; either pacs pipeline slow → transactions pipeline slow | Medium (throughput) |
| Naming collision (#108) persists and traps the next dev | Chronic confusion tax |
| Anomaly Detection notebook (#111) uses `instg_mmb_id`/`instd_mmb_id` as join keys + features; those disappear in MVP unless the short-term pacs join is added | High for that consumer |

---

## Effort Assessment

### Track A — Minimal / immediate fix (not applicable)

There is no meaningful minimal Track A for this issue. Any partial fix leaves either:

- The pacs-fed pipeline still populating `gold/transactions` (defeats the purpose), or
- The skip guard removed without the ETL rewrite (breaks the orchestrator by trying to run a still-pacs-oriented ETL against Ozone JSON files it doesn't know how to read).

The nearest thing to a Track A is a documentation-only change: add a warning to `data-lineage.md` and `TransactionsETL.py` explaining the wrong source and the plan. Effort: ~30 min. Value: minimal.

### Track B — Full rewrite (recommended, and the only real option)

| Step | File(s) | Effort |
|---|---|---|
| B1: Rewrite `TransactionsETL.bronze` to read `{bucket}/transaction/` and produce singular `bronze/transaction` with flat TMS fields + `source`/`destination` | `TransactionsETL.py` | ~1 day |
| B2: Rewrite `TransactionsETL.silver` to derive `debtor_entity_id`/`creditor_entity_id` via `account → account_holder → entity` joins; keep silver metrics (event_ts, event_date) sourced from flat `creDtTm` | `TransactionsETL.py`, may need Spark reads from `bronze/account`, `bronze/account_holder`, `bronze/entity` | ~1 day |
| B3: Rewrite `TransactionsETL.gold` to project scalar TMS-normalised fields; **no pacs enrichment for MVP** | `TransactionsETL.py` | ~0.25 day |
| B4: Delete `CombinedPacs.py`; add direct pacs registry entries | `lakehouse_automation_pipeline.py`, delete `CombinedPacs.py` | ~0.5 day |
| B5: Remove skip guard; add `transaction`/`transactions` registry entries | `lakehouse_automation_pipeline.py` | ~0.25 day |
| B6: Update `views_orchestrator.py` readiness paths | `views_orchestrator.py` | ~0.1 day |
| B7: Rename Hudi paths to singular in the ETL, drop old plural tables | `TransactionsETL.py`, runtime | ~0.25 day |
| B8: Update `data-lineage.md` | `data-lineage.md` | ~0.5 day |
| B9: Tighten view-builder candidate lists (#108 Track A) | 3 view-builder files | ~0.25 day |

**Schema migration required:** No PostgreSQL schema changes; Hudi tables are drop-and-rebuild.
**Frontend changes required:** No — but consumers (see #111) need coordinated updates.

### Recommended sequencing

1. **Ship #109 first** (independent, small NiFi cleanup — reduces cross-contamination risk during #110 rollout).
2. **Ship #110 as a single atomic PR family** — the ETL rewrite, skip guard removal, `CombinedPacs` deletion, view-builder candidate-list hardening (from #108 Track A), and `views_orchestrator.py` paths must land together.
3. **Ship #111 immediately after #110** — do not leave consumers on the old plural table names for long; consumer coverage is documented in [issues/biar/111](../111/).
4. **Post-MVP Gold enrichment** (Design Requirement 6 in the issue) — separate follow-up PR after #110/#111 stabilise. Adds pacs joins at Gold layer, eliminating per-consumer joins.

---

## Acceptance Criteria

### Track B

- [ ] `TransactionsETL` reads exclusively from `{bucket}/transaction/` — no pacs bronze reads.
- [ ] `_join_mode` removed.
- [ ] `bronze/transaction`, `silver/transaction`, `gold/transaction` (singular) exist; old plural tables dropped.
- [ ] `CombinedPacsETL` file deleted; `Pacs008ETL` / `Pacs002ETL` registered directly.
- [ ] Skip guard removed from `_route_etl`.
- [ ] `_ETL_REGISTRY` has `"transaction": TransactionsETL` and `"transactions": TransactionsETL`.
- [ ] `gold/transaction` schema includes `source`, `destination`, `debtor_entity_id`, `creditor_entity_id`, `end_to_end_id`, `tx_type`, `tx_status`, `tx_amount`, `tx_ccy`, `tx_msg_id`, `event_ts`, `event_date`, `tenant_id`, `ingested_at_ts`, `event_to_ingest_ms`, `source_file_path`, `record_hash`.
- [ ] `gold/transaction` does **not** include `intr_bk_sttlm_amt`, `xchg_rate`, `instg_mmb_id`, `instd_mmb_id`, `charge_count`, `debtor_name`, `creditor_name` — those are deferred to post-MVP (see Design Requirement 6).
- [ ] View builders' candidate lists tightened to explicit `expected = "transaction"` binding (#108 Track A).
- [ ] `views_orchestrator.py` uses singular paths.
- [ ] `data-lineage.md` refreshed.
- [ ] All #111 consumers coordinated (see [111](../111/)).

---
