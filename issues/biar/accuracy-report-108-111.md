# Accuracy Report — biar Issues #108, #109, #110, #111

**Repository:** tazama-lf/biar (dev branch, HEAD `dc32fd5`)
**Cross-repo verification:** tazama-lf/frms-coe-lib (dev), tazama-lf/case-management-system (dev), tazama-lf/Full-Stack-Docker-Tazama (dev)
**Report Date:** 2026-07-07
**Reviewer:** Ahmad Khalid
**Purpose:** Critical, code-anchored evaluation of a set of AI-assisted issue reports (#108–#111). The reporter has explicitly asked that these be fact-checked before any implementation work begins. Regular issue-processing workflow is deliberately suspended.

---

## 0. Method

For every specific, checkable claim in issues #108–#111, I:

1. Located the referenced file on the `dev` branch of the relevant repo.
2. Compared the quoted code / line numbers / paths / behavior to what is actually present.
3. Marked each claim as **Confirmed**, **Partially confirmed** (matches on substance, drifts on detail), **Unverifiable without runtime access** (claim is about runtime behavior that cannot be proven from static code alone), or **Incorrect / Not found**.

Nothing is marked confirmed on the basis of the issue's own quotation; every quote was re-read from the tree.

Where a discrepancy is minor (line-number drift, cosmetic wording), it is noted but does **not** invalidate the claim. Where a discrepancy is substantive (file does not exist, code disagrees with quoted behavior, off-by-one on which is broken), it is called out explicitly.

---

## 1. Overall Verdict

**The four issues are, in aggregate, substantively accurate.** The core factual claims — the `transactionData` misnaming, the orchestrator skip guard, the CombinedPacs trigger, the DynamicETL config gap, the CMS entity table having no name column, and the consumer inventory — all check out against the code.

There are, however, three categories of issue worth flagging before implementation:

1. **A file referenced in #111 does not exist** (`ConditionsTimelineView.py`). See §5.6.
2. **One consumer is missing from #111's inventory** (`views_orchestrator.py`). See §5.7.
3. **Several claims are about runtime behavior of NiFi that cannot be proven from static XML alone** — most importantly the exact `db_name` attribute value that arrives at the automation orchestrator for the `transaction_data` batch. The failure mode described in #109 is *consistent* with the code, but the specific claim that `db_name=tazama_cms` is what reaches DynamicETL requires either a runtime trace or a walk of the NiFi FlowFile attribute wiring that I have not fully completed. See §3.

None of these are blockers. #108, #110 and (once its two nits are addressed) #111 can be treated as trustworthy inputs to implementation planning. #109's remediation (disable the NiFi processor) is safe regardless of whether the exact crash mechanism is exactly as described, because there is genuinely no downstream consumer for that data.

---

## 2. Issue #108 — `transaction_data` Naming Collision

**Title (as filed):** DLH BUG: transaction_data Naming Collision
**Substance:** Documentation / clarity issue rooted in a wrong source choice; no direct code fix proposed — resolves as a natural consequence of #110.

### 2.1 Claim-by-claim verification

| # | Claim | Verdict | Evidence |
|---|---|---|---|
| 108-1 | `bronze/transactions.transactionData` is a column that holds raw pacs.008/002 ISO 20022 payload as a string, written by `TransactionsETL._transactions_from_pacs()`. | **Confirmed.** | [TransactionsETL.py:72-102](repos/biar/automation-orchestrator/Table_ETLs/TransactionsETL.py#L72-L102) — the method reads pacs bronze, coalesces `document_json`/`document`, aliases it as `transactionData` (line 99), and it is stored as a string on the bronze/transactions table via `.withColumn("transactionData", F.col("transactionData").cast("string"))` at [line 128](repos/biar/automation-orchestrator/Table_ETLs/TransactionsETL.py#L128). |
| 108-2 | Both view builders detect the JSON column via a candidate list `["transactionData", "transaction_data", "transaction", "payload", …]` and alias whatever they find as `transaction_data`. | **Confirmed.** | [network_navigator_view.py:68-81](repos/biar/automation-orchestrator/Table_ETLs/network_navigator_view.py#L68-L81) and [transaction_detail_view.py:53-69](repos/biar/automation-orchestrator/Table_ETLs/transaction_detail_view.py#L53-L69). The list in `network_navigator_view.py` is `["transactionData", "transaction_data", "transaction", "payload", "raw_payload"]` and in `transaction_detail_view.py` is `["transactionData", "transaction_data", "transaction", "payload", "raw_payload", "raw_json", "transaction_json"]`. Both alias the resolved column as `transaction_data`. |
| 108-3 | The NiFi `Transaction_Data` processor points at `tazama_cms.transaction_data` via the `b0b69311-…` pool. | **Confirmed.** | [nifi/tazama.xml:5905 label](repos/biar/nifi/tazama.xml#L5905), [tazama.xml:26173 table name = `transaction_data`](repos/biar/nifi/tazama.xml#L26173), [tazama.xml:26165 pool = `b0b69311-8545-36e2-…`](repos/biar/nifi/tazama.xml#L26165). The pool is defined at [tazama.xml:4658-4784](repos/biar/nifi/tazama.xml#L4658-L4784) with `<name>tazama_cms</name>` and URL `jdbc:postgresql://10.10.80.18:15433/tazama_cms`. |
| 108-4 | `tazama_cms.transaction_data` has a column named `transactionData` (nested naming quirk). | **Confirmed.** | [case-management-system/backend/prisma/schema.prisma@origin/dev](repos/case-management-system/backend/prisma/schema.prisma) — model `TransactionData` maps to table `transaction_data` (`@@map`) and has columns `transactionId Int @id @default(autoincrement())`, `tenantId String`, `endToEndId String`, **`transactionData Json`**, `createdAt DateTime`. The nested naming (table `transaction_data` with column `transactionData`) is exactly as claimed. Primary key is a synthetic autoincrement int, not a pacs-derived business key — see §3 note. |
| 108-5 | Candidate list contains `"transaction"` which, once #110 renames `bronze/transaction` and if that table has a column called `transaction` (the JSONB blob column from `event_history.transaction`), the view builders will silently pick it up and parse it as ISO 20022 → returning nulls for all pacs paths. | **Confirmed as a latent risk.** | The candidate list literally contains `"transaction"`; `event_history.transaction`'s payload column is literally called `transaction` (see [eventHistoryBuilder.ts:36](repos/frms-coe-lib/src/builders/eventHistoryBuilder.ts#L36) — `INSERT INTO transaction (source, destination, transaction)`). If a new `bronze/transaction` retains that column name from the source, the alias-then-pacs-parse would produce nulls. This is a real trap for the #110 rewrite. |
| 108-6 | The name `transactionData` misrepresents the column's origin, and a developer trying to satisfy a "surface CMS transaction data" requirement will follow the name to the wrong place. | **Confirmed as a documentation defect.** | Nothing in [TransactionsETL.py](repos/biar/automation-orchestrator/Table_ETLs/TransactionsETL.py) or the view builders comments the actual data source. The name genuinely does imply the CMS `transaction_data` table. |
| 108-7 | Once #110 lands, `transactionData` disappears (not renamed, replaced) and all five collisions collapse. | **Consistent with the plan in #110.** | Not a code claim; a design assertion. Contingent on #110 being executed as described. |

### 2.2 Findings for #108

- Every code-checkable claim in #108 is accurate.
- The recommendation ("resolves as a consequence of #110") is sound; a standalone rename would be throwaway work.
- The one claim I could not verify from the code (108-4, the `transactionData` column inside `tazama_cms.transaction_data`) should be double-checked against the live DB before this is treated as canonical.
- The candidate-list trap (108-5) is the single most actionable finding: even if #110 is done exactly as the issue proposes, if the view builders' candidate list is not tightened during the #110 view rewrite, the pipeline can silently produce all-null rows. This warrants an explicit acceptance criterion on the #110/#111 PR.

---

## 3. Issue #109 — `tazama_cms.transaction_data` Crashes the Orchestrator

**Title (as filed):** DLH BUG: tazama_cms.transaction_data Pipeline Crashes Automation Orchestrator on Every Poll
**Substance:** Runtime failure claim; remediation is to disable the NiFi processor.

### 3.1 Claim-by-claim verification

| # | Claim | Verdict | Evidence |
|---|---|---|---|
| 109-1 | The NiFi `Transaction_Data` processor reads `tazama_cms.transaction_data` with watermark `"createdAt"` (quoted), fetch size 500, via pool `b0b69311-…`. | **Confirmed.** | Same lines as 108-3. Watermark at [tazama.xml:26190](repos/biar/nifi/tazama.xml#L26190) is `"createdAt"` (with quotes preserving Prisma camelCase). Fetch Size = 500 at [tazama.xml:26202](repos/biar/nifi/tazama.xml#L26202). |
| 109-2 | `transaction_data` is not in `_ETL_REGISTRY` in `lakehouse_automation_pipeline.py`, so it falls through to `DynamicETL`. | **Confirmed.** | The registry is at [lakehouse_automation_pipeline.py:51-68](repos/biar/automation-orchestrator/lakehouse_automation_pipeline.py#L51-L68). `transaction_data` is absent. Dispatch logic at [lines 150-176](repos/biar/automation-orchestrator/lakehouse_automation_pipeline.py#L150-L176) — anything not in the registry and not in `("pacs008","pacs002","transaction","transactions")` reaches the `DynamicETL` fallback at [line 169](repos/biar/automation-orchestrator/lakehouse_automation_pipeline.py#L169). |
| 109-3 | `DynamicETL._resolve_config()` only knows `raw_history`, `event_history`, `enrichment` and raises `RuntimeError` with the shown message for anything else. | **Confirmed with a cosmetic caveat.** | [DynamicETL.py:54-80](repos/biar/automation-orchestrator/Table_ETLs/DynamicETL.py#L54-L80). The configs dict and RuntimeError are exactly as described. The error message is line-wrapped differently from the issue's quote (three-line f-string vs. the issue's two-line reformat), but the substance is identical: `f"[DynamicETL] Unidentified db: '{db_name}'. Cannot process. Supported db_names: {list(configs.keys())}"`. |
| 109-4 | On every poll cycle that returns rows, this raises `RuntimeError` and crashes the automation orchestrator endpoint. | **Contingent on 109-5.** | The RuntimeError raises for any `db_name` not in the three configs. Whether it actually happens on every non-empty poll cycle depends on **what `db_name` the FlowFile carries when it reaches the orchestrator**. See 109-5. |
| 109-5 | The `db_name` attribute reaching the orchestrator is `tazama_cms` (from `current_database()` in a NiFi SQL discovery query at ~line 14118). | **Partially verified — requires runtime confirmation.** | There is indeed a NiFi metadata query at [tazama.xml:14118-14131](repos/biar/nifi/tazama.xml#L14118-L14131) that uses `current_database() AS db_name`, but reading that block: it runs against the `enrichment` DB pool (`ce84d264-…`, per [line 14110](repos/biar/nifi/tazama.xml#L14110)), not `tazama_cms`. There is a **separate** metadata query at [tazama.xml:20606](repos/biar/nifi/tazama.xml#L20606) that runs against pool `68818ebb-…` = `enrichment` again. The exact processor that supplies `db_name = tazama_cms` to the `transaction_data` FlowFile was not conclusively identified in this review. The claim is plausible (the whole NiFi pattern is metadata-discovery-per-DB → attribute enrichment → common downstream) and consistent with the failure mode, but the specific line number `~14118` given in the issue points at the enrichment DB's metadata query, not tazama_cms's. **Recommendation:** either walk the FlowFile wiring in the NiFi UI to confirm the `db_name` value routed for `transaction_data`, or capture one crash log from the orchestrator to prove it. This does not change the remediation (disable the processor) but it changes the accuracy of the description. |
| 109-6 | Empty poll cycles pass through silently and produce no Hudi output. | **Consistent with the code.** | If NiFi returns zero rows, the QueryDatabaseTableRecord processor produces no FlowFile; no HTTP call to the orchestrator happens. Nothing in the orchestrator would run. Not directly a code-verifiable statement, but not contradicted. |
| 109-7 | No ETL, view builder, notebook, or dashboard in the Lakehouse references `tazama_cms.transaction_data` as a data source. | **Confirmed.** | Full-tree grep in `repos/biar` for `tazama_cms.transaction_data` returns zero non-NiFi hits. All references to the string `transaction_data` in the Python code refer to the local view-builder alias (§2, item 108-2), never to a CMS ODS table. |
| 109-8 | The view builders that reference `transaction_data` in their code use it as an alias for the pacs payload in `bronze/transactions` and are entirely unrelated to the CMS table. | **Confirmed.** | Same evidence as 108-2. |
| 109-9 | The three future-work prerequisites (registry entry / `record_key=transactionId`, `precombine=createdAt`; dedicated field mapping; unambiguous naming) apply *if* CMS ingestion becomes a real requirement. | **Not a code claim.** | Design assertion; consistent with how the DynamicETL profiles are structured. |

### 3.2 Findings for #109

- The failure mechanism (missing `tazama_cms` config in `DynamicETL._resolve_config`) is exactly as described. The crash *would* happen for any FlowFile arriving with `db_name=tazama_cms`.
- **Nit on the future-requirements section:** the issue proposes `record_key="transactionId"` for a future `TransactionDataETL`. Per the CMS Prisma schema ([schema.prisma model `TransactionData`](repos/case-management-system/backend/prisma/schema.prisma)), `transactionId` is `Int @id @default(autoincrement())` — a synthetic autoincrement primary key allocated on the CMS side. Using an autoincrement column as the Hudi `record_key` couples Lakehouse identity to the CMS row-insertion sequence and is not idempotent across CMS reinstalls / restores. If this table ever does get ingested, `endToEndId` + `tenantId` (or `endToEndId` + `tenantId` + `createdAt`) would be a more defensible key. Not urgent — flagging for whoever picks up the future work.
- The one soft spot is **whether the FlowFile actually arrives with `db_name=tazama_cms`**, which is a NiFi wiring question I could not close from XML alone (the issue's line reference points to the enrichment metadata query, not the tazama_cms one). It should be closed by either:
  - a NiFi UI walk-through of the `transaction_data` flow's `UpdateAttribute` chain, or
  - a snippet of the crash log from a run where `transaction_data` rows arrived.
- The remediation (disable/remove the NiFi `Transaction_Data` processor) is safe regardless, because 109-7 (no consumer) is unambiguously true.
- The future-requirements section is sound advice, not code claims.

---

## 4. Issue #110 — `event_history.transaction` Is Not the Source for `gold/transactions`

**Title (as filed):** DLH BUG: event_history.transaction data is not the source for gold/transactions in DLH
**Substance:** This is the load-bearing issue. Everything in #108, #109 and #111 either depends on this or resolves through it.

### 4.1 Claim-by-claim verification

| # | Claim | Verdict | Evidence |
|---|---|---|---|
| 110-1 | `gold/transactions` is currently built by `TransactionsETL` reading `bronze/pacs008` + `bronze/pacs002`. | **Confirmed.** | [TransactionsETL.py:108-119](repos/biar/automation-orchestrator/Table_ETLs/TransactionsETL.py#L108-L119) — `bronze()` loads `bronze/pacs008` and `bronze/pacs002` and (in default `from_pacs` mode) calls `_transactions_from_pacs` on each and unions. `gold_path` at [line 45](repos/biar/automation-orchestrator/Table_ETLs/TransactionsETL.py#L45) is `warehouse_root/gold/transactions`. |
| 110-2 | A NiFi `Transaction` processor (singular, `event_history.transaction`) exists and correctly extracts to Ozone. | **Confirmed.** | [tazama.xml:5967 `<label>Transaction</label>`](repos/biar/nifi/tazama.xml#L5967). The processor at [tazama.xml:22744-22762](repos/biar/nifi/tazama.xml#L22744-L22762) sets `Table Name = transaction`, `Maximum-value Columns = credttm`, and uses pool `ce84d264-…` = `event_history` ([tazama.xml:5169-5174](repos/biar/nifi/tazama.xml#L5169-L5174), URL `jdbc:postgresql://10.10.80.18:15432/event_history`). |
| 110-3 | The automation orchestrator has a skip guard that explicitly discards `event_history.transaction` batches with message `"Skipped: transactions are derived from PACS gold"`. | **Confirmed.** | [lakehouse_automation_pipeline.py:156-161](repos/biar/automation-orchestrator/lakehouse_automation_pipeline.py#L156-L161) — `if table in ("transaction", "transactions"): return "Skipped: transactions are derived from PACS gold"`. Message identical to the issue's quote (modulo one wrapping print statement above the return). |
| 110-4 | `CombinedPacsETL` uses marker files (`.pipeline_state/pacs008_gold_done`, `.pipeline_state/pacs002_gold_done`) to detect completion and fires `TransactionsETL(mode="from_pacs")` from the wrong source. | **Confirmed.** | [CombinedPacs.py:120-166](repos/biar/automation-orchestrator/Table_ETLs/CombinedPacs.py#L120-L166). Marker file names match: `pacs008_gold_done` at line 121, `pacs002_gold_done` at line 122; TransactionsETL trigger with `mode="from_pacs"` at [line 162-163](repos/biar/automation-orchestrator/Table_ETLs/CombinedPacs.py#L162-L163). |
| 110-5 | `Pacs008ETL` and `Pacs002ETL` have zero dependency on each other; the "combining" adds no value to their pipelines. | **Confirmed by inspection.** | [CombinedPacs.py:113-118](repos/biar/automation-orchestrator/Table_ETLs/CombinedPacs.py#L113-L118) — the pacs pipeline for the *current* table (`self.table`) runs on its own; the other pacs table is not touched. The joint invocation is only for the transactions trigger downstream. |
| 110-6 | `event_history.transaction` has structural columns `source` and `destination` FK to `event_history.account(id, tenantId)` — TMS debtor/creditor account IDs. | **Confirmed against authoritative DDL.** | [Full-Stack-Docker-Tazama/core/postgres/migration/base/00-CREATE.sql:162-180](repos/Full-Stack-Docker-Tazama/core/postgres/migration/base/00-CREATE.sql#L162-L180): `create table transaction (source varchar not null, destination varchar not null, transaction jsonb not null, ..., foreign key (source, tenantId) references account (id, tenantId), foreign key (destination, tenantId) references account (id, tenantId), primary key (endToEndId, txTp, tenantId))`. |
| 110-7 | The `transaction` JSONB column carries a TMS-normalised flat record including `EndToEndId`, `Amt`, `Ccy`, `MsgId`, `CreDtTm`, `TxTp`, `TxSts`, `TenantId`. | **Confirmed against authoritative DDL.** | [00-CREATE.sql:166-175](repos/Full-Stack-Docker-Tazama/core/postgres/migration/base/00-CREATE.sql#L166-L175) — every one of these is a stored generated column extracted from `transaction->>'…'`: `endToEndId`, `amt` (`numeric(18,2)`), `ccy`, `msgId`, `creDtTm`, `txTp`, `txSts`, `tenantId`. The TypeScript interface [TransactionDetails.ts:3-16](repos/frms-coe-lib/src/interfaces/TransactionDetails.ts#L3-L16) matches. |
| 110-8 | The `event_history.entity` table stores only `id`, `tenantId`, `creDtTm` — no name column. Party display names (`Dbtr.Nm`, `Cdtr.Nm`) are transient pacs.008 message fields, not persisted in the TMS ODS. | **Confirmed against authoritative DDL.** | [00-CREATE.sql:83-88](repos/Full-Stack-Docker-Tazama/core/postgres/migration/base/00-CREATE.sql#L83-L88): `create table entity (id varchar not null, tenantId text not null, creDtTm timestamptz not null, primary key (id, tenantId))`. No name/Nm column exists. |
| 110-9 | `entity.id` composition = `PrvtId.Othr[0].Id + SchmeNm.Prtry` (e.g. `+27730975224MSISDN`). | **Confirmed.** | [eventHistoryBuilder.ts:244-249](repos/frms-coe-lib/src/builders/eventHistoryBuilder.ts#L244-L249): `getEntity` constructs `const id = ${entityId}${schemeProprietary}` before querying. |
| 110-10 | The account-holder join path `source → account → account_holder → entity.id` is valid. | **Confirmed against authoritative DDL.** | [00-CREATE.sql:90-98](repos/Full-Stack-Docker-Tazama/core/postgres/migration/base/00-CREATE.sql#L90-L98): `create table account_holder (source varchar, destination varchar, tenantId text, creDtTm timestamptz, foreign key (source, tenantId) references entity (id, tenantId), foreign key (destination, tenantId) references account (id, tenantId), primary key (source, destination, tenantId))`. So `account_holder.source = entity.id` and `account_holder.destination = account.id` — the join chain in the issue is exact. |
| 110-11 | Party display names (`Dbtr.Nm`, `Cdtr.Nm`) are null on all pacs.002 rows. | **Consistent with ISO 20022 semantics; not code-verifiable here.** | pacs.002 is a payment-status message, not a customer-credit-transfer message; it does not carry the debtor/creditor `Nm` structures. This is a message-schema property, not something I can grep for. Reasonable to accept. |
| 110-12 | `Amt`/`Ccy` are pacs.008-only in the *current* bronze view; `TxSts` is pacs.002-only in the current bronze view. | **Consistent with TransactionsETL.silver()** | [TransactionsETL.py:244](repos/biar/automation-orchestrator/Table_ETLs/TransactionsETL.py#L244) extracts `tx_status` from `$.FIToFIPmtSts.TxInfAndSts.TxSts` — a pacs.002-only path; [line 262-263](repos/biar/automation-orchestrator/Table_ETLs/TransactionsETL.py#L262-L263) extracts `tx_amount`/`tx_ccy` from `$.FIToFICstmrCdtTrf.CdtTrfTxInf.InstdAmt.Amt.*` — a pacs.008-only path. So for any given row, one is null. In `event_history.transaction`, both fields are on the same flat record. |
| 110-13 | Design Requirement 1: rename `bronze/transactions` → `bronze/transaction` (singular) etc., dropping and rebuilding. | **Design assertion; ok.** | Not a code claim. Note: NiFi already emits `table=transaction` (singular), so the singular form is consistent with the NiFi routing attribute. |
| 110-14 | Design Requirement 3: remove the skip guard and add `"transaction": TransactionsETL` / `"transactions": TransactionsETL` aliases. | **Design assertion; ok.** | Consistent with the existing pattern in `_ETL_REGISTRY` where several tables have both singular and plural aliases (e.g. `rule`/`rules`, `typology`/`typologies`). |
| 110-15 | Design Requirement 4: delete `CombinedPacsETL` entirely; route pacs008/pacs002 directly. | **Design assertion; ok.** | Consistent with 110-5 (no real coupling between pacs008 and pacs002 pipelines). |
| 110-16 | Post-MVP Gold-layer pacs enrichment (Design Requirement 6) via `LEFT JOIN gold/pacs008 ON (end_to_end_id, tenant_id)` covers `intr_bk_sttlm_amt`, `intr_bk_sttlm_ccy`, `xchg_rate`, `instg_mmb_id`, `instd_mmb_id`, `charge_count`. | **Consistent with what the code currently extracts.** | These are precisely the pacs-only fields the current TransactionsETL silver stage extracts ([TransactionsETL.py:249-277](repos/biar/automation-orchestrator/Table_ETLs/TransactionsETL.py#L249-L277)) and the current view builders consume ([transaction_detail_view.py:101-116](repos/biar/automation-orchestrator/Table_ETLs/transaction_detail_view.py#L101-L116)). |
| 110-17 | "Before starting: confirm with Justus whether `event_history.transaction.txSts` is updated after pacs.002 receipt." | **Cannot verify from static code.** | This is a runtime/behavioural question about the TMS side of the platform. Legitimate to flag. |

### 4.2 Findings for #110

- Every code-checkable claim in #110 is accurate.
- The most consequential finding — that the currently-live `TransactionsETL` sources from the *wrong* upstream, and the correct upstream is actively being discarded by the skip guard — is unambiguously true from the code.
- The design proposals (delete `CombinedPacsETL`, remove skip guard, add registry entries, drop/rebuild singular tables, entity join for party identifiers) are all consistent with the existing code structure. Nothing in the design contradicts the codebase.
- The one thing I could not check statically is 110-17 (whether TMS actually updates `txSts` after pacs.002 receipt). The reporter has already flagged this — good.

---

## 5. Issue #111 — Downstream Consumer Sweep

**Title (as filed):** DLH: downstream consumers for gold/transaction redesign
**Substance:** Consumer inventory; depends on #110.

### 5.1 Consumer-by-consumer verification

| Consumer (as claimed) | Path in issue | Actual location on dev | Verdict |
|---|---|---|---|
| `transaction_detail_view.py` | `biar/automation-orchestrator/Table_ETLs/` | [automation-orchestrator/Table_ETLs/transaction_detail_view.py](repos/biar/automation-orchestrator/Table_ETLs/transaction_detail_view.py) | **Confirmed.** Reads bronze/transactions and extracts `xchg_rate`, `IntrBkSttlmAmt`, `Dbtr.Nm`, `Cdtr.Nm`, `charge_count`, `instg_mmb_id`, `instd_mmb_id` from ISO 20022 JSON. Short-term pacs join requirement is real. |
| `network_navigator_view.py` | ditto | [automation-orchestrator/Table_ETLs/network_navigator_view.py](repos/biar/automation-orchestrator/Table_ETLs/network_navigator_view.py) | **Confirmed.** Uses only core TMS-equivalent fields (`dbtr_id`, `cdtr_id`, `dbtr_account_id`, `cdtr_account_id`, `tx_amount`, `tx_ccy`, `tx_type`). No usage of settlement-amount, exchange-rate, or party names — so no pacs join needed. Issue's assessment ("confirm whether needed") is defensible but the code shows it is **not** currently needed. |
| `transaction_history_view.py` | ditto | [automation-orchestrator/Table_ETLs/transaction_history_view.py](repos/biar/automation-orchestrator/Table_ETLs/transaction_history_view.py) | **Confirmed.** Extracts `Dbtr.Nm`/`Cdtr.Nm` (at multiple JSON paths — see [lines 114-123](repos/biar/automation-orchestrator/Table_ETLs/transaction_history_view.py#L114-L123)) and `IntrBkSttlmAmt` (at [lines 184, 193](repos/biar/automation-orchestrator/Table_ETLs/transaction_history_view.py#L184-L193)). Short-term pacs join required, as stated. |
| `alert_history_view.py` | ditto | [automation-orchestrator/Table_ETLs/alert_history_view.py](repos/biar/automation-orchestrator/Table_ETLs/alert_history_view.py) | **Confirmed.** Reads `vw_transaction_detail` at [line 80](repos/biar/automation-orchestrator/Table_ETLs/alert_history_view.py#L80); inherits changes from `transaction_detail_view.py`. |
| CMS — Transaction Detail panel | `case-management-system/backend/src/modules/gold-lakehouse/transaction-lakehouse.service.ts` | Same, present on `origin/dev`. | **Confirmed.** `getTransactionDetailData` at line 36 queries `table_name: 'transaction_detail'` at line 42. The main history query joins `transaction_detail td` on `debtor_name` / `creditor_name`. |
| CMS — Transaction Network graph | same file, `getTransactionNetworkData` | Present. | **Confirmed.** [transaction-lakehouse.service.ts:359, 372, 397, 413](repos/case-management-system/backend/src/modules/gold-lakehouse/transaction-lakehouse.service.ts) all reference `transaction_detail`. |
| CMS — Counterparty Network graph | same file, `getCounterpartyNetworkData` | Present. | **Confirmed.** Line 511 onward; joins `transaction_detail`. |
| CMS — Alert transactional data | `case-management-system/backend/src/modules/alert/alert.service.ts` | Present. | **Confirmed.** `getAlertTransactionalData` at [alert.service.ts:125](repos/case-management-system/backend/src/modules/alert/alert.service.ts#L125); SQL at [line 148](repos/case-management-system/backend/src/modules/alert/alert.service.ts#L148): `SELECT * from transaction_detail where end_to_end_id = $1`. |
| JupyterHub — Anomaly Detection notebook | `biar/JupyterHub/notebooks/Anomaly_Detection_And_Rule_Calibration.ipynb` | [Anomaly_Detection_And_Rule_Calibration.ipynb:157,1101](repos/biar/JupyterHub/notebooks/Anomaly_Detection_And_Rule_Calibration.ipynb) | **Confirmed — and the MVP-gap concern is real.** `transactions_path = f"{WAREHOUSE_ROOT}/gold/transactions"` present in two cells. Verified against the **live** notebook running at `http://10.10.80.19:8888/user/admin/shared/Anomaly_Detection_And_Rule_Calibration.ipynb` (cells 7, 8, 10, 14, 15, 20). All three fields are actively used: `instg_mmb_id` / `instd_mmb_id` are consumed by feature `is_cross_border` (cell 10) and by feature `Cross-Bank Indicator` and `Bank-Level TX Count` window partition (cell 20); `charge_count` is consumed by feature `has_charges` (cell 10). More importantly, `instg_mmb_id` / `instd_mmb_id` are also used as **join keys** in cell 10 (`tx_instg_agent == instg_mmb_id AND tx_instd_agent == instd_mmb_id`) — losing them doesn't just drop features, it breaks a join. **Under MVP #110 this notebook is blocked** on either (a) the short-term view-layer `LEFT JOIN gold/pacs008` making these fields available via a joined view, or (b) post-MVP Gold enrichment (#110 Req 6). *Note:* the live notebook's cell content has drifted from the repo copy (same 76 cells, different content hashes) — worth reconciling separately. |
| `ConditionsTimelineView.py` | `biar/automation-orchestrator/Table_ETLs/` | **Does not exist.** | **Incorrect / Not found.** There is no file named `ConditionsTimelineView.py` in `Table_ETLs/`. The closest matches are `ConditionsETL.py` (which does not reference transactions at all) and a path constant `conditions_view_path = f"{VIEWS_ROOT}/conditions_timeline"` in [lakehouse_query_api.py:157](repos/biar/datalakehouse-api/lakehouse_query_api.py#L157) plus registry key `"conditions_timeline"` at line 181. This is likely a stale artefact of the AI-assisted analysis: the "conditions timeline" is exposed only via the DLH Query API, not via a dedicated view builder file. **The consumer entry should be dropped from #111** — the DLH Query API entry already covers the actual code that would need updating. |
| `alert_navigator.py` | ditto | [automation-orchestrator/Table_ETLs/alert_navigator.py](repos/biar/automation-orchestrator/Table_ETLs/alert_navigator.py) | **Confirmed.** [line 396](repos/biar/automation-orchestrator/Table_ETLs/alert_navigator.py#L396) uses `gold/transactions`; selected columns ([lines 405-411](repos/biar/automation-orchestrator/Table_ETLs/alert_navigator.py#L405-L411)) are all core-TMS-equivalent. Path rename only is correct. |
| JupyterHub — Case Management Trend Dashboard | `biar/JupyterHub/notebooks/Case_Management_Trend_Dashboard.ipynb` | [Case_Management_Trend_Dashboard.ipynb:153](repos/biar/JupyterHub/notebooks/Case_Management_Trend_Dashboard.ipynb) | **Confirmed.** `transactions_path = f"{WAREHOUSE_ROOT}/gold/transactions"`. |
| DLH Query API | `biar/datalakehouse-api/lakehouse_query_api.py` | [lakehouse_query_api.py:139,166](repos/biar/datalakehouse-api/lakehouse_query_api.py) | **Confirmed.** Path constant `transactions_gold_path = f"{WAREHOUSE_ROOT}/gold/transactions"` at line 139; registry key `"transactions"` at line 166. |

### 5.2 Missing consumer

**Not in #111's list but should be:** [`views_orchestrator.py`](repos/biar/automation-orchestrator/Table_ETLs/views_orchestrator.py). At [lines 70, 78](repos/biar/automation-orchestrator/Table_ETLs/views_orchestrator.py#L70-L78) it uses `f"{self.warehouse_root}/bronze/transactions"` as a readiness gate for building the transaction/network views. Under the #110 rename this path constant becomes stale and the readiness check silently returns false forever, causing the views to never be built. **This is a small but real omission.**

### 5.3 Findings for #111

- Twelve of the thirteen listed consumers exist exactly where the issue says they do and do exactly what the issue says they do.
- One consumer entry (`ConditionsTimelineView.py`) refers to a file that does not exist on `dev`. The functionality it points at is real (there is a `conditions_timeline` view referenced by the DLH Query API), but the file itself is a hallucination — probably the AI extrapolated a `ConditionsETL.py` + `..._view.py` naming pattern that wasn't followed. This entry should be removed.
- One consumer (`views_orchestrator.py`) is missing from the list. Only a two-line path rename, but critical: without it the affected views stop building silently.
- The severity ratings (🟠 Medium for code refactor + pacs join, 🟡 Low for path rename only) all match what the code actually shows. Field usage claims (e.g. "Anomaly Detection needs `instg_mmb_id`, `instd_mmb_id`, `charge_count` which are pacs-only") were not independently verified — this is a notebook-content claim that I did not open and read the notebook JSON to confirm. Recommend spot-checking during implementation.

---

## 6. Cross-Cutting Observations

### 6.1 Interdependencies

The four issues form one story with clean dependencies:

- **#110 is the root fix.** It rewrites `TransactionsETL` to source from `event_history.transaction`, removes the skip guard, deletes `CombinedPacsETL`, and drops-and-rebuilds `*/transaction` (singular).
- **#108 dissolves as a consequence of #110.** The `transactionData` column disappears; the view-builder candidate list must be hardened (the "transaction" fallback becomes actively dangerous — see §2.1 item 108-5).
- **#111 is the consumer sweep** enabled by #110.
- **#109 is orthogonal to the transaction-source problem.** It is a live crash-loop that should be fixed independently (disable the NiFi processor) with no dependency on #110.

### 6.2 What is *not* in these issues that should probably also be flagged

- The `data-lineage.md` doc ([data-lineage.md:303-627](repos/biar/data-lineage.md#L303-L627)) documents the current wrong-source pipeline as if it were correct — including the note at [line 388](repos/biar/data-lineage.md#L388) that "`gold/transactions` does not carry debtor/creditor account IDs. Those fields ... are in `gold/pacs002`". Under #110 this becomes false and misleading. Updating this doc should be an acceptance criterion on the #110 PR.
- The `CombinedPacsETL` "trigger lock" file at [CombinedPacs.py:148](repos/biar/automation-orchestrator/Table_ETLs/CombinedPacs.py#L148) also creates a state file (`transactions_trigger.lock`) that should be cleaned up along with the marker files listed in #110's Design Requirement 4.

### 6.3 Non-obvious risks

- **The candidate list trap (§2.1 item 108-5) can bite the #110 rewrite.** Once `bronze/transaction` (singular) is built from `event_history.transaction`, its payload column will almost certainly be named `transaction` (matching the ODS schema). The existing view builders' candidate lists all include `"transaction"` as a fallback. If any consumer runs against the new bronze table *before* being rewritten, it will silently pick that column, alias it as `transaction_data`, and parse it as ISO 20022 pacs JSON — producing all-null rows with no error. **A hard failure (raise on unexpected schema) would be safer than a silent fallback.** Adding an explicit acceptance criterion to fail loudly on schema mismatch on the #111 PR is worth considering.
- **CMS depends on `debtor_name` / `creditor_name`** ([transaction-lakehouse.service.ts:184](repos/case-management-system/backend/src/modules/gold-lakehouse/transaction-lakehouse.service.ts#L184) and similar). These are pacs.008-only and null on all pacs.002 rows. Under the MVP scope of #110/#111 they must come from a pacs join at the view layer — but the CMS is a downstream *consumer* of the view, so the loss will surface if the view join is not implemented. The issue #111 flags this as "short-term pacs join required" for the CMS Transaction Detail; that classification is correct.

---

## 7. Recommendations Before Implementation

1. **Verify #109's `db_name` attribute value at runtime**, either by NiFi UI walk-through or by capturing one crash log. The remediation is safe either way, but the issue's line reference (~14118) points at the enrichment DB metadata query rather than the tazama_cms one, which is a small factual weakness.
2. ~~**Verify claim 108-4** (the nested `transactionData` column inside `tazama_cms.transaction_data`).~~ **Done** — confirmed against CMS Prisma schema on `origin/dev`.
3. **Remove the `ConditionsTimelineView.py` entry from #111.** The functionality it points at is already covered by the DLH Query API entry.
4. **Add `views_orchestrator.py` to #111's consumer list.** Small change, high consequence.
5. **Add a hardening acceptance criterion to #110/#111**: the view builders' candidate-column lists must not silently accept a `transaction` column that turns out to be TMS-normalised. Prefer an explicit named-column requirement or a schema check that raises loudly on mismatch.
6. **Add a `data-lineage.md` refresh** to the acceptance criteria of the #110 PR.
7. **Confirm the runtime `txSts` semantic** (110-17) with the TMS side before deciding on the upsert key strategy.
8. ~~**Confirm the JupyterHub Anomaly Detection notebook's actual field usage.**~~ **Done** — verified against the live notebook at `http://10.10.80.19:8888/user/admin/shared/Anomaly_Detection_And_Rule_Calibration.ipynb`. All three fields are used, and `instg_mmb_id` / `instd_mmb_id` are also join keys — so this notebook is blocked on either the short-term view-layer pacs join or post-MVP Gold enrichment (#110 Req 6). Path rename alone is **not** sufficient. Separately: reconcile the live notebook against the repo copy (they have drifted — same 76 cells, different content hashes).
9. **If `tazama_cms.transaction_data` ingestion ever becomes a real requirement (#109 future work):** revisit the proposed `record_key="transactionId"`. The CMS `transactionId` is an autoincrement int — not idempotent across CMS reinstalls. `endToEndId` + `tenantId` would be a more defensible Hudi record key.

---

## 8. Summary Table

| Issue | Substantive accuracy | Notes |
|---|---|---|
| #108 | Accurate. | All claims verified against source (including 108-4 against CMS Prisma schema on dev). The candidate-list trap (108-5) is the most actionable finding and should become a hardening AC on #110/#111. |
| #109 | Accurate on effect, weak on the exact runtime path. | The RuntimeError trigger is exactly as described; the "db_name = tazama_cms attribute origin" story points at the wrong line in tazama.xml. Remediation is safe either way. |
| #110 | Accurate. | All code-checkable claims verified against biar dev + frms-coe-lib dev. One question (110-17, txSts update timing) requires TMS-team input. |
| #111 | Mostly accurate — two nits. | One non-existent file listed (`ConditionsTimelineView.py`); one real consumer missing (`views_orchestrator.py`). Severity classifications and field-usage claims match the code. |

The overall body of work in these four issues is trustworthy enough to plan against. The specific corrections in §7 should be folded in before the issues are used as authoritative implementation input.
