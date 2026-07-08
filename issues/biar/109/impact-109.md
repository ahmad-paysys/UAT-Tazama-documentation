# Impact Study — Issue #109: tazama_cms.transaction_data Pipeline Crashes Automation Orchestrator on Every Poll

**Issue:** [#109](https://github.com/tazama-lf/biar/issues/109)
**Study Date:** 2026-07-08
**Author:** Ahmad Khalid
**Repository:** tazama-lf/biar

---

## Summary

The committed NiFi flow includes a `Transaction_Data` processor that reads `tazama_cms.transaction_data` and forwards batches to the automation orchestrator. The orchestrator's `DynamicETL` fallback recognises only three databases (`raw_history`, `event_history`, `enrichment`) and raises `RuntimeError` on any others, so a `tazama_cms` batch would crash every non-empty poll cycle. There is no Lakehouse consumer for this data. **Track A** removes the processor from the committed NiFi flow — safe in isolation, and — per live verification — the deployed flow at `10.10.80.19:8088` does not currently include the processor, so there is no live incident to fix, only a latent hazard on redeploy. **Track B** (add proper handling if the table ever becomes a Lakehouse requirement) is deferred.

---

## Confirmed Root Cause

- `Transaction_Data` processor committed in `biar/nifi/tazama.xml` (label at line 5905; `Table Name = transaction_data` at line 26173; pool `b0b69311-…` → `tazama_cms` at lines 4779, 4784).
- Watermark column is `"createdAt"` (Prisma camelCase, double-quoted) at line 26190.
- `DynamicETL._resolve_config()` at `DynamicETL.py:54-80` allowlists only `raw_history`, `event_history`, `enrichment` and raises `RuntimeError` for anything else.
- Orchestrator `_route_etl` at `lakehouse_automation_pipeline.py:150-176` sends anything not in `_ETL_REGISTRY` and not in the pacs/transaction guards to `DynamicETL`.
- `transaction_data` is absent from `_ETL_REGISTRY` (lines 51–68 in the same file).
- **Live NiFi at 10.10.80.19:8088 (2026-07-08) does not contain a running `Transaction_Data` processor.** Distinct table names in the deployed flow: account, account_holder, alerts, cases, cms_usernames, comments, condition, entity, evaluation, network_map, pacs002, pacs008, pain001, pain013, rule, tasks, transaction, typology.
- **CMS Prisma schema (`case-management-system/backend/prisma/schema.prisma` on dev):** model `TransactionData` → table `transaction_data` with columns `transactionId Int @id @default(autoincrement())`, `tenantId`, `endToEndId`, `transactionData Json`, `createdAt`. **The suggested `record_key="transactionId"` is unsafe** — autoincrement primary keys are not idempotent across CMS reinstalls.

---

## Track A — Disable / remove the processor in the committed NiFi flow

### What Changes

Remove the `Transaction_Data` processor and its exclusive downstream chain from `biar/nifi/tazama.xml`. Roughly ~100–200 lines of XML depending on how downstream `UpdateAttribute`, `ReplaceText`, `PutS3Object`, and `InvokeHTTP` are structured. No code changes required in the Python orchestrator.

### Impact

| Property | Value |
|---|---|
| Files changed | 1 (`nifi/tazama.xml`) |
| Schema migration required | No |
| Frontend changes required | No |
| Downtime required | No |
| Risk of regression | Low (only if downstream trace prunes shared processors — check by processor id, not name) |
| Reversibility | Trivial (revert the commit; NiFi allows re-import) |

What this fixes immediately:
- No latent crash risk on redeploy from committed XML.
- Removes a naming source of #108's collision (partial).

What this does not fix:
- The `transactionData` column in `bronze/transactions` (that's #110).
- The view-builder candidate-list trap (that's a Track A AC on #110 — see #108).

Track A is safe to ship in isolation.

---

## Track B — Proper `tazama_cms.transaction_data` ingestion (deferred)

### What Changes

Only if a real product requirement lands. Would introduce a `TransactionDataETL` class, a `_ETL_REGISTRY` entry, and — critically — a safe Hudi record key.

### Schema Impact

| Change | Note |
|---|---|
| New Hudi table `bronze/tazama_cms_transaction_data` (or similar unambiguous name) | Must avoid the naming collision documented in #108 |
| Record key selection | Do **not** use CMS `transactionId` (autoincrement). Use `endToEndId + tenantId` (or with `createdAt` for uniqueness across replays). |

### Backend Code Impact

| Area | Files (hypothetical for Track B) |
|---|---|
| New ETL class | `automation-orchestrator/Table_ETLs/TransactionDataETL.py` |
| Registry entry | `automation-orchestrator/lakehouse_automation_pipeline.py` |
| Payload template routing | `nifi/tazama.xml` (add / re-enable processor, ensure Template B carries `db_name=tazama_cms`) |
| DynamicETL config addition | Alternative to a dedicated ETL: add `tazama_cms` config profile to `DynamicETL._resolve_config()` |

### Frontend Code Impact

None; this is a backend/pipeline concern.

---

## Side Effects and Risks

### Risks of Track A Alone

| Risk | Likelihood | Mitigation |
|---|---|---|
| XML surgery accidentally removes a shared component id (e.g. a ReplaceText that other processors also feed into) | Medium | Delete by processor id, verify each connection endpoint before removal; regenerate the flow diff and eyeball for orphans; import to a fresh NiFi instance and run a start/stop cycle |
| A future dev copies `Transaction_Data` config from history and re-enables it, unaware of the missing orchestrator handler | Low | Add a comment near the (removed) block in the NiFi flow-hygiene doc, or a `# removed intentionally, see biar#109` marker in `data-lineage.md` |
| CI treats `tazama.xml` as an opaque blob and doesn't run any schema/reference validation | Medium | Consider a `xmllint` step in CI that at minimum checks for internally consistent processor id references (out of scope for this fix) |

### Risks of Track B

Deferred — evaluate only if the requirement lands.

### Cross-Issue Dependencies

- **#108**: this issue's fix removes one of the five naming-collision vectors (the NiFi processor). Full collision resolution comes from #110.
- **#110**: independent. #110 does not touch `Transaction_Data`; #109 does not touch `event_history.transaction`.

---

## Effort Estimate

| Track | Files | Effort |
|---|---|---|
| A — Disable / remove | 1 (`nifi/tazama.xml`) | 1–2 h including verification |
| B — Add proper handling | ~3–5 (new ETL, registry, NiFi, docs) | Deferred; est. 1–2 dev-days when needed |

---

## Acceptance Criteria (Verification Checklist)

### Track A

- [ ] `Transaction_Data` processor block removed from committed `biar/nifi/tazama.xml`.
- [ ] All downstream connections exclusive to this processor also removed (no dangling `sourceId` / `destinationId` references).
- [ ] Grep of `tazama.xml` for `transaction_data` / `Transaction_Data` returns zero hits.
- [ ] Fresh NiFi cluster imports the flow without errors.
- [ ] Live NiFi behaviour unchanged (was already absent).

### Track B (deferred)

- [ ] `TransactionDataETL` class exists with `record_key ∈ {endToEndId+tenantId, endToEndId+tenantId+createdAt}` (not `transactionId`).
- [ ] `_ETL_REGISTRY` entry: `"transaction_data": TransactionDataETL`.
- [ ] Naming reviewed against #108 (unambiguous, no re-introduction of `transactionData` collision).
- [ ] Consumer identified and scoped before enabling.

---

## Recommended Sequencing

1. Ship Track A independently or as a small NiFi cleanup PR — no dependency on #110 or #111.
2. Ship Track A **before or during** the #110 landing to reduce risk that a redeploy of the committed flow revives the processor mid-rollout.
3. Do not undertake Track B unless a real Lakehouse consumer emerges.
