# GitHub Issue & PR Templates — Issue #110

---

## GitHub Issue Body

### Problem

`gold/transactions` in the Data Lakehouse is built by `TransactionsETL` reading raw `bronze/pacs008` + `bronze/pacs002`. This is the wrong source. The correct source is `event_history.transaction` — the TMS-normalised ODS table with `source` / `destination` FK columns and a flat JSONB payload containing all business fields for every transaction (pacs.008 and pacs.002 alike in one row).

The NiFi `Transaction` (singular) processor is running live and correctly polling `event_history.transaction` at `10.10.80.19:8088`. Every batch it sends to the orchestrator is discarded by an explicit skip guard in `_route_etl` that returns `"Skipped: transactions are derived from PACS gold"`. `CombinedPacsETL` is the only path that populates `bronze/transactions`, doing so from the wrong upstream.

### Root Cause

`gold/transactions` was designed under the assumption that raw pacs messages are the primary transaction source. The TMS ODS layer (`event_history.transaction`) — which was built for exactly this purpose — was never wired in. Two mechanisms enforce the wrong state: the skip guard in the orchestrator, and the marker-file trigger in `CombinedPacsETL`. All downstream symptoms (missing `source`/`destination`, half-null Amt/Ccy/TxSts fields, party names sourced from ISO paths that don't exist on pacs.002 rows, naming collisions with the CMS `transaction_data` table) trace back to this single wrong-source decision.

### Impact

- `gold/transactions` lacks `source` / `destination` account IDs (present in the ODS as FK columns, absent from the current pipeline).
- Every pacs.002 row has null `Amt` / `Ccy`; every pacs.008 row has null `TxSts`.
- `Dbtr.Nm` / `Cdtr.Nm` are sourced from transient pacs.008 message fields — null on all pacs.002 rows.
- The live NiFi `Transaction` processor's output is silently discarded on every poll cycle; Ozone accumulates JSON files nobody reads.
- Naming collision with `tazama_cms.transaction_data` (see #108) — five distinct meanings for the same identifier.
- Blocks the Anomaly Detection notebook, CMS Transaction Detail panel, CMS Alert transactional data, and 10 other consumers from having a clean transaction data model to read.

### Proposed Fix

Two tracks — but Track A is documentation-only; the real fix is Track B and it must land atomically.

#### Track A — Documentation intent (optional, ~0.5 dev-day)
- Add a warning comment above `TransactionsETL.bronze()` naming the wrong source and linking to this issue.
- No behaviour change. No file impact beyond the comment. No migration.

#### Track B — Rewrite (~1 dev-week + review + UAT)
- Rewrite `TransactionsETL` to read from `{bucket}/transaction/` (NiFi-landed Ozone JSON).
- Delete `CombinedPacsETL`; add direct `Pacs008ETL` / `Pacs002ETL` entries to `_ETL_REGISTRY`.
- Remove skip guard; add `"transaction": TransactionsETL` / `"transactions": TransactionsETL` to `_ETL_REGISTRY`.
- Rename Hudi paths from `*/transactions` to `*/transaction` (singular, matching the ODS name and NiFi routing attribute); drop and rebuild.
- Populate `source`, `destination`, `debtor_entity_id`, `creditor_entity_id` and TMS-flat scalar fields at Gold.
- Do **not** include pacs-specific fields (`intr_bk_sttlm_amt`, `xchg_rate`, `instg_mmb_id`, `instd_mmb_id`, `charge_count`) in MVP `gold/transaction` — defer to post-MVP Gold enrichment.
- Update `views_orchestrator.py` readiness paths.
- Fold #108 view-builder candidate-list hardening into the same PR.
- Refresh `data-lineage.md`.
- Consumer sweep tracked in #111.

Migration: no PostgreSQL schema changes. Hudi tables are drop-and-rebuild; NiFi replays via its own `credttm` watermark.

### Acceptance Criteria

- [ ] `TransactionsETL` reads exclusively from `{bucket}/transaction/`; no pacs bronze reads.
- [ ] `_join_mode` and `mode` parameter removed.
- [ ] `CombinedPacs.py` deleted.
- [ ] Skip guard removed from `_route_etl`.
- [ ] `_ETL_REGISTRY` has `"pacs008"`, `"pacs002"`, `"transaction"`, `"transactions"` mapped to their respective ETL classes.
- [ ] `bronze/transaction`, `silver/transaction`, `gold/transaction` (singular) exist; plural counterparts dropped.
- [ ] `gold/transaction` schema: `transaction_id`, `end_to_end_id`, `tenant_id`, `tx_type`, `tx_status`, `tx_amount`, `tx_ccy`, `tx_msg_id`, `source_account_id`, `destination_account_id`, `debtor_entity_id`, `creditor_entity_id`, `event_ts`, `event_date`, `ingested_at_ts`, `source_file_path`, `record_hash`.
- [ ] `gold/transaction` does **not** contain any pacs-only column at MVP.
- [ ] View builders' candidate lists tightened (see #108).
- [ ] `views_orchestrator.py` uses singular paths.
- [ ] `data-lineage.md` refreshed.
- [ ] All 14 #111 consumers updated.
- [ ] `txSts` update semantics confirmed with TMS team before finalising upsert key.

---

## PR Descriptions

### Track A PR

**PR Title:** `docs(dlh): mark TransactionsETL wrong-source pending #110 rewrite`

**PR Body:**

- **Summary**
  - Adds a `# TODO(#110)` comment above `TransactionsETL.bronze()` documenting that the current pacs-bronze source is wrong.
  - No behaviour change; purely a signpost for future readers until the full rewrite lands.
- **Test Plan**
  - [ ] `git grep TODO(#110)` returns the expected two entries.
- **Related Issue:** #110

### Track B PR

**PR Title:** `feat(dlh): source gold/transaction from event_history.transaction (fixes #110)`

**PR Body:**

- **Summary**
  - Rewrite `TransactionsETL` to read from `{bucket}/transaction/` (NiFi-landed Ozone JSON from `event_history.transaction`).
  - Delete `CombinedPacsETL`; register `Pacs008ETL` / `Pacs002ETL` / `TransactionsETL` directly in `_ETL_REGISTRY`.
  - Remove the `"transaction"/"transactions"` skip guard in `_route_etl`.
  - Rename Hudi paths from `*/transactions` to `*/transaction`; drop old tables; rebuild singular-named tables from NiFi watermark.
  - Add TMS-flat schema (`source_account_id`, `destination_account_id`, `debtor_entity_id`, `creditor_entity_id`, and the flat business fields).
  - Fold in #108 view-builder candidate-list hardening.
  - Update `views_orchestrator.py`.
- **Test Plan**
  - [ ] Unit: `TransactionsETL.bronze` reads Ozone JSON without pacs bronze reads.
  - [ ] Unit: `_route_etl` does not skip `transaction`/`transactions`.
  - [ ] Unit: `_route_etl` does not import `CombinedPacsETL`.
  - [ ] Unit: gold projects the expected column set; no pacs-only columns.
  - [ ] Integration: synthetic pacs.008 → `gold/transaction` row appears with `source_account_id` populated and `event_ts` correct.
  - [ ] Integration: matching pacs.002 → `tx_status` on the row reflects the ACCC.
  - [ ] Integration: `views_orchestrator` runs `TransactionDetailViewETL` / `TransactionHistoryViewETL` / `NetworkNavigatorViewETL` / `ConditionsTimelineViewETL` after `bronze/transaction` populates.
  - [ ] Manual: CMS Transaction Detail panel (post-#111 view rewrite) — `debtor_name`/`creditor_name` present on pacs.008-based rows via short-term view join.
  - [ ] Manual: Anomaly Detection notebook — either runs (view-layer pacs join) or fails cleanly (documented MVP gap).
- **Related Issue:** #110
- **Migration**
  - Deploy new code first, then drop plural Hudi tables:
    - `bronze/transactions`, `silver/transactions`, `gold/transactions`.
  - Delete marker files:
    - `.pipeline_state/pacs008_gold_done`
    - `.pipeline_state/pacs002_gold_done`
    - `.pipeline_state/transactions_trigger.lock`
  - NiFi state: no reset required for normal operation. If a full backfill is desired, clear the `Transaction` processor's state via NiFi UI (Server B, `http://10.10.80.19:8088`) before deploy.
- **Sequencing Note**
  - Must land AFTER `#109` NiFi cleanup (small, independent).
  - Must land ATOMICALLY with view-builder hardening from #108 Track A and with the consumer sweep in #111.
  - Must land BEFORE any long-lived consumer work on `gold/transaction` post-MVP enrichment (Design Requirement 6).
