# GitHub Issue & PR Templates — Issue #111

---

## GitHub Issue Body

### Problem

Fourteen consumers across view builders, JupyterHub notebooks, CMS backend services, the DLH Query API, and the views orchestrator reference `bronze/transactions` / `silver/transactions` / `gold/transactions` or read pacs-specific fields (`intr_bk_sttlm_amt`, `xchg_rate`, `instg_mmb_id`, `instd_mmb_id`, `charge_count`, `debtor_name`, `creditor_name`) that MVP `gold/transaction` (post-#110) does not carry. Left as-is, these consumers break the moment #110 lands: `views_orchestrator` never triggers the view ETLs, DLH Query API returns 404 for legacy `?table=transactions`, CMS Transaction Detail panels blank out on party names, and the Anomaly Detection notebook fails on missing `instg_mmb_id` / `instd_mmb_id` / `charge_count` columns (which it uses both as feature-engineering inputs AND as join keys — confirmed against the live notebook at `10.10.80.19:8888`).

### Root Cause

Not an independent bug — this issue is the coordinated consumer sweep for #110. `TransactionsETL` is being rewritten to source from `event_history.transaction` (see #110), the Hudi tables are being renamed from plural to singular, and MVP Gold intentionally excludes pacs-specific enrichment (deferred to post-MVP Design Requirement 6). Every downstream consumer that reads the plural tables or the pacs-only columns is affected.

### Impact

- 7 consumers need only a path rename (`ConditionsTimelineView.py`, `alert_navigator.py`, `views_orchestrator.py`, `Case_Management_Trend_Dashboard.ipynb`, `datalakehouse-api/lakehouse_query_api.py`, and the two path references in `Anomaly_Detection_And_Rule_Calibration.ipynb`).
- 4 view builders need a rewrite to read from `gold/transaction` and — where they display pacs-only fields — to add a short-term view-layer `LEFT JOIN gold/pacs008 ON (end_to_end_id, tenant_id)` (composite key mandatory to avoid cross-tenant collisions).
- 3 CMS backend services depend on the view builders and stay largely unchanged in code, but must be tested end-to-end.
- Anomaly Detection notebook needs a schema-gap decision: add the pacs join at the notebook layer (Option A, recommended), or block on post-MVP (Option C).
- DLH Query API URL contract change: `?table=transactions` → `?table=transaction`. Keep alias for one release; announce deprecation.

Two verification corrections to note:
- `ConditionsTimelineView.py` — was absent from dev at initial review, now present after PR #92 (merged 2026-06-29). Path rename only.
- `views_orchestrator.py` — was not in the original consumer list, but its `_hudi_ready` checks against `bronze/transactions` and `gold/transactions` are load-bearing readiness gates. Path rename only, but critical.

### Proposed Fix

#### Track A — Path renames (7 files, ~2 dev-hours)
Rename `bronze/transactions` / `silver/transactions` / `gold/transactions` → singular in the 7 path-rename consumers. Bundle with #110's PR; do not ship independently.

#### Track B — Full sweep (7 remaining consumers, ~4–6 dev-days)
Rewrite the three transaction-derived view builders to source from `gold/transaction`, use TMS-flat scalar fields and structural columns, and add composite-key `LEFT JOIN gold/pacs008` for pacs-only fields displayed. Test CMS backend service integration. Decide on Anomaly Detection notebook option (recommended: A).

Migration: no PostgreSQL schema changes; view builders rebuild their Hudi outputs on next `views_orchestrator` cycle.

### Acceptance Criteria

- [ ] All 7 path-rename consumers reference singular paths.
- [ ] `datalakehouse-api/lakehouse_query_api.py` `GOLD_PATHS` key `"transaction"` (with `"transactions"` alias for one release).
- [ ] `transaction_detail_view.py`, `transaction_history_view.py`, `network_navigator_view.py` all read `gold/transaction`.
- [ ] `transaction_detail_view.py` and `transaction_history_view.py` include composite-key `LEFT JOIN gold/pacs008 ON (end_to_end_id, tenant_id)` for pacs-only fields.
- [ ] View builders' candidate lists tightened (see #108 Track A) — no silent aliasing.
- [ ] CMS Transaction Detail returns non-null `debtor_name`/`creditor_name` for pacs.008-based rows in E2E test.
- [ ] Anomaly Detection notebook decision documented in the PR body.
- [ ] Live JupyterHub / repo notebook drift acknowledged as separate follow-up.

---

## PR Descriptions

### Track A PR

**PR Title:** `refactor(dlh): rename gold/transactions → gold/transaction across path-only consumers`

**PR Body:**

- **Summary**
  - Rename `bronze/transactions` / `silver/transactions` / `gold/transactions` to singular in 7 consumers.
  - No behaviour change beyond the rename.
- **Test Plan**
  - [ ] Every touched file grep-clean of the plural path.
  - [ ] `views_orchestrator` triggers all 4 transaction-derived view ETLs after #110 lands.
  - [ ] DLH Query API responds to both `?table=transactions` (alias) and `?table=transaction` (new).
- **Related Issue:** #111
- **Sequencing Note:** Must land in the same PR family as #110.

### Track B PR

**PR Title:** `refactor(dlh): rewrite transaction views for gold/transaction + short-term pacs join (fixes #111)`

**PR Body:**

- **Summary**
  - Rewrite `transaction_detail_view.py`, `transaction_history_view.py`, `network_navigator_view.py` to source from `gold/transaction`.
  - Add composite-key `LEFT JOIN gold/pacs008 ON (end_to_end_id, tenant_id)` in `transaction_detail_view.py` and `transaction_history_view.py` to surface `debtor_name`, `creditor_name`, `xchg_rate`, `intr_bk_sttlm_amt`.
  - `network_navigator_view.py` — no pacs join needed; uses structural fields only.
  - `alert_history_view.py` unchanged (inherits from vw_transaction_detail).
  - CMS backend services unchanged in code; add E2E tests.
  - Anomaly Detection notebook: Option A (pacs join at notebook layer).
- **Test Plan**
  - [ ] Unit tests: view builders produce expected column set incl. joined pacs fields.
  - [ ] Unit tests: `network_navigator_view.py` does not read `gold/pacs008`.
  - [ ] Integration: pacs.008 message → `vw_transaction_detail` row with names; matching pacs.002 → same row's `tx_status` updates.
  - [ ] Integration: CMS Transaction Detail shows `debtor_name`/`creditor_name` for pacs.008-based rows.
  - [ ] Integration: Anomaly Detection notebook completes cell 20 without missing-column error.
  - [ ] Integration: `views_orchestrator` triggers all 4 views after #110's Gold table populates.
  - [ ] Manual: DLH Query API `?table=transactions` (alias) and `?table=transaction` both return data.
- **Related Issue:** #111
- **Migration**
  - No PostgreSQL migration.
  - View builders rebuild their Hudi outputs on next `views_orchestrator` cycle. If clean state is needed, delete `views/vw_transaction_detail`, `views/vw_transaction_history`, `views/network_navigator`, `views/vw_conditions_timeline` directories before deploy.
- **Sequencing Note**
  - Must land AFTER #110's Gold rebuild has produced `gold/transaction` at least once (verify via `hudi-cli` show tables).
  - Must land WITH #108 view-builder candidate-list hardening in the same PR family.
  - Post-MVP: retire per-consumer pacs joins after Design Requirement 6 lands in a follow-up PR.
