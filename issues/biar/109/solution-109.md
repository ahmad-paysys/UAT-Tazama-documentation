# Complete Fix — Exact Code Changes

Track A (below) can merge independently as a small NiFi cleanup PR. Track B is deferred and only applies if a real Lakehouse requirement for `tazama_cms.transaction_data` emerges.

---

### Track A — Remove `Transaction_Data` processor from committed NiFi flow (PR 1 of 1)

#### File: `biar/nifi/tazama.xml`

NiFi flow XML edits are id-based, not text-based. The processor to remove is:

- Processor id: locate by `<name>` / `<label>` = `Transaction_Data` and `<Table Name>transaction_data</Table Name>` (committed at approx. line 5905 for the label and line 26173 for the table name).
- Its DBCP pool reference: `b0b69311-8545-36e2-0000-000000000000` (the `tazama_cms` pool). **Do not** remove the pool itself — it is shared with `tasks`, `cms_usernames`, and other live processors.

**BEFORE** (schematic — the processor block, approximate lines 26130–26260):

```xml
<processors>
  <id>{PROC_ID}</id>
  <name>Transaction_Data</name>
  ...
  <properties>
    <entry><key>Database Connection Pooling Service</key>
           <value>b0b69311-8545-36e2-0000-000000000000</value></entry>
    <entry><key>Table Name</key><value>transaction_data</value></entry>
    <entry><key>Maximum-value Columns</key><value>"createdAt"</value></entry>
    ...
  </properties>
</processors>
```

**AFTER:** the entire `<processors>…</processors>` block for `Transaction_Data` is removed. All `<connections>` blocks whose `source` or `destination` id equals `{PROC_ID}` are also removed. If any downstream processors (`UpdateAttribute`, `ReplaceText`, `PutS3Object`) were exclusive to this branch (i.e. their only inbound edge was from `{PROC_ID}`), they are also removed. If they are shared with other flows, they stay.

Also remove any `<label>Transaction_Data</label>` block (approx. line 5905) if it exists as a canvas label.

##### Recipe for editors

The safe way to perform this removal is:

1. Import the current committed `tazama.xml` into a scratch NiFi instance (empty flow file). Locate the `Transaction_Data` processor on the canvas.
2. Follow every outbound edge; note the ids of every downstream processor and the ids of every connection.
3. For each downstream processor, check if it has any other inbound edge. If not, it belongs to the `Transaction_Data` branch and must be removed too. If yes (shared), only the connection is removed.
4. Delete the identified processors and connections through the NiFi UI.
5. Export the modified flow. Diff against the committed `tazama.xml`; the diff should be limited to the identified ids.

Doing this by direct XML text-edit is error-prone because id references cross-cut the file. Use the NiFi UI as the source of the edit, not `sed`.

---

### Track B — Add proper handling (deferred; only when requirement lands)

Not part of this PR. When needed:

#### Step B1 — Choose a safe Hudi record key

Not `transactionId` (autoincrement, not idempotent). Prefer `endToEndId + tenantId` (composite), or `endToEndId + tenantId + createdAt` if same-e2e-id updates need to be tracked separately.

#### Step B2 — Add ETL config

**File:** `automation-orchestrator/Table_ETLs/DynamicETL.py`, lines 54–80.

**BEFORE:**

```python
def _resolve_config(self, db_name: str) -> dict:
    configs = {
        "raw_history":   { "record_key": "messageid",  "precombine": "credttm", ... },
        "event_history": { "record_key": "_key",       "precombine": "credttm", ... },
        "enrichment":    { "record_key": "id",         "precombine": "created_at", ... },
    }
    if db_name not in configs:
        raise RuntimeError(...)
```

**AFTER (Option 1 — add `tazama_cms` to DynamicETL):**

```python
configs = {
    "raw_history":   { ... },
    "event_history": { ... },
    "enrichment":    { ... },
    "tazama_cms":    { "record_key": "end_to_end_id_tenant_id",  # composite key emitted by ETL
                       "precombine": "createdAt",
                       "canonical_cols": {"end_to_end_id_tenant_id": "string", "createdAt": "timestamp"} },
}
```

**AFTER (Option 2 — dedicated `TransactionDataETL`):** add a new class in `automation-orchestrator/Table_ETLs/TransactionDataETL.py` and register it in `_ETL_REGISTRY` at `lakehouse_automation_pipeline.py:51-68`:

```python
_ETL_REGISTRY: dict[str, type] = {
    ...
    "transaction_data": TransactionDataETL,
}
```

Option 2 is preferred if the CMS payload needs bespoke JSON extraction; Option 1 is simpler if standard flattening is sufficient.

#### Step B3 — Re-enable the NiFi processor

If Option 2 is chosen, ensure the payload template used by `Transaction_Data`'s downstream `ReplaceText` is Template A (no `db_name` needed, `table=transaction_data` routes via `_ETL_REGISTRY`). If Option 1 is chosen, ensure Template B is used and `db_name=tazama_cms` is set on the FlowFile.

---

### Test cases

#### Unit tests to add (Track B only, when built)

- `test_transaction_data_record_key_is_composite`: assert the ETL emits `end_to_end_id + tenantId` as record key, not `transactionId`.
- `test_transaction_data_precombine_is_created_at`: assert precombine value is `createdAt`.

#### Integration / E2E scenarios (Track A)

1. Fresh NiFi cluster: import edited `tazama.xml`. Verify no startup errors, no dangling connection references, no orphan processors.
2. On existing live NiFi: no expected change (was already absent). Compare processor count and running state before/after: identical.
3. `grep -c "transaction_data" biar/nifi/tazama.xml` returns `0`.

#### Data migration validation SQL

None for Track A.

#### Manual / UAT checks

| # | Scenario | Steps | Expected Result |
|---|---|---|---|
| 1 | Sanity of live flow | Query `http://10.10.80.19:8088/nifi-api/process-groups/root/processors` for `Table Name = transaction_data` | Zero results (unchanged) |
| 2 | Import to scratch NiFi | Import edited `tazama.xml` into a fresh NiFi container | No errors, flow starts cleanly |
| 3 | Downstream processor audit | Trace every removed processor id in the diff — confirm none are referenced by any remaining `<connections>` | No dangling ids |

---

### Overall Impact of the Fix

| Area | Before | After |
|---|---|---|
| Committed NiFi flow | Contains `Transaction_Data` processor whose batches would crash orchestrator on redeploy | Processor absent; no latent hazard |
| Live NiFi flow | Already absent | Unchanged |
| Naming collision (see #108) | 1 of 5 vectors — the NiFi processor | Removed; leaves 4 vectors, addressed by #110 |
| Future-work debt | Latent XML block waiting to bite on redeploy | Clean — future ingestion (if needed) starts from a blank slate |

---

### Fix Summary

The bug is that a NiFi processor was committed without a corresponding orchestrator handler. Enabling the processor would raise `RuntimeError` from `DynamicETL._resolve_config()` on every non-empty poll cycle, because `tazama_cms` is not on the allowlist. Live confirms the processor is not currently running — but it exists in the committed XML and would come back on any fresh redeploy.

Track A removes the processor block from `biar/nifi/tazama.xml` and any of its exclusive downstream components. It has no code impact on the Python orchestrator, no schema impact, no frontend impact, and no downtime. It is safe to ship in isolation from #108, #110, or #111.

Track B (proper ingestion) is deferred until a real Lakehouse consumer for `tazama_cms.transaction_data` exists. When it does, the safe Hudi record key is a composite of `endToEndId + tenantId` — not the CMS's autoincrement `transactionId`, which is non-idempotent across CMS reinstalls.
