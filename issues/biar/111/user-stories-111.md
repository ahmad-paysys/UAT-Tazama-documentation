# User Stories — Issue #111

Each story below is a discrete piece of work under Tracks A and B of #111. They land as one atomic PR family with [#110](../110/) (see [github-issue-111.md](github-issue-111.md)).

---

## US-111-1 — Path rename `views_orchestrator.py`

**Body:**
Change three lines in `views_orchestrator.py` from plural to singular. Without this, the four transaction-derived view ETLs (`TransactionDetailViewETL`, `TransactionHistoryViewETL`, `NetworkNavigatorViewETL`, `ConditionsTimelineViewETL`) never trigger after #110's Hudi rename.

**Scope:**
- Track: A
- Files: `automation-orchestrator/Table_ETLs/views_orchestrator.py`
- Change: text edits on lines 71, 79, 108.

**Acceptance Criteria:**
- [ ] `bronze/transactions` → `bronze/transaction` on lines 71, 79.
- [ ] `gold/transactions` → `gold/transaction` on line 108.
- [ ] `views_orchestrator.run()` triggers all 4 transaction-derived view builders in UAT.

**Testing:**
- Manual: `views_orchestrator.run()` invocation in UAT after #110 deploy.

---

## US-111-2 — Path rename `ConditionsTimelineView.py`

**Body:**
Change the `gold/transactions` path to `gold/transaction`.

**Scope:**
- Track: A
- Files: `automation-orchestrator/Table_ETLs/ConditionsTimelineView.py`
- Change: line 98.

**Acceptance Criteria:**
- [ ] `gold/transaction` referenced; `gold/transactions` no longer referenced.
- [ ] `ConditionsTimelineViewETL.run()` produces `vw_conditions_timeline` in UAT.

**Testing:**
- Unit: mock `gold/transaction` reads; verify view builder assembles.

---

## US-111-3 — Path rename `alert_navigator.py`

**Body:**
Change the `gold/transactions` path constant to singular.

**Scope:**
- Track: A
- Files: `automation-orchestrator/Table_ETLs/alert_navigator.py`
- Change: line 396.

**Acceptance Criteria:**
- [ ] `gold/transaction` referenced.
- [ ] Alert Navigator view assembles in UAT.

**Testing:**
- Unit / integration as above.

---

## US-111-4 — Path rename DLH Query API (with temporary alias)

**Body:**
Rename `transactions_gold_path` → `transaction_gold_path`; update `GOLD_PATHS` key to `"transaction"`; retain `"transactions"` as a deprecation alias for one release. Publish deprecation notice via API docs.

**Scope:**
- Track: A
- Files: `datalakehouse-api/lakehouse_query_api.py`
- Change: lines 139, 166.

**Acceptance Criteria:**
- [ ] `GOLD_PATHS["transaction"]` returns `gold/transaction` path.
- [ ] `GOLD_PATHS["transactions"]` (alias) returns same path for one release.
- [ ] API OpenAPI/docs mention deprecation.

**Testing:**
- Curl: `GET /query?table=transactions` returns data with `Deprecation: true` response header.
- Curl: `GET /query?table=transaction` returns same data.

---

## US-111-5 — Path rename Case Management Trend Dashboard notebook

**Body:**
Change the `transactions_path` variable in the notebook.

**Scope:**
- Track: A
- Files: `JupyterHub/notebooks/Case_Management_Trend_Dashboard.ipynb`
- Change: cell containing the path constant.

**Acceptance Criteria:**
- [ ] Notebook loads `gold/transaction`.
- [ ] Notebook renders visualisations without error in UAT.

**Testing:**
- Manual: load notebook via JupyterHub after deploy.

---

## US-111-6 — Rewrite `transaction_detail_view.py` with pacs join

