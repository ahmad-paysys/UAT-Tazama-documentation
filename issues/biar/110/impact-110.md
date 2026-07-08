# Impact Study — Issue #110: event_history.transaction data is not the source for gold/transactions

**Issue:** [#110](https://github.com/tazama-lf/biar/issues/110)
**Study Date:** 2026-07-08
**Author:** Ahmad Khalid
**Repository:** tazama-lf/biar

---

## Summary

`gold/transactions` is currently sourced from raw pacs.008/002 bronze via `TransactionsETL`, orchestrated by `CombinedPacsETL` on marker files. The correct source — `event_history.transaction` — is being polled live by NiFi but discarded by an explicit skip guard in the automation orchestrator. The rewrite reroutes `TransactionsETL` to consume the NiFi-landed Ozone files at `${bucket}/transaction/`, produces singular-named Hudi tables with TMS-normalised flat fields plus `source`/`destination` FKs, derives party identifiers via the `entity` join chain, and defers pacs-specific enrichment (`xchg_rate`, `intr_bk_sttlm_amt`, `instg_mmb_id`, `instd_mmb_id`, `charge_count`) to a post-MVP Gold-layer join. Consumers requiring the deferred fields add a short-term view-layer `LEFT JOIN gold/pacs008` (see #111). There is no meaningful Track A; the fix is inherently a coordinated multi-file rewrite. Consumer sweep is tracked in [#111](../111/); NiFi processor cleanup for `transaction_data` is tracked in [#109](../109/).

---

## Confirmed Root Cause

- `TransactionsETL.bronze` at `TransactionsETL.py:108-119` reads `bronze/pacs008` and `bronze/pacs002` unconditionally.
- Skip guard at `lakehouse_automation_pipeline.py:158-163` discards every `transaction`/`transactions` batch that NiFi sends.
- `CombinedPacsETL.run()` at `CombinedPacs.py:113-172` is the only invoker of `TransactionsETL`; deleting it and removing the skip guard together is the only sequence that works.
- NiFi `Transaction` (singular) processor is **live**: verified via `http://10.10.80.19:8088/nifi-api/process-groups/root/processors` — id `711a2219-…`, table `transaction`, watermark `credttm`, pool `event_history`, state RUNNING. Downstream chain writes Ozone JSON files that the orchestrator then discards.
- **Live payload template** carried by the `Transaction` FlowFile (from `ReplaceText 81dd19c9-…`): `{raw_path, bucket, table, object_key, execute_notebook}` — no `db_name` — matches Template A. So the skip-guard path is hit *before* `db_name` is used; no crash today, just silent discard.
- `event_history.transaction` DDL (Full-Stack-Docker-Tazama `00-CREATE.sql:162-197`): `(source, destination, transaction jsonb, endToEndId, amt, ccy, msgId, creDtTm, txTp, txSts, tenantId)` with FKs to `account(id, tenantId)` and primary key `(endToEndId, txTp, tenantId)`. All claims in the issue about column existence verified.
- `entity` DDL at `00-CREATE.sql:83-88`: `(id, tenantId, creDtTm)` — no name column. `Dbtr.Nm`/`Cdtr.Nm` are transient pacs.008 fields, not persisted.
- `account_holder` DDL at `00-CREATE.sql:90-98`: `(source, destination, tenantId, creDtTm)` where `source` FK = `entity.id`, `destination` FK = `account.id`. The join chain in Design Requirement 7 is exact.

---

## Track A — No minimal fix

There is no partial change that provides value without breaking something else. Documenting the intent in `data-lineage.md` is the only isolatable action; it does nothing to reduce the runtime symptom.

---

## Track B — Full rewrite

### What Changes

Rewrite `TransactionsETL`, delete `CombinedPacsETL`, remove skip guard, add registry entries, rename Hudi paths to singular, update `views_orchestrator.py`, tighten view-builder candidate lists (see #108), refresh `data-lineage.md`, and coordinate the consumer sweep (see #111).

### Schema Impact

| Change | Migration Notes | Risk |
|---|---|---|
| Drop plural Hudi tables `bronze/transactions`, `silver/transactions`, `gold/transactions` | Runtime state — delete directories on Ozone | Reversible: `TransactionsETL` recreates them if reverted; but consumers will have new names |
| Create singular Hudi tables `bronze/transaction`, `silver/transaction`, `gold/transaction` | Built by rewritten ETL on first run | Low — new tables |
| No PostgreSQL schema changes | — | None |
| Delete NiFi marker files `.pipeline_state/pacs008_gold_done`, `pacs002_gold_done`, `transactions_trigger.lock` | Simple `rm` on the orchestrator container's warehouse volume | Trivial |

### Backend Code Impact

#### Core rewrites

| File | Change |
|---|---|
| `automation-orchestrator/Table_ETLs/TransactionsETL.py` | Full rewrite — read Ozone `{bucket}/transaction/*.json` in `bronze`; join to `account_holder`/`entity` in `silver`; project scalar TMS fields in `gold`. Remove `_join_mode`. Rename paths to singular. |
| `automation-orchestrator/lakehouse_automation_pipeline.py` | Remove skip guard (lines 158–163); add registry entries; remove `CombinedPacsETL` import and dispatch; add direct `Pacs008ETL`/`Pacs002ETL` entries. |
| `automation-orchestrator/Table_ETLs/CombinedPacs.py` | **Delete.** |
| `automation-orchestrator/Table_ETLs/views_orchestrator.py` | Change readiness paths at lines 71, 79, 108 from plural to singular. |

#### New files

None strictly required.

#### Enum/reference sweep

| File | Change |
|---|---|
| `data-lineage.md` | Update transaction pipeline section |
| `datalakehouse-api/lakehouse_query_api.py` | Path constants and registry keys — coordinated in [#111](../111/) |
| JupyterHub notebooks | Path renames — coordinated in [#111](../111/) |
| View builders (3 files) | Candidate list tightening — coordinated in [#108](../108/) Track A |
| CMS backend services (2 files) | View column adjustments — coordinated in [#111](../111/) |

### Frontend Code Impact

None from #110 directly. CMS frontend changes (if any) are downstream of the CMS-backend view queries, which are in #111. Test fixtures scope: CMS end-to-end tests that assert `debtor_name`/`creditor_name` populate for pacs.008 payment scenarios need to be updated to reflect the short-term join path — but the fixtures themselves largely stay the same shape.

---

## Side Effects and Risks

### Risks of Track A Alone

Not applicable (no Track A).

### Risks of Track B

| Risk | Likelihood | Mitigation |
|---|---|---|
| View builders' `"transaction"` candidate fallback silently picks up the new bronze `transaction` column and parses TMS-normalised JSON as ISO 20022 → all-null output | High if the candidate lists are not tightened | Fold #108 Track A into the same PR. Add unit test that a bronze df with `transaction` column but no `transactionData` column produces a valid `transaction_data` alias if and only if the explicit expected-column check is passed |
| Marker files (`pacs008_gold_done` etc.) linger and confuse future readers | Low | Deletion step in migration playbook |
| Long-tail consumers not yet inventoried (search `gold/transactions`, `bronze/transactions`, `silver/transactions` in every repo) | Medium | Grep before merge; #111 lists 14 known consumers |
| `debtor_name` / `creditor_name` columns missing at Gold layer break CMS Transaction Detail | High | View-layer short-term pacs join (see #111 Track A row for Transaction Detail); or wait for post-MVP Gold enrichment |
| Anomaly Detection notebook (`instg_mmb_id`/`instd_mmb_id` join keys) breaks | High (confirmed by live notebook inspection) | Short-term pacs join in a view builder, or defer notebook to post-MVP |
| `txSts` update semantics on `event_history.transaction` after pacs.002 receipt — is the row updated in place, or a new row inserted? | Unknown from code | **Must confirm with Justus / TMS team before finalising the Hudi upsert key strategy.** Primary key on the TMS side is `(endToEndId, txTp, tenantId)` — the same E2E can have both a pacs.008 row (txTp='pacs.008.001.10') and a pacs.002 row (txTp='pacs.002.001.12'). If txSts is stored on the pacs.008 row (updated after 002 receipt), Hudi upsert must dedupe on `(endToEndId, tenantId)`. If it's stored on the pacs.002 row, dedupe on `(endToEndId, txTp, tenantId)`. |
| ODS-side row updates arrive at NiFi with new `credttm` (or not) — determines whether Hudi upserts see fresh rows | Unknown from code | Confirm with TMS team; run a short test in UAT |

### Cross-Issue Dependencies

- **#108**: The view-builder candidate-list hardening AC is folded into the #110 PR. #108 has no independent code fix.
- **#109**: Independent. Recommended to ship first (small NiFi cleanup) so the committed flow does not revive the `Transaction_Data` processor during #110 rollout.
- **#111**: Depends on #110. The 14 consumers must be updated after #110 lands and `gold/transaction` is available; some need view-layer pacs joins.

---

## Effort Estimate

| Track | Files | Effort |
|---|---|---|
| A (documentation only) | 2 | 0.5 dev-day |
| B (rewrite + support) | ~9 in biar + docs + coordinated #111 sweep | 4–6 dev-days for the biar side; #111 has its own line-item |

Total for the #110 + #111 delivery bundle: ~2 dev-weeks + review + UAT.

---

## Acceptance Criteria (Verification Checklist)

### Track A

Not applicable.

### Track B

- [ ] `TransactionsETL` reads exclusively from `{bucket}/transaction/`; no pacs bronze reads in `bronze()`.
- [ ] `_join_mode` and `mode` parameter removed from `TransactionsETL.run`.
- [ ] `CombinedPacs.py` deleted; `_ETL_REGISTRY` has direct `Pacs008ETL` / `Pacs002ETL` entries.
- [ ] Skip guard removed; `_ETL_REGISTRY` has `"transaction": TransactionsETL` and `"transactions": TransactionsETL`.
- [ ] `gold/transaction` (singular) built; plural `gold/transactions` dropped.
- [ ] `gold/transaction` schema includes `source`, `destination`, `debtor_entity_id`, `creditor_entity_id` and TMS-flat fields; excludes pacs-only fields.
- [ ] View builders' candidate lists tightened (#108 Track A).
- [ ] `views_orchestrator.py` uses singular paths.
- [ ] `data-lineage.md` refreshed.
- [ ] #111 consumers updated in a coordinated PR family.
- [ ] `txSts` update semantics confirmed with TMS team before final upsert-key selection.
- [ ] E2E test: a pacs.008 message is processed, `gold/transaction` row appears with correct `source`/`destination`; then a pacs.002 message with matching E2E is processed and `txSts` reflects it.

---

## Recommended Sequencing

1. Ship [#109](../109/) (independent, small).
2. Confirm `txSts` update semantics with TMS team; decide Hudi upsert key.
3. Ship #110 + #108 Track A + #111 (as coordinated PR family or one PR).
4. Post-MVP: ship Design Requirement 6 (Gold layer pacs enrichment) as a follow-up. Retire per-consumer view-layer joins as they become redundant.
