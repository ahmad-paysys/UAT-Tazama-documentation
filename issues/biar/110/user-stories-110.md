# User Stories — Issue #110

Each story below is a discrete piece of work under Track B of #110. They land as one atomic PR family (see [github-issue-110.md](github-issue-110.md)).

---

## US-110-1 — Rewrite `TransactionsETL.bronze` to read Ozone JSON from `event_history.transaction`

**Body:**
`TransactionsETL.bronze()` currently reads `bronze/pacs008` and `bronze/pacs002` and produces a `bronze/transactions` (plural) Hudi table with a `transactionData` string column holding raw ISO 20022 payload. Rewrite it to read the JSON files landed at `{bucket}/transaction/*.json` by the live NiFi `Transaction` processor (which polls `event_history.transaction` on `10.10.80.19:15432`). Produce a singular-named `bronze/transaction` Hudi table with the TMS-flat scalar fields and preserve the JSONB `transaction` payload as a JSON string for downstream inspection.

**Scope:**
- Track: B
- Files: `automation-orchestrator/Table_ETLs/TransactionsETL.py`
- Change: rewrite `bronze()` and paths (`bronze_path`, `silver_path`, `gold_path` → singular).

**Acceptance Criteria:**
- [ ] `TransactionsETL.bronze_path` returns `f"{warehouse_root}/bronze/transaction"`.
- [ ] `bronze()` reads `self.spark.read.json(source_path)` where `source_path = s3a://${bucket}/transaction/…`.
- [ ] No calls to `self.spark.read.format("hudi").load(".../bronze/pacs008")` or `.../bronze/pacs002` in `bronze()`.
- [ ] Resulting bronze DF has columns `source, destination, transaction (json string), end_to_end_id, tx_type, tx_status, tx_amount, tx_ccy, tx_msg_id, cre_dt_tm, tenant_id, created_at_ts, source_file_path, transaction_id, record_hash`.

**Testing:**
- Unit test with a small `event_history.transaction`-shaped JSON list; assert bronze produces the expected DF schema.
- Integration test end-to-end (see US-110-9).

---

## US-110-2 — Rewrite `TransactionsETL.silver` with `account_holder` + `entity` join

