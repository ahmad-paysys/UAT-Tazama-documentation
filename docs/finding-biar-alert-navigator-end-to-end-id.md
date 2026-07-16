# Finding: BIAR removed `end_to_end_id` from `alert_navigator_header` (2026-07-10)

**Filed:** 2026-07-16
**Context:** While diagnosing CMS issue [#248](https://github.com/tazama-lf/case-management-system/issues/248) (Alert Navigator 500 error), traced the missing `anh.end_to_end_id` column back through BIAR history.

## Summary

The `alert_navigator_header` Hudi view previously exposed an `end_to_end_id` column. It was intentionally removed on **2026-07-10** in a BIAR commit that reworked the pacs002→pacs008 bridge join in the ETL. This is the upstream reason CMS's Alert Navigator query started failing.

CMS never actually consumed the value — the field was selected and grouped by in the query but not read into the response object — so the CMS-side fix is to delete the reference. **No BIAR action needed for CMS #248.** This document exists purely so the history isn't lost.

## Timeline (from `repos/biar/automation-orchestrator`, file `Table_ETLs/alert_navigator.py`)

| Commit | Date | Author | `alert_navigator_header` includes `end_to_end_id`? |
|---|---|---|---|
| `8226620` (initial split of monolith) | — | — | No |
| `f9576c3`, `56fac20` | — | — | No |
| **`035b8db`** | **2026-05-11** | Hassan Rizwan | **Added.** `_build_header()` gained a pacs002→pacs008 bridge join (`g_tx_lookup`) that carried `end_to_end_id` into the header output. |
| `d2cc03d`, `c6731af` | — | — | Still present |
| **`4d00a0a`** | **2026-07-10** | Hassan Rizwan | **Removed.** Commit message: "fixed parsing of datatype for tenant id column in tasks table from long to string, fixed alert navigator transaction status source for cms visualisation". Bridge rewritten: fields renamed to `msg_end_to_end_id` / `lookup_end_to_end_id` and both explicitly dropped before write. |
| HEAD (present) | 2026-07-16 | — | No |

Branches carrying the removal: `origin/dev`, `origin/feat-paysys-nifi`, `origin/fixes`, `origin/paysys/missingTenantId`.

## Why the removal happened

The `4d00a0a` change was a legitimate fix — the previous version used an *inner* join through the pacs002→pacs008 bridge, meaning any alert whose pacs002 lacked a matching pacs008 would drop out of the header entirely. The rewrite replaces that with an outer join and a coalesce over multiple lookup keys, so alerts without matching payment enrichment still surface with the correct `transaction_status`. Dropping the `end_to_end_id` output column was ETL housekeeping bundled into that fix.

Nothing in the CMS response depended on the column (see #248), so the removal was safe from BIAR's perspective. It just broke a CMS query that had been silently selecting a dead column.

## Why CMS was still selecting it

The CMS SQL had been written in April 2026 (commit `992bd5a2` on CMS) when the query was rewritten to target `alert_navigator_header`. That rewrite kept `anh.end_to_end_id` in the SELECT/GROUP BY as a copy-paste artifact from the pre-refactor query (which read from `transactions`, where the column exists). The response-mapping code in the same commit dropped the field, so the column selection became dead code — but the SQL still parsed and executed as long as BIAR happened to expose the column.

Between May 11 and July 10, BIAR did expose it, so the CMS query worked by accident. After July 10, the accident stopped working.

## Architectural note

`end_to_end_id` is a transaction-side identifier (ISO 20022 `EndToEndId` on pacs.008 / pain.001). It doesn't semantically belong on an *alert* header — the header describes the alert, and payment identifiers should be looked up through the alert's transaction reference (`transaction_id`, `tx_msg_id`) against the `transactions` / `transaction_detail` views. The current BIAR shape is cleaner. If Alert Navigator ever needs to display end-to-end ID, the right approach is to add a client-side lookup or a backend join against `transaction_detail`, not to re-add the column to the header view.

## Cross-references

- CMS issue: https://github.com/tazama-lf/case-management-system/issues/248
- CMS file with the dead reference: `solutions/case-management-system/backend/src/modules/gold-lakehouse/alerts-lakehouse.service.ts` (lines ~87 and ~140)
- BIAR file: `repos/biar/automation-orchestrator/Table_ETLs/alert_navigator.py`, `_build_header()`
- BIAR removal commit: `4d00a0a` (2026-07-10, Hassan Rizwan)
- BIAR addition commit: `035b8db` (2026-05-11, Hassan Rizwan)
