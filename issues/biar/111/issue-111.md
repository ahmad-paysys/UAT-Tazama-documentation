# Issue #111 — DLH: downstream consumers for gold/transaction redesign

**Repository:** tazama-lf/biar
**Issue:** [DLH: downstream consumers for gold/transaction redesign](https://github.com/tazama-lf/biar/issues/111)
**Author:** Sandy-at-Tazama
**State:** Open
**Report Date:** 2026-07-08
**Verified against:** biar `ef0c856` (dev), case-management-system `2b14c12f` (dev), Full-Stack-Docker-Tazama `e218211` (dev), live JupyterHub at `http://10.10.80.19:8888`, live NiFi at `http://10.10.80.19:8088`.

---

## Executive Summary

Tracks the downstream consumer updates required after issue #110's `TransactionsETL` rewrite lands and `gold/transaction` (singular) replaces `gold/transactions` (plural). Fourteen consumers across view builders, JupyterHub notebooks, CMS services, the DLH Query API, and one additional consumer discovered during verification (`views_orchestrator.py`) reference the plural table names or read pacs-specific fields that will not exist in MVP `gold/transaction`.

MVP `gold/transaction` (per [#110](../110/)) delivers all TMS core fields plus `debtor_entity_id`/`creditor_entity_id` from the account-holder → entity join. It does **not** deliver pacs-specific settlement/exchange/agent fields (`intr_bk_sttlm_amt`, `xchg_rate`, `instg_mmb_id`, `instd_mmb_id`, `charge_count`). Consumers requiring those add a short-term view-layer `LEFT JOIN gold/pacs008 ON (end_to_end_id, tenant_id)`. Post-MVP Gold enrichment (Design Requirement 6 in #110) will centralise the join and remove the short-term per-consumer joins.

**Verification notes:**
- `ConditionsTimelineView.py` referenced in the original issue was not on `dev` at initial review but **has since been merged via PR #92** (`fix: added CMS username for dashboards and fixed TMS metrics calculation`, merged 2026-06-29). It is now on `dev` at `automation-orchestrator/Table_ETLs/ConditionsTimelineView.py` and reads `gold/transactions` at line 98. Path rename only (Low severity).
- **Newly identified consumer not in the original issue:** `automation-orchestrator/Table_ETLs/views_orchestrator.py` uses `bronze/transactions` at lines 71, 79 and `gold/transactions` at line 108 as readiness gates. Small path-rename change but critical — without it, the view builders would never trigger after #110's rename.
- **JupyterHub Anomaly Detection notebook (upgraded from "confirm" to "confirmed"):** live inspection of the running notebook at `http://10.10.80.19:8888/user/admin/lab` confirms it uses `instg_mmb_id`, `instd_mmb_id`, and `charge_count` as **both feature-engineering inputs (features `is_cross_border`, `has_charges`, `Cross-Bank Indicator`, `Bank-Level TX Count`) AND join keys (cell 10)**. Losing them doesn't just drop features — it breaks a join. Path rename alone is insufficient.

---

## How It Works Today — Confirmed in Code

For each of the 14 consumers, the exact reference locations are recorded below. Verification notes reflect the state on the specified branches at snapshot time.

### 🟠 Medium — code refactor required

#### 1. `transaction_detail_view.py`

**Location:** `biar/automation-orchestrator/Table_ETLs/transaction_detail_view.py`

Reads `bronze/transactions` (via `self.transactions_bronze_path` at line ~27) and extracts:
- `xchg_rate` from `$.FIToFICstmrCdtTrf.CdtTrfTxInf.XchgRate` at line 103
- `IntrBkSttlmAmt` at lines 101-102
- `Dbtr.Nm` / `Cdtr.Nm` at lines 89, 91
- `charge_count` computed at lines 116-152
- `instg_mmb_id` / `instd_mmb_id` at lines 106-113

**Required:** rename to `bronze/transaction`; replace ISO-path extractions with structured column reads; add short-term `LEFT JOIN gold/pacs008` for `xchg_rate` and `intr_bk_sttlm_amt`; use `debtor_entity_id`/`creditor_entity_id` from #110 Silver.

#### 2. `network_navigator_view.py`

**Location:** `biar/automation-orchestrator/Table_ETLs/network_navigator_view.py:25`

Reads `bronze/transactions`; extracts:
- `dbtr_id` / `cdtr_id` at lines 97-98 (ISO paths)
- `dbtr_account_id` / `cdtr_account_id` at lines 99-100
- `tx_amount` / `tx_ccy` at lines 101-102

**No usage of settlement-amount, exchange-rate, or party names** — confirmed by grep. Required: rename to `bronze/transaction`; use `source_account_id`/`destination_account_id` and `debtor_entity_id`/`creditor_entity_id` directly. No pacs join needed.

#### 3. `transaction_history_view.py`

**Location:** `biar/automation-orchestrator/Table_ETLs/transaction_history_view.py:25`

Extracts `Dbtr.Nm` / `Cdtr.Nm` at lines 114-123 (three fallback paths each) and `IntrBkSttlmAmt` at lines 184, 193. Required: rename to `bronze/transaction`; add short-term `LEFT JOIN gold/pacs008` for names and settlement amount if kept in the view.

#### 4. `alert_history_view.py`

**Location:** `biar/automation-orchestrator/Table_ETLs/alert_history_view.py:80`

Reads `vw_transaction_detail`. Inherits all changes from `transaction_detail_view.py` (item #1). No independent change needed.

#### 5. CMS — Transaction Detail panel

**Location:** `case-management-system/backend/src/modules/gold-lakehouse/transaction-lakehouse.service.ts@origin/dev`
- `getTransactionDetailData` at line 36
- Queries `table_name: 'transaction_detail'` at line 42
- Main history join at line 184: `LEFT JOIN transaction_detail td ON th.transaction_id = td.transaction_id AND th.tenant_id = td.tenant_id` — uses `td.debtor_name`, `td.creditor_name`.

Depends on `vw_transaction_detail` (item #1). Update after view builder redesign.

#### 6. CMS — Transaction Network graph

**Location:** same file, `getTransactionNetworkData` at line 359. Depends on `transaction_detail` view.

#### 7. CMS — Counterparty Network graph

**Location:** same file, `getCounterpartyNetworkData` at line 511. Depends on `transaction_detail` view.

#### 8. CMS — Alert transactional data

**Location:** `case-management-system/backend/src/modules/alert/alert.service.ts:125` — `getAlertTransactionalData`, SQL at line 148: `SELECT * from transaction_detail where end_to_end_id = $1`. Depends on `vw_transaction_detail` (item #1).

#### 9. JupyterHub — Anomaly Detection notebook

**Location:** `biar/JupyterHub/notebooks/Anomaly_Detection_And_Rule_Calibration.ipynb` (cells 7, 8, 10, 14, 15, 20). **Live copy verified against the running notebook** at `http://10.10.80.19:8888/user/admin/shared/Anomaly_Detection_And_Rule_Calibration.ipynb`. Uses:
- `transactions_path = f"{WAREHOUSE_ROOT}/gold/transactions"` (cells 7, 14) — path rename required.
- `col("instg_mmb_id")`, `col("instd_mmb_id")`, `col("charge_count")` at cell 10, lines 238–240 — feature-engineering inputs.
- `is_cross_border = when(col("instg_mmb_id") != col("instd_mmb_id"), 1).otherwise(0)` at cell 10 lines 413–418 — derived feature.
- `has_charges = when(col("charge_count") > 0, 1).otherwise(0)` at cell 10 lines 419–424 — derived feature.
- `(col("tx_instg_agent") == col("instg_mmb_id"))` at cell 10 lines 440–450 — **join key**.
- `Cross-Bank Indicator` feature at cell 20 (`col("tx.instg_mmb_id") != col("tx.instd_mmb_id")`).
- `Bank-Level TX Count` window partition on `tx.instg_mmb_id` at cell 20.

**Consequence:** path rename alone is insufficient. This notebook is blocked on either (a) the short-term view-layer `LEFT JOIN gold/pacs008 ON (end_to_end_id, tenant_id)` producing `instg_mmb_id`, `instd_mmb_id`, `charge_count` as if they were part of gold/transaction, or (b) post-MVP Gold enrichment (#110 Design Requirement 6).

**Additional gap:** the live notebook and the repo copy on `dev` have drifted — same 76 cells, different content hashes. See [repos/JupyterNotebooks/README.md](../../../repos/JupyterNotebooks/README.md). This drift should be reconciled separately from #111.

### 🟡 Low — path rename only

#### 10. `ConditionsTimelineView.py`

**Location:** `biar/automation-orchestrator/Table_ETLs/ConditionsTimelineView.py:98` — `spark.read.format("hudi").load(f"{self.warehouse_root}/gold/transactions")`. Confirmed on dev after PR #92 merge. Reads only structural fields — no `IntrBkSttlm` / `XchgRate` / `Nm` / `instg` / `instd` / `charge_count` / `debtor_name` / `creditor_name` (grep-verified). Path rename only.

#### 11. `alert_navigator.py`

**Location:** `biar/automation-orchestrator/Table_ETLs/alert_navigator.py:396`. Reads columns `tx_msg_id, tx_status, tx_amount, tx_ccy, transaction_id, end_to_end_id` — all TMS core. Path rename only.

#### 12. JupyterHub — Case Management Trend Dashboard

**Location:** `biar/JupyterHub/notebooks/Case_Management_Trend_Dashboard.ipynb:153`. Path rename only.

#### 13. DLH Query API

**Location:** `biar/datalakehouse-api/lakehouse_query_api.py:139, 166`. Path constant `transactions_gold_path = f"{WAREHOUSE_ROOT}/gold/transactions"` and `GOLD_PATHS` key `"transactions"`. Rename to `"transaction"` (singular) — but note: this changes the API URL keys for API clients. Coordinate with API consumers.

### 🟡 Additional consumer (not in original issue)

#### 14. `views_orchestrator.py` (Low — path rename)

**Location:** `biar/automation-orchestrator/Table_ETLs/views_orchestrator.py:71, 79, 108`. Uses `bronze/transactions` (lines 71, 79) and `gold/transactions` (line 108, added by the recent ConditionsTimeline merge) as readiness gates. Small text edits but critical: without them, `TransactionDetailViewETL`, `TransactionHistoryViewETL`, `NetworkNavigatorViewETL`, and `ConditionsTimelineViewETL` never trigger after the #110 rename.

---

## Root Cause Analysis

There is no independent root cause — this issue exists because the underlying wrong-source problem (#110) requires the plural-to-singular rename plus a partial schema change (pacs-specific fields dropped from Gold), and both changes are visible at every downstream. Consumer effort splits into "just rename the path" (7 consumers including the two `views_orchestrator.py` / `ConditionsTimelineView.py` late additions) and "adjust to the new schema, adding a short-term pacs join if the deferred fields are needed" (7 consumers).

---

## Blast Radius — Full File Inventory

### Files that must change

| File | Category | Reason |
|---|---|---|
| `automation-orchestrator/Table_ETLs/transaction_detail_view.py` | 🟠 | Rewrite to `gold/transaction`; add pacs join |
| `automation-orchestrator/Table_ETLs/network_navigator_view.py` | 🟠 | Rewrite to `gold/transaction`; no pacs join needed |
| `automation-orchestrator/Table_ETLs/transaction_history_view.py` | 🟠 | Rewrite to `gold/transaction`; add pacs join for names + IntrBkSttlm |
| `automation-orchestrator/Table_ETLs/alert_history_view.py` | 🟠 | Downstream of #1 |
| `automation-orchestrator/Table_ETLs/ConditionsTimelineView.py` | 🟡 | Path rename |
| `automation-orchestrator/Table_ETLs/alert_navigator.py` | 🟡 | Path rename |
| `automation-orchestrator/Table_ETLs/views_orchestrator.py` | 🟡 | Path rename |
| `case-management-system/backend/src/modules/gold-lakehouse/transaction-lakehouse.service.ts` | 🟠 | Depends on view rewrite |
| `case-management-system/backend/src/modules/alert/alert.service.ts` | 🟠 | Depends on view rewrite |
| `biar/JupyterHub/notebooks/Anomaly_Detection_And_Rule_Calibration.ipynb` | 🟠 | Path + MVP gap (join key loss) |
| `biar/JupyterHub/notebooks/Case_Management_Trend_Dashboard.ipynb` | 🟡 | Path rename |
| `biar/datalakehouse-api/lakehouse_query_api.py` | 🟡 | Path constant + registry key |

### Files indirectly affected

| File | Why |
|---|---|
| `biar/data-lineage.md` | Documents the pipeline (updated in #110) |
| Live JupyterHub notebooks (drifted from repo) | Reconciliation not in #111 scope |

---

## Side-Effect Map

| Consequence if left unfixed after #110 lands | Impact |
|---|---|
| `views_orchestrator.py` never triggers `TransactionDetailViewETL` / `TransactionHistoryViewETL` / `NetworkNavigatorViewETL` / `ConditionsTimelineViewETL` because `bronze/transactions` and `gold/transactions` no longer exist | Critical — all transaction-derived views stop building |
| DLH Query API returns 404 for `?table=transactions` | High — external consumers break |
| Anomaly Detection notebook fails on missing `instg_mmb_id` / `instd_mmb_id` / `charge_count` columns | High — MVP training pipeline stops |
| CMS Transaction Detail panel shows blank `debtor_name`/`creditor_name` for pacs.008-based rows | Medium-high — UX regression |
| Two Jupyter notebooks fail to load transactions path | Low — path rename is trivial |

---

## Effort Assessment

### Track A — Path renames only (partial fix)

Ship the four path-rename consumers ahead of the full #111 work — `alert_navigator.py`, `ConditionsTimelineView.py`, `views_orchestrator.py`, `Case_Management_Trend_Dashboard.ipynb`, and the DLH Query API constant. But **do not merge Track A before #110 lands** — the paths would be wrong under the current plural naming.

| Step | File | Effort |
|---|---|---|
| 5× path-rename edits | 5 files | ~1 dev-hour |
| Verify via live UAT | — | ~1 hour |

Schema migration: no. Frontend changes: DLH Query API URL params change may affect API consumers — communicate.

### Track B — Full sweep including view rewrites + CMS + notebook

The 🟠 consumers need to be updated together with #110 in the same PR family; otherwise the views break for the transition window.

| Step | Files | Effort |
|---|---|---|
| Rewrite `transaction_detail_view.py` | 1 | ~1 dev-day |
| Rewrite `transaction_history_view.py` | 1 | ~1 dev-day |
| Rewrite `network_navigator_view.py` | 1 | ~0.5 dev-day |
| CMS backend column adjustments | 2 | ~0.5 dev-day |
| Anomaly Detection notebook — either add view-layer pacs join upstream (in a view builder) or flag as blocked | 1 | ~0.5–1 dev-day |
| Path renames (7 consumers) | 7 | ~1 dev-hour |
| UAT | — | ~1 dev-day |

Schema migration: no. Frontend: none directly; CMS backend view-column dependencies covered.

### Recommended sequencing

1. Confirm `txSts` update semantics (dependency from #110-US10).
2. Ship #110 + #111 as one atomic PR family (or two tightly-coupled PRs merged together).
3. Post-merge: reconcile drifted JupyterHub notebooks separately.
4. Post-MVP: retire per-consumer pacs joins once #110's Design Requirement 6 (Gold enrichment) lands.

---

## Acceptance Criteria

### Track A (path renames only — partial)

- [ ] `ConditionsTimelineView.py`, `alert_navigator.py`, `views_orchestrator.py`, `Case_Management_Trend_Dashboard.ipynb`, and `lakehouse_query_api.py` reference `bronze/transaction` / `silver/transaction` / `gold/transaction` (singular).
- [ ] DLH Query API's `GOLD_PATHS` uses key `"transaction"` instead of `"transactions"`.
- [ ] Any API URL query strings using `?table=transactions` return a helpful deprecation error or transparently route to the new key (decide with API consumers).

### Track B (full sweep)

- [ ] All three view builders (`transaction_detail_view.py`, `transaction_history_view.py`, `network_navigator_view.py`) read `gold/transaction`; use `source_account_id`, `destination_account_id`, `debtor_entity_id`, `creditor_entity_id`.
- [ ] `transaction_detail_view.py` and `transaction_history_view.py` include `LEFT JOIN gold/pacs008 ON (end_to_end_id, tenant_id)` for `debtor_name`/`creditor_name`, `xchg_rate`, `intr_bk_sttlm_amt` (and any other pacs-only fields displayed).
- [ ] Anomaly Detection notebook: either (a) upstream view builder exposes `instg_mmb_id`/`instd_mmb_id`/`charge_count` via pacs join and notebook consumes them, or (b) notebook is marked blocked on post-MVP.
- [ ] CMS Transaction Detail / Alert transactional data queries against `transaction_detail` continue to return `debtor_name`/`creditor_name` for pacs.008-based rows.
- [ ] All 14 consumer files rebuilt/tested without errors.
- [ ] View-builder candidate lists tightened (#108 Track A) — no silent aliasing.

---
