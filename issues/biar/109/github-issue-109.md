# GitHub Issue Comment — Issue #109

## User Story

**Title:** Remove committed but undeployed `Transaction_Data` NiFi processor

**Body:**
The committed `biar/nifi/tazama.xml` includes a `Transaction_Data` processor that reads `tazama_cms.transaction_data` and forwards batches to the automation orchestrator. `DynamicETL._resolve_config()` only allowlists `raw_history`, `event_history`, and `enrichment`, so any batch would raise `RuntimeError` on the orchestrator side. Live NiFi at `http://10.10.80.19:8088` does not currently include the processor, so there is no active crash today — but the committed XML is a latent hazard that will revive the bug on any fresh redeploy. There is no Lakehouse consumer for this data. Remove the processor block and its exclusive downstream chain from `nifi/tazama.xml`.

**Scope:**
- Files: `biar/nifi/tazama.xml`
- Change: Remove the `Transaction_Data` processor block and any downstream `UpdateAttribute`/`ReplaceText`/`PutS3Object`/`InvokeHTTP` processors exclusive to its branch. Do **not** remove the shared `tazama_cms` DBCP pool (`b0b69311-…`) — it is used by `tasks` and `cms_usernames`.
- Migration required: No
- Frontend changes: No

**Acceptance Criteria:**
- [ ] `Transaction_Data` processor and its exclusive downstream chain removed from `biar/nifi/tazama.xml`.
- [ ] `grep -c "transaction_data" biar/nifi/tazama.xml` returns `0`.
- [ ] Fresh NiFi cluster imports the edited flow with no dangling connection references and starts cleanly.
- [ ] The `tazama_cms` DBCP pool remains; `tasks` and `cms_usernames` processors remain and function.

**Testing:**
- Import edited `tazama.xml` into a scratch NiFi container; verify no startup errors and no orphan processors.
- Diff against committed XML — the diff should touch only the identified `Transaction_Data` processor id and its exclusive downstream components.
- On live NiFi: verify the processor list is unchanged (was already absent).
- Deferred future-work note (Track B, only if `tazama_cms.transaction_data` ever becomes a Lakehouse requirement): use `endToEndId + tenantId` as the Hudi `record_key`, **not** the CMS `transactionId` autoincrement column. That autoincrement is not idempotent across CMS reinstalls and would compromise Hudi upsert semantics.