**Body:**
Switch to `gold/transaction`. Remove the candidate-list JSON resolver (see #108). Add composite-key `LEFT JOIN gold/pacs008 ON (end_to_end_id, tenant_id)` to source `debtor_name`, `creditor_name`, `intrbk_sttlm_amt`, `intrbk_sttlm_ccy`, `xchg_rate`, `instg_mmb_id`, `instd_mmb_id`, `charge_count`.

**Scope:**
- Track: B
- Files: `automation-orchestrator/Table_ETLs/transaction_detail_view.py`
- Change: full rewrite.

**Acceptance Criteria:**
- [ ] Reads `gold/transaction` and `gold/pacs008`.
- [ ] Composite-key join on both `end_to_end_id` AND `tenant_id`.
- [ ] Emits `debtor_name`, `creditor_name`, `intrbk_sttlm_amt`, `intrbk_sttlm_ccy`, `exchange_rate`, `instg_mmb_id`, `instd_mmb_id`, `charge_count` (all nullable — null on pacs.002-only rows).
- [ ] Emits `debtor_id`, `creditor_id` from `debtor_entity_id`, `creditor_entity_id` (structural — always populated).

**Testing:**
- Unit: mock two DFs; assert join behaviour.
- Integration: pacs.008 + pacs.002 → view has names on both rows (from pacs.008 side).

---

## US-111-7 — Rewrite `transaction_history_view.py` with pacs join

**Body:**
Same treatment as `transaction_detail_view.py` — switch to `gold/transaction`; add composite-key `LEFT JOIN gold/pacs008` for the display fields the view exposes.

**Scope:**
- Track: B
- Files: `automation-orchestrator/Table_ETLs/transaction_history_view.py`
- Change: full rewrite.

**Acceptance Criteria:**
- [ ] Reads `gold/transaction`.
- [ ] Includes composite-key `LEFT JOIN gold/pacs008`.
- [ ] Emits the same field set as before but sourced from the join, not from raw ISO paths.

**Testing:**
- Unit / integration as above.

---

## US-111-8 — Rewrite `network_navigator_view.py` — no pacs join

**Body:**
Switch to `gold/transaction`. Use `source_account_id`, `destination_account_id`, `debtor_entity_id`, `creditor_entity_id` directly. Remove all ISO path extractions. **No pacs join needed** (grep-verified — the view uses only structural fields).

**Scope:**
- Track: B
- Files: `automation-orchestrator/Table_ETLs/network_navigator_view.py`
- Change: full rewrite of resolution logic.

**Acceptance Criteria:**
- [ ] Reads `gold/transaction` only.
- [ ] No reads from `gold/pacs008`, `bronze/pacs008`, or `bronze/transaction`.
- [ ] Emits `dbtr_id`, `cdtr_id`, `dbtr_account_id`, `cdtr_account_id`, `tx_amount`, `tx_ccy`, `tx_type`.

**Testing:**
- Unit: mock `gold/transaction` DF; view produces expected columns.

---

## US-111-9 — CMS backend integration tests

**Body:**
The CMS services `transaction-lakehouse.service.ts` and `alert.service.ts` query `transaction_detail` / `vw_transaction_detail`. No code change expected. Add E2E integration tests confirming that `getTransactionDetailData`, `getTransactionNetworkData`, `getCounterpartyNetworkData`, and `getAlertTransactionalData` all return non-null names for pacs.008-based rows after the view rewrite.

**Scope:**
- Track: B
- Files: CMS test suite.
- Change: add tests; no production code change.

**Acceptance Criteria:**
- [ ] Test seeds a pacs.008 + matching pacs.002 scenario.
- [ ] `getTransactionDetailData(endToEndId)` returns `debtor_name` and `creditor_name` populated.
- [ ] `getAlertTransactionalData(endToEndId)` returns similarly populated row.

**Testing:**
- As above.

---

## US-111-10 — Anomaly Detection notebook — schema-gap decision

**Body:**
The Anomaly Detection notebook uses `instg_mmb_id`, `instd_mmb_id`, `charge_count` both as feature-engineering inputs (features `is_cross_border`, `has_charges`, `Cross-Bank Indicator`, `Bank-Level TX Count`) AND as join keys (cell 10, verified against the live notebook at `10.10.80.19:8888`). Under MVP `gold/transaction` these fields are absent. Choose one:

- **Option A (recommended):** add a Spark join to `gold/pacs008` at the top of the notebook, producing `instg_mmb_id`/`instd_mmb_id`/`charge_count` before the feature-engineering cells.
- **Option B:** create a new view builder `anomaly_features_view.py` that provides the joined output; notebook reads from that view.
- **Option C:** mark notebook blocked on post-MVP Gold enrichment; do not run it.

**Scope:**
- Track: B
- Files: `JupyterHub/notebooks/Anomaly_Detection_And_Rule_Calibration.ipynb` (Options A/B) or a documentation-only marker (Option C).
- Change: notebook cells (A), or new view builder (B), or issue tracking (C).

**Acceptance Criteria:**
- [ ] Decision documented in the PR body.
- [ ] Under Option A: cell 8/9 adds a pacs join before cells 10 and 20; both cells run without error.
- [ ] Under Option B: notebook path constant points to the new view.
- [ ] Under Option C: notebook is marked blocked in its README/first cell.

**Testing:**
- Manual: run notebook cells 1–20 in UAT after deploy.

---

## US-111-11 — Reconcile live-vs-repo JupyterHub notebook drift

**Body:**
All 9 shared notebooks on the live JupyterHub have drifted from the copies in `biar/JupyterHub/notebooks/` on dev (same or off-by-one cell counts, different content hashes). See [repos/JupyterNotebooks/README.md](../../../repos/JupyterNotebooks/README.md) for the drift table. This is not in #111's critical path but should be reconciled to prevent future confusion.

**Scope:**
- Track: separate follow-up (not required for #110/#111 to close)
- Files: 9 `*.ipynb` files under `biar/JupyterHub/notebooks/`
- Change: sync repo copies from live, or vice versa, per team decision.

**Acceptance Criteria:**
- [ ] Content hashes match between live and repo.

**Testing:**
- SHA-1 hash comparison (script in `repos/JupyterNotebooks/README.md`).

---

## US-111-12 — Post-MVP: retire per-consumer pacs joins

**Body:**
Once #110 Design Requirement 6 (post-MVP Gold enrichment) lands and `gold/transaction` includes `intr_bk_sttlm_amt`, `xchg_rate`, `instg_mmb_id`, `instd_mmb_id`, `charge_count` (and any other pacs-only fields promoted), remove the short-term `LEFT JOIN gold/pacs008` from `transaction_detail_view.py`, `transaction_history_view.py`, and (if used) the Anomaly Detection notebook.

**Scope:**
- Track: post-MVP follow-up
- Files: 2–3 view builders + notebook
- Change: remove pacs join; read enriched columns directly from `gold/transaction`.

**Acceptance Criteria:**
- [ ] No view builder references `gold/pacs008` for cross-join enrichment.
- [ ] Notebook Option A join (if chosen) removed.

**Testing:**
- Regression: field values match pre-retirement values for a representative pacs.008 + pacs.002 scenario.
