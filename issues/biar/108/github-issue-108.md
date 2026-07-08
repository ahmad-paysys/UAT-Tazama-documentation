# GitHub Issue Comment — Issue #108

## User Story

**Title:** Fold `transaction_data` naming collision fix into the #110 rewrite (hardening AC only)

**Body:**
The naming collision documented here is a symptom of the wrong-source problem in #110, not an independent bug. Once `TransactionsETL` is rewritten to source from `event_history.transaction` (#110), the `transactionData` column is deleted, not renamed, and all five collision vectors collapse as a natural consequence. The only standalone action needed is to add a hardening acceptance criterion to the #110 PR: the view-builders' candidate detection list currently includes `"transaction"` as a fallback, which will silently pick up the new payload column and parse TMS-normalised JSON as ISO 20022 (producing all-null rows with no error) unless replaced with an explicit named-column binding.

**Scope:**
- Files: `transaction_detail_view.py`, `transaction_history_view.py`, `network_navigator_view.py` (all changes atomic with #110)
- Change: Replace candidate-list fallback with explicit `expected = "transaction"` check that raises on drift
- Migration required: No
- Frontend changes: No

**Acceptance Criteria:**
- [ ] `#110` PR removes the `"transaction"` fallback from the candidate lists in all three view builders.
- [ ] Candidate resolution raises `ValueError` on unknown columns rather than silently aliasing.
- [ ] `data-lineage.md` no longer describes `transactionData` as a bronze column on the transactions Hudi table.
- [ ] Grep of `transactionData` across `automation-orchestrator/`, `datalakehouse-api/`, and `JupyterHub/` returns zero hits post-#110.

**Testing:**
- Grep the repo post-#110 for `transactionData` / `"transaction_data"` fallbacks to confirm zero occurrences outside the CMS Prisma schema.
- Unit-test that `_resolve_json_column` raises on a bronze DataFrame missing the `transaction` column.
- Deploy #110 with Track A and confirm view builders bind to `transaction` explicitly; TMS-normalised fields populate correctly.
- Sanity: live NiFi at `http://10.10.80.19:8088` currently does not have a `Transaction_Data` processor deployed, so there is no live crash to reproduce from this issue directly (see #109 for the related concern).
