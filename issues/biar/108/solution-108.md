# Complete Fix — Exact Code Changes

Issue #108 has **no standalone code fix**. The naming collision dissolves as a natural consequence of #110. This solution file documents the two hardening ACs that must be folded into the #110 view-builder rewrite so that the `"transaction"` fallback in the candidate list does not become a silent-null trap after #110 lands.

Track A (below) can be merged in isolation — but it should be merged **inside** the #110 PR, not before it, because it depends on the bronze schema decisions #110 will make.

---

### Track A — View-builder candidate-list hardening (folded into #110's PR)

#### File: `automation-orchestrator/Table_ETLs/transaction_detail_view.py`

**BEFORE** ([lines 53-69](../../../repos/biar/automation-orchestrator/Table_ETLs/transaction_detail_view.py)):

```python
def _resolve_json_column(self, df: DataFrame) -> DataFrame:
    """Auto-detect and normalize the raw JSON payload column."""
    candidates = [
        "transactionData",
        "transaction_data",
        "transaction",
        "payload",
        "raw_payload",
        "raw_json",
        "transaction_json",
    ]
    json_col = next((c for c in candidates if c in df.columns), None)
    if json_col is None:
        raise ValueError(
            f"No raw JSON column found in bronze/transactions. "
            f"Tried: {candidates}\nAvailable: {df.columns}"
        )
    print(f"[TransactionDetailViewETL] Using JSON column: {json_col} → transaction_data")
    return df.withColumn("transaction_data", F.col(json_col).cast("string"))
```

**AFTER** (post-#110, requires an explicit expected column):

```python
def _resolve_json_column(self, df: DataFrame) -> DataFrame:
    """
    Resolve the TMS-normalised transaction payload column.

    Post-#110 the source is always bronze/transaction and the column is 'transaction'
    (JSONB from event_history.transaction). No pacs fallback; fail loudly on drift.
    """
    expected = "transaction"
    if expected not in df.columns:
        raise ValueError(
            f"[TransactionDetailViewETL] expected column '{expected}' not found in bronze/transaction. "
            f"Available: {df.columns}"
        )
    return df.withColumn("transaction_data", F.col(expected).cast("string"))
```

Removes the ambiguous multi-column fallback. If the bronze schema drifts the ETL fails loudly rather than silently aliasing the wrong column.

#### File: `automation-orchestrator/Table_ETLs/transaction_history_view.py`

Same treatment as `transaction_detail_view.py`. Replace the candidate list ([lines 55-66](../../../repos/biar/automation-orchestrator/Table_ETLs/transaction_history_view.py)) with an explicit `expected = "transaction"` check.

#### File: `automation-orchestrator/Table_ETLs/network_navigator_view.py`

Same treatment for [lines 68-81](../../../repos/biar/automation-orchestrator/Table_ETLs/network_navigator_view.py). Replace the candidate list with an explicit `expected = "transaction"` check.

**Impact of Track A alone:** if merged before #110's ETL rewrite lands, these three view builders will start raising errors on every run (because bronze/transactions still exposes `transactionData`, not `transaction`). That is why Track A must be merged **inside** the #110 PR, atomically with the ETL and schema changes — not before.

---

### Track B — Subsumed by #110

There is no separate Track B for #108. The full remediation is #110 in its entirety. See [issues/biar/110/solution-110.md](../110/solution-110.md).

---

### Test cases

#### Unit tests to add

- `test_resolve_json_column_raises_on_missing_expected`: assert `_resolve_json_column` on a `DataFrame` without a `transaction` column raises `ValueError`.
- `test_resolve_json_column_returns_alias_when_present`: assert the returned DataFrame has a `transaction_data` column and the original `transaction` column preserved.

#### Integration / E2E scenarios

1. Deploy #110 without Track A. Verify view builders still work (silent alias picks up `transaction` — but returns nulls, since the JSON is TMS-normalised not ISO 20022). **This is the bug Track A prevents.**
2. Deploy #110 with Track A. Verify view builders explicitly bind to `transaction` and downstream views expose TMS-normalised fields.
3. Rename the bronze column temporarily to `transaction_payload` and verify the ETL raises loudly instead of silently producing nulls.

#### Data migration validation SQL

Not applicable — no schema migration in #108.

#### Manual / UAT checks

| # | Scenario | Steps | Expected Result |
|---|---|---|---|
| 1 | Grep post-#110 for `transactionData` | `grep -rn transactionData repos/biar/automation-orchestrator/ repos/biar/datalakehouse-api/ repos/biar/JupyterHub/` | Zero hits (all references replaced) |
| 2 | Grep post-#110 for `transaction_data` alias | `grep -rn "transaction_data" repos/biar/automation-orchestrator/Table_ETLs/*_view.py` | Only as the aliased column name inside the view; not as a candidate list |

---

### Overall Impact of the Fix

| Area | Before | After |
|---|---|---|
| Meaning of `transactionData` in the Lakehouse | Pacs payload (mis-named) | Column deleted (via #110) |
| Meaning of `transaction_data` alias in view builders | Fallback to any of 5–7 candidate columns | Explicit binding to `transaction` |
| Failure mode when bronze schema drifts | Silent alias to unrelated column → all nulls | Loud `ValueError` |
| Dev mental model | Ambiguous; 5 collision vectors | Unambiguous; only CMS Prisma retains the name (unambiguously CMS-scoped) |

---

### Fix Summary

Nothing about #108 is a bug you can fix in isolation. The name `transactionData` was chosen for a Hudi column that holds raw pacs message strings; the choice was wrong because the column shouldn't exist in the first place (pacs bronze is the wrong source for `gold/transactions` — that's issue #110). Once #110 rewrites the pipeline to read from `event_history.transaction`, the column is not renamed but deleted, and the name becomes free of ambiguity.

The one live risk during the #110 migration is the view-builder candidate list. It contains `"transaction"` as a fallback. If the new bronze payload column ends up called `transaction` (which is the natural choice given the ODS uses that name), all three view builders will silently pick it up and parse it as ISO 20022 pacs JSON — producing all-null fields. Track A here forces each view builder to bind to an explicit column name and raise loudly on drift.

There is no Track B specific to #108. Track B is #110.