**Body:**
Silver must derive `debtor_entity_id` and `creditor_entity_id` from `event_history.entity` via the `account_holder` join chain (Design Requirement 7). Both are FK-guaranteed to be present per the ODS schema. Party names are **not** in the ODS (see [issues/biar/110/issue-110.md §6](issue-110.md#6-event_historyentity-has-no-name-column)) and are therefore absent from Silver.

**Scope:**
- Track: B
- Files: `automation-orchestrator/Table_ETLs/TransactionsETL.py`
- Change: rewrite `silver()`.

**Acceptance Criteria:**
- [ ] Silver joins `bronze/account_holder` twice: once on `source == ah.destination` (debtor) and once on `destination == ah.destination` (creditor); tenantId matched in both.
- [ ] Output includes `debtor_entity_id`, `creditor_entity_id`.
- [ ] Silver produces `event_ts` from `cre_dt_tm` and `event_date`.
- [ ] Dedup Window on `transaction_id`, ordered by `created_at_ts desc` — keeps latest.

**Testing:**
- Unit test with a mock `account_holder` DF and a synthetic transaction; assert `debtor_entity_id` and `creditor_entity_id` match expected entity ids.

---

## US-110-3 — Rewrite `TransactionsETL.gold` for MVP scope

**Body:**
Project scalar TMS-normalised fields to Gold. Do **not** include pacs-only fields (`intr_bk_sttlm_amt`, `xchg_rate`, `instg_mmb_id`, `instd_mmb_id`, `charge_count`) — those are deferred to post-MVP Gold enrichment.

**Scope:**
- Track: B
- Files: `automation-orchestrator/Table_ETLs/TransactionsETL.py`
- Change: rewrite `gold()`.

**Acceptance Criteria:**
- [ ] Gold schema exactly: `transaction_id, end_to_end_id, tenant_id, tx_type, tx_status, tx_amount, tx_ccy, tx_msg_id, source_account_id, destination_account_id, debtor_entity_id, creditor_entity_id, event_ts, event_date, ingested_at_ts, source_file_path, record_hash`.
- [ ] No column named `intr_bk_sttlm_amt` / `xchg_rate` / `instg_mmb_id` / `instd_mmb_id` / `charge_count` / `debtor_name` / `creditor_name`.
- [ ] Raises if any non-scalar column remains.

**Testing:**
- Unit assertion on the column list.

---

## US-110-4 — Delete `_join_mode`, `_transactions_from_pacs`, simplify `run`

**Body:**
Remove the legacy join-mode code path and simplify the `run` orchestration.

**Scope:**
- Track: B
- Files: `automation-orchestrator/Table_ETLs/TransactionsETL.py`
- Change: delete methods `_transactions_from_pacs`, `_join_mode`; simplify `run` to accept only `source_path`.

**Acceptance Criteria:**
- [ ] `_join_mode` and `_transactions_from_pacs` symbols not present in file.
- [ ] `run(source_path: str)` signature; no `mode` parameter.

**Testing:**
- Unit: `run` calls `bronze` → `silver` → `gold` in order.

---

## US-110-5 — Delete `CombinedPacs.py` and register pacs ETLs directly

**Body:**
`CombinedPacsETL` exists solely to trigger the wrong `TransactionsETL(mode="from_pacs")`. It has no independent value once #110 lands.

**Scope:**
- Track: B
- Files: `automation-orchestrator/Table_ETLs/CombinedPacs.py` (delete); `automation-orchestrator/lakehouse_automation_pipeline.py`.
- Change: delete the file; remove the `CombinedPacsETL` import and dispatch; add `"pacs008": Pacs008ETL` and `"pacs002": Pacs002ETL` to `_ETL_REGISTRY`.

**Acceptance Criteria:**
- [ ] `Table_ETLs/CombinedPacs.py` does not exist.
- [ ] `_ETL_REGISTRY` includes keys `"pacs008"` and `"pacs002"`.
- [ ] `_route_etl` has no branch for `("pacs008", "pacs002")` calling `CombinedPacsETL`.

**Testing:**
- Unit: `_route_etl("pacs008", ...)` dispatches to `Pacs008ETL`.
- Integration: sending a pacs.008 message produces `gold/pacs008` without producing `gold/transactions`.

---

## US-110-6 — Remove skip guard and register `TransactionsETL`

**Body:**
Remove the `("transaction", "transactions")` skip guard in `_route_etl` and register `TransactionsETL` under both keys.

**Scope:**
- Track: B
- Files: `automation-orchestrator/lakehouse_automation_pipeline.py`.
- Change: delete `if table in ("transaction", "transactions"): return "Skipped: …"`; add `"transaction": TransactionsETL, "transactions": TransactionsETL` to `_ETL_REGISTRY`.

**Acceptance Criteria:**
- [ ] No `"Skipped: transactions are derived from PACS gold"` string in the file.
- [ ] `_route_etl("transaction", …)` dispatches to `TransactionsETL`.

**Testing:**
- Unit as above.

---

## US-110-7 — Update `views_orchestrator.py` readiness paths

**Body:**
Change `bronze/transactions` → `bronze/transaction` and `gold/transactions` → `gold/transaction` in `views_orchestrator.py`.

**Scope:**
- Track: B
- Files: `automation-orchestrator/Table_ETLs/views_orchestrator.py`.
- Change: three line edits at lines 71, 79, 108.

**Acceptance Criteria:**
- [ ] All `_hudi_ready` checks against transaction paths use the singular form.

**Testing:**
- Unit: given all upstream directories exist under singular names, all view builders trigger; given plural forms only, none trigger.

---

## US-110-8 — Refresh `data-lineage.md`

**Body:**
Update the transactions section of `data-lineage.md` to reflect the new source, new column set, and singular naming. Remove the note that `gold/transactions` doesn't carry account IDs.

**Scope:**
- Track: B
- Files: `biar/data-lineage.md`.
- Change: rewrite transaction pipeline section.

**Acceptance Criteria:**
- [ ] No reference to `TransactionsETL(mode="from_pacs")`.
- [ ] No claim that `gold/transaction*` "does not carry debtor/creditor account IDs".
- [ ] New section describes: NiFi Transaction processor → Ozone → `bronze/transaction` → `silver/transaction` (entity join) → `gold/transaction` (scalar TMS fields).

**Testing:**
- Doc review.

---

## US-110-9 — End-to-end UAT

**Body:**
Deploy the rewrite to UAT and validate the end-to-end path.

**Scope:**
- Track: B
- Files: n/a (deployment)
- Change: deploy to UAT.

**Acceptance Criteria:**
- [ ] Synthetic pacs.008 to TMS produces a `gold/transaction` row with the expected fields.
- [ ] Matching pacs.002 updates `tx_status`.
- [ ] No plural-named Hudi tables written after deploy.
- [ ] NiFi `Transaction` processor's Ozone output is consumed (not lingering).
- [ ] All view builders build without error (post-#111 changes applied).
- [ ] Anomaly Detection notebook (post-#111) runs or fails cleanly.

**Testing:**
- UAT playbook execution and sign-off.

---

## US-110-10 — Confirm `txSts` update semantics with TMS team

**Body:**
Before finalising the Hudi upsert key, confirm with Justus (or whoever owns the TMS side) whether `event_history.transaction.txSts` is updated in place on the pacs.008 row after pacs.002 receipt, or whether a new row is inserted for the pacs.002 message. This affects whether Hudi dedupe key is `(endToEndId, tenantId)` or `(endToEndId, txTp, tenantId)`.

**Scope:**
- Track: B (blocker)
- Files: none (external confirmation)

**Acceptance Criteria:**
- [ ] Decision documented in the PR body of Track B PR.
- [ ] Hudi record key set to match the decision.

**Testing:**
- Not applicable.
