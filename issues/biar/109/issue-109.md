# Issue #109 — DLH BUG: tazama_cms.transaction_data Pipeline Crashes Automation Orchestrator on Every Poll

**Repository:** tazama-lf/biar
**Issue:** [DLH BUG: tazama_cms.transaction_data Pipeline Crashes Automation Orchestrator on Every Poll](https://github.com/tazama-lf/biar/issues/109)
**Author:** Sandy-at-Tazama
**State:** Open
**Report Date:** 2026-07-08
**Verified against:** biar `ef0c856` (dev), live NiFi at `http://10.10.80.19:8088`, Full-Stack-Docker-Tazama `e218211` (dev), case-management-system `2b14c12f` (dev).

---

## Executive Summary

The committed NiFi flow at `biar/nifi/tazama.xml` includes a `Transaction_Data` processor that polls the CMS ODS table `tazama_cms.transaction_data` and hands the payload to the automation orchestrator. There is no known handler for `tazama_cms` on the orchestrator side — `DynamicETL._resolve_config()` recognises only `raw_history`, `event_history`, and `enrichment`, and raises `RuntimeError` for anything else. If the processor is ever enabled with a payload template that carries `db_name=tazama_cms` (Template B pattern in the flow), every non-empty poll cycle crashes the orchestrator's ETL job. There is no consumer of `tazama_cms.transaction_data` anywhere in the Lakehouse: it is a CMS-side operational table with no current Lakehouse requirement.

**Live-flow finding (2026-07-08):** the running NiFi at `http://10.10.80.19:8088` does **not** currently include a `Transaction_Data` processor. Deployed processors include `transaction` (from `event_history`), `pacs008` (from `raw_history`), `tasks` (from `tazama_cms`) and 15 others — but not `transaction_data`. The bug as described is a latent property of the committed flow; it is not causing a live incident today, but it will the moment the processor is re-enabled or the committed flow is redeployed.

The remediation is unchanged from the issue's proposal: remove the `Transaction_Data` processor (and any related routing) from the committed NiFi flow. The future-requirements section of the issue (adding `tazama_cms` to `_ETL_REGISTRY` or `DynamicETL._resolve_config()` if this table ever becomes a Lakehouse requirement) remains valid advice but is not current work.

---

## How It Works Today — Confirmed in Code

### 1. NiFi `Transaction_Data` processor (committed but not deployed)

Committed [`biar/nifi/tazama.xml`](../../../repos/biar/nifi/tazama.xml):

| Setting | Value | Source |
|---|---|---|
| Label | `Transaction_Data` | line 5905 |
| Table Name | `transaction_data` | line 26173 |
| Watermark column | `"createdAt"` (double-quoted, camelCase) | line 26190 |
| Fetch Size | 500 | line 26202 |
| Connection Pool | `b0b69311-8545-36e2-…` | line 26165 |
| Pool name / URL | `tazama_cms` / `jdbc:postgresql://10.10.80.18:15433/tazama_cms` | lines 4779, 4784 |

### 2. Live NiFi (2026-07-08)

Queried via `http://10.10.80.19:8088/nifi-api/process-groups/root/processors`:

- 158 processors, all `RUNNING`.
- Distinct `Table Name` values: `account, account_holder, alerts, cases, cms_usernames, comments, condition, entity, evaluation, network_map, pacs002, pacs008, pain001, pain013, rule, tasks, transaction, typology`.
- No processor with `Table Name = transaction_data`.

So the live flow at snapshot time does not include this processor. It is *committed* in `nifi/tazama.xml` but not *deployed*.

### 3. Payload templates — two variants coexist

Live inspection shows the flow uses two distinct orchestrator-payload templates via `ReplaceText` processors:

**Template A** (used by `transaction`, `pacs008`, `tasks`, and 5 other tables):

```json
{
  "raw_path": "s3a://${bucket}/${table}/${object_key}",
  "bucket": "${bucket}",
  "table": "${table}",
  "object_key": "${object_key}",
  "execute_notebook": true
}
```

No `db_name`. Table name is `transaction`, `pacs008`, etc.

**Template B** (used by the dynamic-discovery flow at `ReplaceText` id `03da3a70-…`):

```json
{
  "raw_path": "s3a://#{pbucket}/${db_name}_${table_name}/${object_key}",
  "bucket": "#{pbucket}",
  "db_name": "${db_name}",
  "table": "${table_name}",
  "object_key": "${object_key}",
  "execute_notebook": true
}
```

This is the template whose `${db_name}` attribute resolves via the `current_database()` NiFi SQL discovery query the issue references.

### 4. Orchestrator handling — `_route_etl`

[`lakehouse_automation_pipeline.py:150-176`](../../../repos/biar/automation-orchestrator/lakehouse_automation_pipeline.py):

```python
def _route_etl(self, table, source_path, db_name=None) -> str:
    if table in ("pacs008", "pacs002"):
        CombinedPacsETL(...).run(source_path); return "All Done"
    if table in ("transaction", "transactions"):
        ...; return "Skipped: transactions are derived from PACS gold"
    if table in self._ETL_REGISTRY:
        self._ETL_REGISTRY[table](...).run(source_path); return "All Done"
    # FALLBACK — unknown table → DynamicETL with db_name
    DynamicETL(self.spark, self.warehouse_root, db_name=db_name, table=table).run(source_path)
    return "All Done (DynamicETL fallback)"
```

`transaction_data` is not in `_ETL_REGISTRY` at lines 51–68. It falls through to `DynamicETL`.

### 5. `DynamicETL._resolve_config()` — the crash point

[`DynamicETL.py:54-80`](../../../repos/biar/automation-orchestrator/Table_ETLs/DynamicETL.py):

```python
def _resolve_config(self, db_name: str) -> dict:
    configs = {
        "raw_history":   {...},
        "event_history": {...},
        "enrichment":    {...},
    }
    if db_name not in configs:
        raise RuntimeError(
            f"[DynamicETL] Unidentified db: '{db_name}'. "
            f"Cannot process. Supported db_names: {list(configs.keys())}"
        )
    return configs[db_name]
```

For a payload with `db_name=tazama_cms`, this raises immediately.

### 6. No Lakehouse consumer

Full-tree grep in `biar/` for `tazama_cms.transaction_data` returns zero non-NiFi hits. All references to the token `transaction_data` in Python view-builder code refer to the local pacs-payload alias (see #108), not to this CMS table.

### 7. `tasks` from `tazama_cms` *does* have live handling

`tasks` is deployed live and routes to `TasksETL` via `_ETL_REGISTRY` — no `DynamicETL` fallback needed, so the fact that `tazama_cms` isn't in the DynamicETL config dict is irrelevant for `tasks`. The gap only bites tables that are (a) in `tazama_cms` and (b) not in `_ETL_REGISTRY`.

---

## Root Cause Analysis

`DynamicETL._resolve_config()` is a strict allowlist keyed on database name. `tazama_cms` is not on the allowlist because there was no design intent to ingest anything from `tazama_cms` beyond the handful of tables that already have dedicated ETL classes (`tasks`, `cms_usernames`). The `Transaction_Data` processor was committed to `nifi/tazama.xml` at some point without a corresponding orchestrator-side handler; every non-empty batch it sends to `_route_etl` would fall through to `DynamicETL` with `db_name=tazama_cms` and crash. The processor was subsequently disabled or not deployed to the live flow — but the committed XML retains it.

---

## Blast Radius — Full File Inventory

### Files that must change

| File | What's affected | Lines |
|---|---|---|
| `biar/nifi/tazama.xml` | Remove or disable the `Transaction_Data` processor and any UpdateAttribute / ReplaceText / connection references specific to it | ~5905 (label), 26173 (table name), 26165 (pool), plus surrounding processor block and connections |

### Files indirectly affected

| File | Why |
|---|---|
| `automation-orchestrator/Table_ETLs/DynamicETL.py` | Would need `tazama_cms` config only if the table ever becomes a real Lakehouse requirement |
| `automation-orchestrator/lakehouse_automation_pipeline.py` | Would need a `_ETL_REGISTRY` entry only if the table ever becomes a real Lakehouse requirement |

---

## Side-Effect Map

| Consequence if left unfixed | Impact |
|---|---|
| If the committed NiFi flow is redeployed (e.g. via `nifi/init.sh` on a fresh cluster), the `Transaction_Data` processor comes back and every non-empty poll cycle crashes the orchestrator | High — silent hazard on redeploy |
| Naming collision with view builders (see #108) misleads readers into thinking Lakehouse coverage exists | Medium — chronic confusion tax |
| Once #110 renames the `transaction` table pipeline, any residual `transaction_data`-related state may be revived by mistake | Low but real — clean removal now prevents cross-contamination during #110 |

---

## Effort Assessment

### Track A — Disable / remove the NiFi processor (recommended)

Delete or disable the `Transaction_Data` processor block in `biar/nifi/tazama.xml`. Also delete the downstream `UpdateAttribute` / `ReplaceText` / `PutS3Object` / `InvokeHTTP` connections that are exclusive to this branch (if any exist in the committed XML; they need to be traced by processor id since XML editing at this scale is error-prone).

| Step | File | Effort |
|---|---|---|
| Locate the `Transaction_Data` processor block by id and remove | `biar/nifi/tazama.xml` | ~1 h (including tracing downstream connections and verifying no shared components are pruned) |
| Regenerate / redeploy the flow to a test NiFi instance to confirm no dangling references | test env | ~30 min |
| Verify against live NiFi that behaviour is unchanged (live doesn't have it anyway) | live NiFi | ~10 min |

Schema migration: no. Frontend changes: no.

### Track B — Add proper handling if ingestion becomes a requirement (deferred)

If `tazama_cms.transaction_data` is ever needed in the Lakehouse (currently no requirement):

1. **`_ETL_REGISTRY`**: add `"transaction_data": TransactionDataETL` pointing to a new dedicated ETL class.
2. **Field mapping**: a dedicated ETL is needed to handle Prisma camelCase columns (`transactionId`, `transactionData`, `endToEndId`, `tenantId`, `createdAt`) and to select an appropriate record key. **Warning:** the issue proposes `record_key="transactionId"`, but per the CMS Prisma schema `transactionId` is `Int @default(autoincrement())` — a synthetic autoincrement not idempotent across CMS reinstalls or restores. Use `endToEndId + tenantId` or `endToEndId + tenantId + createdAt` as the Hudi record key instead.
3. **Naming**: ensure any new Hudi table name is unambiguous — do not re-introduce the `transaction_data`/`transactionData` collision documented in #108.

None of this is current work.

### Recommended sequencing

1. Ship Track A (remove from committed NiFi flow) alongside the #110 PR family or as an independent small NiFi cleanup PR.
2. Defer Track B until a real product requirement lands.

---

## Acceptance Criteria

### Track A

- [ ] `Transaction_Data` processor and its exclusive downstream chain removed from `biar/nifi/tazama.xml`.
- [ ] Redeploying the flow to a fresh NiFi cluster produces a running flow with no `transaction_data` table processor.
- [ ] Grep of the committed `tazama.xml` for `transaction_data` / `Transaction_Data` returns zero hits.
- [ ] No regressions in other live processors (`tasks` and `cms_usernames` still poll `tazama_cms` correctly).

### Track B (deferred — enable only when the requirement is signed off)

- [ ] Dedicated `TransactionDataETL` class exists with correct Hudi record_key selection (not `transactionId`).
- [ ] `_ETL_REGISTRY` entry added.
- [ ] Field mapping documented.
- [ ] Naming reviewed against #108 principle.

---
