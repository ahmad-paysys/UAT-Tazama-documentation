# Sandy's Findings — Consolidated Review

**Reviewer:** Ahmad Khalid
**Report Date:** 2026-07-14
**Repository:** tazama-lf/biar (branch `dev`, HEAD `4bf7dbb`)
**Issues covered:** #80, #95, #109, #110, #111, #112, #116

---

## Executive Summary — State of Things Overall

Sandy has run a code-level re-audit of the seven BIAR issues that were closed or claimed fixed. The picture is:

- **Structural fixes have largely landed** — the transactions ETL rewrite (#110), its downstream consumer sweep (#111), the removal of the crash-prone `transaction_data` NiFi processor (#109), the new CMS-side ingest for the case-management rework (#116), and the missing `cms_usernames` NiFi ingest (#112 primary bug) are all code-confirmed as done.
- **Two secondary code bugs are still open in code**:
  - #112 secondary — `cms_usernames.py` Hudi `record_key="id"` (SERIAL) instead of `user_id`. Duplicate-user risk on any NiFi state reset.
  - #109 residual — `DynamicETL._resolve_config()` still only supports `raw_history` / `event_history` / `enrichment`. The crash is latent, not eliminated — any future rogue processor with a `tazama_cms` `db_name` will crash the orchestrator batch.
- **Two design-requirement deviations from #110** are still uncorrected in code:
  - Path naming remained plural (`bronze/silver/gold transactions`) even though the design specified singular. System is internally coherent, but diverges from the logged design.
  - Party-entity join (Requirement 7) — `debtor_entity_id` / `creditor_entity_id` are not produced in the gold-tier select. There is no entity join.
- **One consumer residue from #111** — `transaction_history_view.py` still tries **pacs ISO paths first** in its coalesce chains before falling back to the new flat `transaction` JSON. Not a functional bug (coalesce shields it) but dead code against the new source.
- **Data-lineage gaps in the query API (#111 addendum → linked to #115)** — `GOLD_PATHS` in `lakehouse_query_api.py` has no `gold/pacs002`, no `gold/account`, and its `typologies` key still points to **bronze**. Any consumer going through the query API cannot do pacs joins.
- **Live dashboards are broken (#80, #95)** — Sandy attached screenshots of runtime errors on TMS Performance, Executive Overview, Case Management Trend, and Fraud Trend Analysis dashboards. The likely blocker is the Case Management notebook's attempt to `load_tenant_hudi("gold/cms_usernames")` — that table has no `tenant_id` / `tx_tenant_id` / `tenantid` column, so the tenant-filter helper raises before the notebook can execute anything else. The dashboards' schema-mapping fallbacks otherwise look correct.
- **#112 also raised a policy question** (still open with Justus): is the notebook's use of *current* Keycloak display names for *historical* investigator attribution acceptable, or does this need a versioning approach in `cms_usernames`?

**Bottom line:** the closure of these seven issues is real, but three separate residuals need scheduling: (a) a small `cms_usernames` Hudi key fix, (b) a live-dashboard debugging pass focused on the tenant-filter blocker in the Case Management notebook, and (c) two deferred design items from #110 (singular paths, entity join). #109 and #111 residuals are low-priority hardening.

---

## Issue-by-Issue Detail — Validated Against Code

### #80 — TMS Performance Dashboard metrics incorrectly calculated — ◐ PARTIALLY FIXED

**Sandy's outstanding claim:** original bucketing / double-counting / inverted formula issues are fixed in code, but the **dashboard is currently failing on a runtime error** (screenshot attached to the issue; error text not captured).

**Code validation (branch `dev`):**

| Sandy's fix claim | State in code | Evidence |
|---|---|---|
| Bucketing by `event_ts` (not `ingested_at_ts`) | ✅ Confirmed | `JupyterHub/notebooks/TMS_Performance_Dashboard.ipynb` lines 433–434 comment; grouping uses `event_ts` |
| `countDistinct("end_to_end_id")` for received | ✅ Confirmed | line 533 |
| `countDistinct("tx_msg_id")` for evaluated | ✅ Confirmed | line 543 |
| `evaluation_gold_path = gold/evaluation` (not `gold/alerts`) | ✅ Confirmed | line 124 (defined), line 207 (loaded) |
| Correct received/evaluated ratio | ✅ Confirmed | line 813 |
| Latency from evaluation `dc_to_report_ms` | ✅ Confirmed | lines 494, 515 |

**Outstanding runtime error — likely root cause (unverified against runtime, but code-confirmed as a blocker if triggered):**

- TMS_Performance_Dashboard loads `gold/transactions` via `load_tenant_hudi()` at [notebook line 116](../../repos/biar/JupyterHub/notebooks/TMS_Performance_Dashboard.ipynb).
- `load_tenant_hudi` checks for one of `("tenant_id", "tx_tenant_id", "tenantid")` on the loaded frame ([line 103](../../repos/biar/JupyterHub/notebooks/TMS_Performance_Dashboard.ipynb)). If none is present, it raises `ValueError`.
- `TransactionsETL.py` gold-tier select (lines 167–186) does output `tenantid` — so this specific load should succeed. The dashboard error is therefore **not** a tenant-column mismatch. A live-log capture of the traceback is required to pin it down.
- Cross-check with `TransactionsETL.py` confirms the output schema (17 columns: `transaction_id`, `amt`, `ccy`, `credttm`, `destination`, `endtoendid`, `msgid`, `source`, `tenantid`, `txsts`, `txtp`, `event_ts`, `event_date`, `created_at_epoch_ms`, `ingested_at_ts`, `event_to_ingest_ms`, `source_file_path`, `record_hash`). All column-name fallbacks (`normalize_transactions_for_dashboard`) in the notebook match — no schema drift.

**Recommendation:** obtain the actual Python traceback from the JupyterHub run — the code-side fix claims all check out.

---

### #95 — Multi-tenancy interim fix (per-notebook tenantId filter) — ◐ DASHBOARDS FAILING

**Sandy's outstanding claims:**
1. Executive Overview Dashboard shows a `tenantId` error and "transactions table (was it renamed `transaction`?)".
2. Case Management Trend Dashboard has errors — same error appearing on TMS Performance and Fraud Trend Analysis dashboards.

**Code validation (branch `dev`):**

| Claim | State | Evidence |
|---|---|---|
| Was `transactions` renamed to `transaction`? | ❌ No — plural retained everywhere | `TransactionsETL.py` lines 30/34/38 define `bronze/silver/gold transactions`; all four dashboards read `gold/transactions` (Executive [line 145](../../repos/biar/JupyterHub/notebooks/Executive_Overview_Dashboard.ipynb), Case Mgmt [line 138](../../repos/biar/JupyterHub/notebooks/Case_Management_Trend_Dashboard.ipynb), TMS [line 116](../../repos/biar/JupyterHub/notebooks/TMS_Performance_Dashboard.ipynb), Fraud [line 108](../../repos/biar/JupyterHub/notebooks/Fraud_Trend_Analysis_Dashboard.ipynb)) |
| Tenant filter helper `load_tenant_hudi` consistently defined | ✅ Yes | Each notebook defines `TENANT_FILTER_COLUMNS = ("tenant_id", "tx_tenant_id", "tenantid")` and applies the filter; TMS line 103, Exec line 103, Case Mgmt line 88, Fraud line 2 |

**Likely root cause of the shared error across dashboards:**

- **Case Management Trend Dashboard** loads `gold/cms_usernames` via `load_tenant_hudi()` at [line 127](../../repos/biar/JupyterHub/notebooks/Case_Management_Trend_Dashboard.ipynb).
- `automation-orchestrator/Table_ETLs/cms_usernames.py` writes a straight pass-through of the source table's five columns — **no tenant column is added** (verified lines 30–51).
- Result: `load_tenant_hudi("gold/cms_usernames")` will raise `ValueError` at line 69–72 because none of `tenant_id` / `tx_tenant_id` / `tenantid` is present. This matches the reported error signature.
- **Redundant tenant filtering** in Case Mgmt Dashboard: cell 4 lines 3–5 re-applies `alerts.filter(alerts.tenant_id == 'TAZAMA')` after `load_tenant_hudi` already filtered — evidence of incomplete refactor.

**For TMS / Executive / Fraud dashboards**, the shared error is more likely a *different* runtime issue (possibly an empty `gold/pacs008` or `gold/evaluation`, or a widget input default). The tenant-filter helper itself operates on `tenantid` which is present in `gold/transactions`. A live traceback is needed to confirm.

**Executive Overview `tenantId` error specifically** — reads `gold/transactions` (tenant column `tenantid` present, filter should succeed). Likely a widget default or a downstream cell touching an empty table. Needs runtime capture.

**Recommendation:**
1. Fix `cms_usernames.py` to propagate `tenant_id` from source (also fixes the notebook blocker).
2. Capture actual tracebacks from each failing dashboard before further diagnosis on TMS/Exec/Fraud.

---

### #109 — DynamicETL crashes on `tazama_cms.transaction_data` — ✅ RESOLVED VIA REMOVAL, RESIDUAL LATENT RISK

**Sandy's outstanding claim:** the fix chosen was to **remove** the `transaction_data` QueryDatabaseTableRecord processor from `nifi/tazama.xml`, so the crash path is unreachable from the template. But `DynamicETL._resolve_config()` still supports only three `db_name` values — the latent risk of a rogue processor persists. Runtime check of the deployed canvas still recommended.

**Code validation (branch `dev`):**

| Claim | State | Evidence |
|---|---|---|
| `transaction_data` processor absent from `nifi/tazama.xml` | ✅ Confirmed | grep for `<value>transaction_data</value>` returns 0 matches |
| `DynamicETL._resolve_config` still crashes on non-supported `db_name` | ✅ Confirmed | `automation-orchestrator/Table_ETLs/DynamicETL.py` lines 54–80: only `raw_history`, `event_history`, `enrichment` accepted; line 75 raises `RuntimeError("Unidentified db …")` |

**Outstanding — non-blocking:**
- Runtime verification of the deployed NiFi canvas (older canvas may still contain the removed processor).
- DynamicETL hardening: either fall through gracefully for `tazama_cms.*` batches (delegate to `_ETL_REGISTRY`), or fail earlier with a clearer message. Low priority.

---

### #110 — TransactionsETL rewritten to consume from `event_history.transaction` — ✅ MAIN FIX DONE, 2 DESIGN DEVIATIONS OUTSTANDING

**Sandy's outstanding claims:**
1. **Path naming** — design specified singular `bronze/silver/gold transaction`; code retained plural. Internally coherent, but diverges from the logged design.
2. **No party-entity join** — Requirement 7 (`debtor_entity_id` / `creditor_entity_id` via an entity join) is not implemented; the gold select only carries flat event-history fields.

**Code validation (branch `dev`):**

| Claim | State | Evidence |
|---|---|---|
| ETL reads `event_history.transaction` JSON, not payment message tables | ✅ Confirmed | `TransactionsETL.py` docstring: "This ETL intentionally does not read or derive anything from payment message tables." |
| `transaction_id` = `TxTp || EndToEndId` concatenation | ✅ Confirmed | in gold select |
| `CombinedPacs.py` deleted | ✅ Confirmed | grep in `Table_ETLs/` shows no such file |
| Orchestrator skip guard removed | ✅ Confirmed | grep in `lakehouse_automation_pipeline.py` for `Skipped: transactions|CombinedPacs|pipeline_state` returns 0 matches |
| NiFi `transaction` processor re-enabled (watermark `credttm`) | ✅ Confirmed | present in `nifi/tazama.xml` |
| **Paths remain plural** (`bronze/silver/gold transactions`) | ⚠️ Confirmed as deviation from design | `TransactionsETL.py` lines 30, 34, 38 — plural |
| **No entity join** — no `debtor_entity_id` / `creditor_entity_id` in gold | ⚠️ Confirmed as deviation from design | `TransactionsETL.py` lines 167–186 output the 17-column flat schema listed under #80 above; no join clause anywhere in the ETL |

**Outstanding:** two design-requirement deviations. Neither is a runtime bug. Need explicit user/design decision on whether to enforce these or amend the design.

---

### #111 — Downstream consumers of the transactions redesign — ◐ SUBSTANTIALLY FIXED, 3 RESIDUALS

**Sandy's outstanding claims:**
1. `transaction_history_view.py` still uses **pacs ISO paths as the primary** extraction attempts, with regex fallbacks only after. Harmless via coalesce, but dead code against the new source.
2. `lakehouse_query_api.py` `GOLD_PATHS` gaps: no `gold/pacs002`, no `gold/account`, and `typologies` key still points to **bronze**. Any consumer routing through the query API cannot do pacs joins. (Cross-linked to #115.)
3. Anomaly Detection notebook has a bigger blocker than the MVP pacs-field gap — `tazama_cms.alerts` has no `updated_at` column; NiFi watermarks on `created_at` so alerts are extracted before `prediction_outcome` is set. All outcome labels frozen at default `TRUE_POSITIVE`. (Cross-linked to #113.)

**Code validation (branch `dev`):**

| Consumer | State | Evidence |
|---|---|---|
| `alert_navigator.py` — new-column remaps | ✅ Confirmed | line 569 `F.col("txsts").cast("string").alias("tx_status")`; line 574 `F.col("endtoendid").cast("string").alias("end_to_end_id")` |
| `transaction_history_view.py` — legacy PACS-first coalesce | ⚠️ Confirmed as residue | lines 201–209 for `tx_msg_id`: PACS ISO paths (FIToFICstmrCdtTrf / FIToFIPmtSts / CstmrCdtTrfInitn) attempted **first** at lines 202–205; `DataCache.MsgId` second (206); regex fallbacks last (207–208) |
| `ConditionsTimelineView.py` reads `gold/transactions`, buckets by event_ts | ✅ Confirmed | line 98 (load), line 216 (`tx_event_ts` bucketing) |
| `views_orchestrator.py` registers view builders | ✅ Confirmed (Sandy said 3, actually **6** are registered) | Registered at lines 66, 72, 73, 86, 100, 113: `AlertNavigatorETL`, `TransactionDetailViewETL`, `TransactionHistoryViewETL`, `NetworkNavigatorViewETL`, `AlertHistoryViewETL`, `ConditionsTimelineViewETL` |
| `lakehouse_query_api.py` GOLD_PATHS: no `gold/pacs002` | ⚠️ Confirmed | `datalakehouse-api/lakehouse_query_api.py` lines 165–192 — no `pacs002` entry |
| GOLD_PATHS: no `gold/account` | ⚠️ Confirmed | only `gold/account_holder` (line 144), not in GOLD_PATHS map |
| GOLD_PATHS: `typologies` still bronze | ⚠️ Confirmed | line 173 uses `typologies_bronze_path` (line 150) which resolves to `bronze/typologies` |
| GOLD_PATHS: `transactions_gold_path` remains plural | ✅ Confirmed | line 139 `gold/transactions` — consistent with ETL |

**Outstanding:**
1. `transaction_history_view.py` cleanup — reorder coalesce so the new flat `transaction` JSON is tried first, or remove the dead pacs-first paths. Low priority, cosmetic.
2. `lakehouse_query_api.py` `GOLD_PATHS` completeness — add `gold/pacs002`, `gold/account`, and move `typologies` to gold. This *is* a blocker for API-based consumers (see #115).
3. Anomaly Detection / `tazama_cms.alerts` watermark issue — separate ticket (#113).

---

### #112 — `gold/cms_usernames` empty; missing NiFi processor — ✅ PRIMARY FIXED, ⚠️ SECONDARY OPEN

**Sandy's outstanding claims:**
1. Primary fix landed — NiFi processor now exists with `max-value = updated_at`.
2. **Secondary bug NOT fixed** — `cms_usernames.py` lines 58–60 still write Hudi with `record_key="id"` (SERIAL surrogate). Should be `user_id`. On any NiFi state reset, this produces duplicate user rows in gold.
3. Silent accuracy problem: notebook uses *current* Keycloak display name for historical case attribution — needs an explicit design decision.

**Code validation (branch `dev`):**

| Claim | State | Evidence |
|---|---|---|
| NiFi processor exists, watermark = `updated_at` | ✅ Confirmed | `nifi/tazama.xml` line 16431 (processor), line 16448 (max-value column `updated_at`) |
| `record_key="id"` (should be `user_id`) | ⚠️ Confirmed as open bug | `automation-orchestrator/Table_ETLs/cms_usernames.py` line 59 (record_key), line 60 (precombine `created_at`) |
| Notebook joins on current name only | ⚠️ Confirmed | `Case_Management_Trend_Dashboard.ipynb` loads `gold/cms_usernames` at line 127 and joins against `user_id` for display name resolution; a comment in the notebook itself acknowledges "A given user_id can appear more than once in cms_usernames (e.g. role…)" — awareness but no historical-versioning approach |
| ETL passes through no `tenant_id` — makes `load_tenant_hudi("gold/cms_usernames")` raise | ⚠️ Confirmed | `cms_usernames.py` lines 30–51 write only the source table's 5 columns; no tenant column added. This is the likely blocker behind the #95 dashboard error |

**Outstanding:**
1. Fix `record_key` = `user_id` (matches the `UNIQUE (user_id)` constraint in Postgres).
2. Propagate a tenant column into `gold/cms_usernames` (unblocks Case Management dashboard).
3. Design decision from Justus on historical name attribution.

---

### #116 — New CMS-schema ETLs and NiFi processors — ✅ COMPLETE

**Sandy's outstanding claim:** all five ETLs registered, all five NiFi processors present with correct max-value columns. Awaiting a retest against the live CMS schema.

**Code validation (branch `dev`):**

| ETL file | Exists | Registered in `_ETL_REGISTRY` | NiFi processor + max-value |
|---|---|---|---|
| `CasePriorityThresholdsETL.py` | ✅ | ✅ (`lakehouse_automation_pipeline.py` lines 26–30, 83–92) | ✅ `case_priority_thresholds` / `updated_at` (nifi/tazama.xml line 22801) |
| `SlaPoliciesETL.py` | ✅ | ✅ | ✅ `sla_policies` / `updated_at` (line 16674) |
| `SlaEscalationThresholdsETL.py` | ✅ | ✅ | ✅ `sla_escalation_thresholds` / `updated_at` (line 25238) |
| `SlaEscalationRecordsETL.py` | ✅ | ✅ | ✅ `sla_escalation_records` / `notified_at` (line 26629) |
| `InvestigationGroupsETL.py` | ✅ | ✅ | ✅ `investigation_groups` / `created_at` (line 45660) |

**Outstanding:** functional retest against the live CMS #214/#220 schema — nothing left in code.

---

## Consolidated Outstanding Work

Ordered by urgency:

| # | Item | Issue | File(s) | Effort |
|---|---|---|---|---|
| 1 | **Add tenant column to `gold/cms_usernames`** — unblocks Case Management Trend Dashboard runtime error | #95, #112 | `automation-orchestrator/Table_ETLs/cms_usernames.py` | Small |
| 2 | Fix `cms_usernames` Hudi `record_key` from `id` → `user_id` | #112 | same file (lines 58–60) | Trivial |
| 3 | Capture live tracebacks from TMS Performance, Executive Overview, Fraud Trend Analysis dashboards to root-cause the remaining runtime failures | #80, #95 | JupyterHub notebooks | Investigation |
| 4 | Extend `GOLD_PATHS` in `lakehouse_query_api.py` to include `gold/pacs002`, `gold/account`; move `typologies` to gold | #111 / #115 | `datalakehouse-api/lakehouse_query_api.py` | Small |
| 5 | Design decision (Justus): historical investigator name attribution | #112 | design | Discussion |
| 6 | Design decision: enforce singular paths for transactions, or accept plural as final | #110 | design | Discussion |
| 7 | Implement party-entity join for `debtor_entity_id` / `creditor_entity_id` (Requirement 7) | #110 | `TransactionsETL.py` | Medium (needs entity source) |
| 8 | `transaction_history_view.py` — reorder coalesce to try new flat `transaction` JSON first | #111 | `automation-orchestrator/Table_ETLs/transaction_history_view.py` lines 201–209 | Trivial |
| 9 | Harden `DynamicETL._resolve_config` — fail-fast or delegate on unsupported `db_name` | #109 | `automation-orchestrator/Table_ETLs/DynamicETL.py` lines 54–80 | Trivial |
| 10 | Verify live NiFi canvas matches template (no stale `transaction_data` processor) | #109 | live NiFi | Ops check |

---

## Notes on Verification Method

- All code citations verified on `origin/dev` at commit `4bf7dbb` on 2026-07-14.
- Runtime errors on dashboards were not reproduced locally — recommendations that depend on tracebacks are called out explicitly.
- Where Sandy's line numbers matched observed code, they are quoted verbatim; where they differed (e.g. views_orchestrator "3 view builders" but 6 actually registered), the code state is preferred.
