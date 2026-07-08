# Impact Study — Issue #111: downstream consumers for gold/transaction redesign

**Issue:** [#111](https://github.com/tazama-lf/biar/issues/111)
**Study Date:** 2026-07-08
**Author:** Ahmad Khalid
**Repository:** tazama-lf/biar

---

## Summary

Fourteen consumers reference `bronze/transactions` / `silver/transactions` / `gold/transactions` or read pacs-specific fields that are being removed from Gold at MVP scope in [#110](../110/). Seven need path renames only (Low severity); seven need code refactors including a short-term view-layer `LEFT JOIN gold/pacs008` for pacs-only fields (Medium). **Track A** ships the low-severity path renames; **Track B** ships the full consumer sweep. Both must land together with #110 to avoid a transition window with broken views. Post-MVP Gold enrichment (Design Requirement 6 in #110) later retires the per-consumer pacs joins.

Verification against dev revealed two important corrections to the original issue:
1. **`ConditionsTimelineView.py`** — originally absent from dev; now present after PR #92 merge (2026-06-29). Path-rename-only.
2. **`views_orchestrator.py`** — not in the original issue's list; small path-rename change but critical (readiness gates).

Live JupyterHub inspection also upgraded the Anomaly Detection notebook from "confirm field usage" to "confirmed: `instg_mmb_id`/`instd_mmb_id` are also join keys" — so path rename alone is insufficient for that consumer.

---

## Confirmed Root Cause

- Not an independent bug — this is the coordinated consumer sweep for #110.
- 14 consumers total; verified at file/line level against biar `ef0c856` (dev) and case-management-system `2b14c12f` (dev).
- Anomaly Detection notebook usage verified against the live copy at `http://10.10.80.19:8888/user/admin/shared/Anomaly_Detection_And_Rule_Calibration.ipynb` on 2026-07-08.
- Live NiFi at `http://10.10.80.19:8088` confirms the `Transaction` (singular) processor is running and producing Ozone JSON that #110's rewritten `TransactionsETL` will consume.

---

## Track A — Path renames (7 consumers)

### What Changes

Rename `*/transactions` → `*/transaction` in seven files. No schema logic changes.

Files:
- `automation-orchestrator/Table_ETLs/ConditionsTimelineView.py`
- `automation-orchestrator/Table_ETLs/alert_navigator.py`
- `automation-orchestrator/Table_ETLs/views_orchestrator.py`
- `JupyterHub/notebooks/Case_Management_Trend_Dashboard.ipynb`
- `JupyterHub/notebooks/Anomaly_Detection_And_Rule_Calibration.ipynb` (path rename; but see Track B for the deeper concern)
- `datalakehouse-api/lakehouse_query_api.py`

### Impact

| Property | Value |
|---|---|
| Files changed | 6–7 (Anomaly counts once for path; Track B revisits it) |
| Schema migration required | No |
| Frontend changes required | No |
| Downtime required | No |
| Risk of regression | Low if landed atomically with #110; High if landed before |
| Reversibility | Trivial (revert commit) |

What this fixes immediately:
- `views_orchestrator.py` triggers views correctly after #110's rename.
- `ConditionsTimelineView.py`, `alert_navigator.py`, both notebooks, and the DLH Query API point at the correct path.

What this does not fix:
- Anything about the schema changes (pacs field removal at Gold). Consumers reading pacs-only fields still break — see Track B.

Track A is safe to ship in isolation *only if* it is merged in the same PR family as #110. It is **not** safe to ship before #110 lands (paths would be wrong under current plural naming).

---

## Track B — Full consumer sweep (7 remaining consumers + hardening)

### What Changes

Consumers that read pacs-only fields at the view or notebook layer must be rewired to source those fields via a short-term `LEFT JOIN gold/pacs008 ON (end_to_end_id, tenant_id)`. TMS core fields (which #110 delivers at Gold) come from the new `gold/transaction`.

### Schema Impact

No schema changes at this layer. Downstream schema is set by #110's Gold schema.

### Backend Code Impact

#### Core rewrites (Medium severity)

| File | Change |
|---|---|
| `automation-orchestrator/Table_ETLs/transaction_detail_view.py` | Switch to `gold/transaction`. Replace ISO path extractions with structured column reads. Use `debtor_entity_id`/`creditor_entity_id`. **Add** `LEFT JOIN gold/pacs008` for `xchg_rate`, `intr_bk_sttlm_amt`, `debtor_name`, `creditor_name`. |
| `automation-orchestrator/Table_ETLs/transaction_history_view.py` | Switch to `gold/transaction`. **Add** `LEFT JOIN gold/pacs008` for the same fields displayed. |
| `automation-orchestrator/Table_ETLs/network_navigator_view.py` | Switch to `gold/transaction`. Use `source_account_id`, `destination_account_id`, `debtor_entity_id`, `creditor_entity_id`. **No pacs join needed** (grep-verified). |
| `automation-orchestrator/Table_ETLs/alert_history_view.py` | Reads `vw_transaction_detail` — inherits changes from `transaction_detail_view.py`. No independent change. |

#### CMS backend

| File | Change |
|---|---|
| `case-management-system/backend/src/modules/gold-lakehouse/transaction-lakehouse.service.ts` | Continues querying `vw_transaction_detail` — no code change per se, but the queries assume `debtor_name`/`creditor_name` columns exist. Once the view builders (above) provide them via pacs join, this file stays as-is. Verify column presence in test. |
| `case-management-system/backend/src/modules/alert/alert.service.ts` | Same — reads `transaction_detail`. |

#### Notebook

| File | Change |
|---|---|
| `biar/JupyterHub/notebooks/Anomaly_Detection_And_Rule_Calibration.ipynb` | Path rename + **schema gap decision**. Options: (a) add pacs join at a view builder feeding the notebook and read from that view instead of raw `gold/transaction`; (b) rewrite the notebook to compute cross-bank / has-charges features from a pacs join it does itself; (c) mark notebook blocked on post-MVP Gold enrichment. |

#### Enum / reference sweep

| File | Change |
|---|---|
| `datalakehouse-api/lakehouse_query_api.py` | `transactions_gold_path` → `transaction_gold_path`; `GOLD_PATHS` key `"transactions"` → `"transaction"`. Coordinate with API consumers (URL query param `?table=transactions` breaks). |

### Frontend Code Impact

No direct frontend changes. The CMS UI relies on the backend service layer; if the CMS backend keeps returning the same shape (via the pacs join), the UI is unaffected. Test fixtures scope: CMS end-to-end tests that assert `debtor_name`/`creditor_name` populate for pacs.008 payment scenarios need to be run against post-#110 view builders to confirm the join produces expected values.

---

## Side Effects and Risks

### Risks of Track A Alone

| Risk | Likelihood | Mitigation |
|---|---|---|
| Track A merged before #110 → all path references break immediately | High | Ship in the same PR family as #110 |
| Track A merged after #110 but before Track B → views build with `transaction` bronze data (via the tightened candidate lists from #108 Track A) but 🟠 consumers still read pacs-only columns and fail | Medium | Do not split Track A from Track B in the release |

### Risks of Track B

| Risk | Likelihood | Mitigation |
|---|---|---|
| View-layer pacs join miscomposed (missing tenant match, wrong direction) → data leakage or wrong rows | Medium | Composite join key `(end_to_end_id, tenant_id)` mandatory in the SQL; peer review + unit test |
| Anomaly Detection notebook silent breakage (option a chosen but view builder missing the fields) | Medium | Unit test on the view; smoke test the notebook end-to-end before merge |
| CMS UI shows blank `debtor_name` for legitimate pacs.008 rows because view join misses | Medium | Unit test the view join with a synthetic pacs.008 row and matching `gold/pacs008` row |
| DLH Query API URL change breaks external consumers | Medium | Deprecation warning; keep `"transactions"` alias for one release |
| Live JupyterHub / repo notebook drift causes deploy mismatch | Low | Reconcile out-of-band; snapshot in [repos/JupyterNotebooks/](../../../repos/JupyterNotebooks/) helps |

### Cross-Issue Dependencies

- **#110**: hard prerequisite. #111 assumes `gold/transaction` (singular) exists with the MVP schema.
- **#108 Track A**: view-builder candidate-list hardening must land with #111 to prevent silent alias picks after the rename.
- **#109**: independent; recommended to ship first as small NiFi cleanup.

---

## Effort Estimate

| Track | Files | Effort |
|---|---|---|
| A (path renames only) | 7 | 1–2 dev-hours |
| B (full sweep + view rewrites + CMS test + notebook decision) | 12–14 across biar and CMS | 4–6 dev-days |

Total for the #110 + #111 delivery bundle: ~2 dev-weeks + review + UAT (matches #110's own estimate).

---

## Acceptance Criteria (Verification Checklist)

### Track A

- [ ] `ConditionsTimelineView.py`, `alert_navigator.py`, `views_orchestrator.py`, both notebooks, and `lakehouse_query_api.py` reference singular paths only.
- [ ] `GOLD_PATHS` includes key `"transaction"`; may retain `"transactions"` as a temporary alias.

### Track B

- [ ] `transaction_detail_view.py`, `transaction_history_view.py`, `network_navigator_view.py` all read `gold/transaction`.
- [ ] `transaction_detail_view.py` and `transaction_history_view.py` include the composite-key `LEFT JOIN gold/pacs008 ON (end_to_end_id, tenant_id)`.
- [ ] View builders' candidate lists tightened to bind to explicit `transaction` column (see #108).
- [ ] `vw_transaction_detail` provides `debtor_name`, `creditor_name`, `xchg_rate`, `intr_bk_sttlm_amt` (via pacs join).
- [ ] CMS Transaction Detail returns non-null names for pacs.008-based rows.
- [ ] Anomaly Detection notebook decision documented: (a) pacs-join path chosen, or (b) notebook blocked on post-MVP.
- [ ] `lakehouse_query_api.py` API URL contract clarified with consumers.

---

## Recommended Sequencing

1. Ship #110 + #111 (Tracks A + B) as one atomic PR family.
2. Include #108 Track A hardening in the same PR family.
3. Reconcile JupyterHub live↔repo notebook drift as a separate follow-up.
4. Post-MVP: implement #110 Design Requirement 6 (Gold enrichment). Retire per-consumer pacs joins.
