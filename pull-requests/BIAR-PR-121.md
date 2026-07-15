# PR Review: BIAR #121 — fix: corrected small bug in anomaly notebook regarding event history transactions lineage downstream effect

**Repo:** tazama-lf/biar
**Branch:** `fixes` → `dev`
**Author:** MuneebPaysys (commits by hassanrizwan-paysys and MAdeel95)
**Date Reviewed:** 2026-07-15
**Label:** bug
**Size:** +44 / -4 lines across 1 file
**Commits:** 3 (`f9bb3cd`, `84184af`, `720ff0a`)
**State:** OPEN
**Existing approvals:** none (CodeRabbit posted `COMMENTED` with 1 Major + 1 Nitpick; author addressed the Major and pushed lint follow-up)

## Table of Contents

- [Overview](#overview)
- [What Changed (Detailed)](#what-changed-detailed)
  - [1. Cell 8 — new helpers and imports](#1-cell-8--new-helpers-and-imports)
  - [2. Cell 9 — swap transactions load site](#2-cell-9--swap-transactions-load-site)
  - [3. Cell 10 — drop redundant F import](#3-cell-10--drop-redundant-f-import)
  - [4. Cell 16 — swap second transactions load site](#4-cell-16--swap-second-transactions-load-site)
- [Code Quality Analysis](#code-quality-analysis)
  - [Strengths](#strengths)
  - [Issues and Observations](#issues-and-observations)
- [Security Assessment](#security-assessment)
- [Test Coverage](#test-coverage)
- [CodeRabbit Activity](#coderabbit-activity)
- [Summary and Verdict](#summary-and-verdict)
- [GitHub Review Comment](#github-review-comment)

---

## Overview

Targeted fix in [JupyterHub/notebooks/Anomaly_Detection_And_Rule_Calibration.ipynb](repos/biar/JupyterHub/notebooks/Anomaly_Detection_And_Rule_Calibration.ipynb) to adapt the anomaly-detection and rule-calibration notebook to the new event-history-only `gold/transactions` lineage.

The gold transactions table now comes straight from the Ozone event-history payload and no longer carries PACS-derived enrichment fields (amount, currency, tx type, status, agent ids, charge count). The notebook depends on those features for calibration and clustering, so it now:

1. Prefers the enriched `views/vw_transaction_detail` (which still exposes the pre-existing feature set) via `load_anomaly_transactions()`.
2. Falls back to raw `gold/transactions` on any exception, printing the failure.
3. Uses a new `_transaction_col` helper for case-insensitive column resolution across three schemas (detail-view, legacy PACS-derived, Ozone) and a `normalize_transactions_for_anomaly(df)` projection to produce a stable output schema.

Targets `dev` — correct for this repo's flow. All required CI checks are green (Python Lint, TypeScript Build/Lint/Test, CodeQL, DCO, hadolint, njsscan, encoding, dependency-review, GPG verify, conventional commits). Docker Build Check is still `IN_PROGRESS` at fetch time — expected to pass, non-blocking. `mergeStateStatus` is `BLOCKED` because branch-protection requires an approval that hasn't landed yet.

| File | Nature of Change |
|------|-----------------|
| `JupyterHub/notebooks/Anomaly_Detection_And_Rule_Calibration.ipynb` | Add `_transaction_col`, `normalize_transactions_for_anomaly`, `load_anomaly_transactions`; swap two `load_tenant_hudi(transactions_path)` call sites to `load_anomaly_transactions()`; consolidate `from pyspark.sql import functions as F` at top of cell 8. |

[↑ Back to top](#pr-review-biar-121--fix-corrected-small-bug-in-anomaly-notebook-regarding-event-history-transactions-lineage-downstream-effect)

---

## What Changed (Detailed)

The changes span four cells of one notebook. Cell numbering below matches the `nbformat` cell index used in the review file (top-of-file = cell 0).

### 1. Cell 8 — new helpers and imports

Before, cell 8 declared path constants only. After, it also imports `functions as F` at the top of the cell and defines three helpers.

```python
# After (relevant additions)
from pyspark.sql.functions import avg, col, count, dayofweek, explode, expr, from_json, hour, stddev, sum, to_date, unix_timestamp, when
from pyspark.sql.types import ArrayType, BooleanType, IntegerType, StringType, StructField, StructType
from pyspark.sql import functions as F   # ← new, imported before any assignment

alerts_path       = f"{WAREHOUSE_ROOT}/bronze/alerts"
typology_path = f"{WAREHOUSE_ROOT}/gold/typologies"
rules_path = f"{WAREHOUSE_ROOT}/gold/rule"
transactions_path = f"{WAREHOUSE_ROOT}/gold/transactions"

transaction_detail_view_path = f"{WAREHOUSE_ROOT}/views/vw_transaction_detail"


def _transaction_col(df, preferred_name, fallback_names=(), cast_type="string"):
    """Return a transaction column across detail-view, old PACS-derived, and Ozone schemas."""
    df_cols_lower = {c.lower(): c for c in df.columns}
    for col_name in (preferred_name, *fallback_names):
        resolved_name = df_cols_lower.get(col_name.lower())
        if resolved_name:
            return F.col(resolved_name).cast(cast_type)
    return F.lit(None).cast(cast_type)


def normalize_transactions_for_anomaly(df):
    """Expose the transaction feature columns expected by this notebook."""
    return df.select(
        _transaction_col(df, "transaction_id").alias("transaction_id"),
        _transaction_col(df, "end_to_end_id", ("endtoendid",)).alias("end_to_end_id"),
        _transaction_col(df, "tenant_id", ("tx_tenant_id", "tenantid")).alias("tenant_id"),
        _transaction_col(df, "tx_msg_id", ("msgid",)).alias("tx_msg_id"),
        _transaction_col(df, "tx_type", ("txtp",)).alias("tx_type"),
        _transaction_col(df, "tx_status", ("txsts",)).alias("tx_status"),
        _transaction_col(df, "tx_amount", ("instructed_amount", "amt"), "double").alias("tx_amount"),
        _transaction_col(df, "tx_ccy", ("instructed_currency", "ccy")).alias("tx_ccy"),
        _transaction_col(df, "instg_mmb_id").alias("instg_mmb_id"),
        _transaction_col(df, "instd_mmb_id").alias("instd_mmb_id"),
        _transaction_col(df, "charge_count", cast_type="int").alias("charge_count"),
        _transaction_col(df, "event_ts", ("tx_event_ts",), "timestamp").alias("event_ts"),
        _transaction_col(df, "event_date", ("tx_event_date",), "date").alias("event_date"),
    )


def load_anomaly_transactions():
    """Prefer the enriched transaction detail view; fall back to raw Ozone transactions."""
    try:
        return normalize_transactions_for_anomaly(load_tenant_hudi(transaction_detail_view_path))
    except Exception as e:
        print(f"Transaction detail view unavailable; falling back to gold/transactions: {e}")
        return normalize_transactions_for_anomaly(load_tenant_hudi(transactions_path))
```

Notes:
- `_transaction_col` builds a lowercase → original mapping so it can resolve `endToEndId`, `TenantId`, etc. across the three source shapes. This form was reached after CodeRabbit's Major finding (see [CodeRabbit Activity](#coderabbit-activity)).
- `normalize_transactions_for_anomaly` always returns the same 13-column schema — nulls where the source has no match — so downstream cells don't need conditional column handling.
- `load_anomaly_transactions` catches any `Exception` raised by `load_tenant_hudi` (including the `ValueError` it throws when no tenant filter column is present) and prints the error before falling back. This is broad — see [Issue 1](#issue-1--broad-except-exception-in-load_anomaly_transactions) below.

### 2. Cell 9 — swap transactions load site

```diff
 rules = load_tenant_hudi(rules_path)

 # Read transactions
-transactions = load_tenant_hudi(transactions_path)
+transactions = load_anomaly_transactions()
```

### 3. Cell 10 — drop redundant `F` import

```diff
 from pyspark.sql.window import Window
-from pyspark.sql import functions as F

 typology_path = f"{WAREHOUSE_ROOT}/bronze/typologies"
```

`F` is now imported once at the top of cell 8. This was added by the final commit `720ff0a` in response to Ahmad's `E402` / `F811` ruff comment.

### 4. Cell 16 — swap second transactions load site

```diff
 # Load the authoritative Hudi table used by downstream metrics.
-df_tx = load_tenant_hudi(transactions_path)
+df_tx = load_anomaly_transactions()
```

Same substitution as cell 9 — the second call site (used later in the calibration/clustering section).

[↑ Back to top](#pr-review-biar-121--fix-corrected-small-bug-in-anomaly-notebook-regarding-event-history-transactions-lineage-downstream-effect)

---

## Code Quality Analysis

### Strengths

- **Correctly diagnoses the schema drift.** The gold transactions table lost PACS-derived enrichment fields; the fix routes through `vw_transaction_detail` where those fields remain, with a raw-table fallback. That is the right layered approach.
- **Stable output schema.** `normalize_transactions_for_anomaly` guarantees a 13-column result with nulls for missing sources, isolating downstream cells from the schema drift.
- **Case-insensitive column resolution.** After round 2, `_transaction_col` uses a lowercase-keyed dict to match `endToEndId` / `tenantId` from raw Ozone tables — no silent nulls due to casing.
- **Multi-schema fallbacks are explicit.** Every projection in `normalize_transactions_for_anomaly` lists its legacy PACS name and its Ozone name as fallbacks, so a reader can audit exactly what maps to what.
- **Scoped changes.** Two call-site swaps, no reordering or refactoring of downstream cells.
- **Ruff lint issues addressed in-branch.** Ahmad's E402/F811 comment was resolved in the final commit before I re-read.

### Issues and Observations

#### Issue 1 — Broad `except Exception` in `load_anomaly_transactions`

**Severity: Minor (Maintainability / Code Quality)**

```python
def load_anomaly_transactions():
    try:
        return normalize_transactions_for_anomaly(load_tenant_hudi(transaction_detail_view_path))
    except Exception as e:
        print(f"Transaction detail view unavailable; falling back to gold/transactions: {e}")
        return normalize_transactions_for_anomaly(load_tenant_hudi(transactions_path))
```

`except Exception` will swallow *any* failure of the primary path — including `KeyboardInterrupt`-adjacent cases (fine here, since `KeyboardInterrupt` inherits from `BaseException`, not `Exception`, so it isn't caught), but also things like an `AttributeError` from a bad `normalize_transactions_for_anomaly` — and silently fall back. If the detail view returns valid rows but the projection breaks, users will get the raw-table result without realising the enrichment was lost.

Two ways to tighten this without over-engineering it for notebook code:

**Option A — narrow the scope to just the load:**

```python
def load_anomaly_transactions():
    try:
        df = load_tenant_hudi(transaction_detail_view_path)
    except Exception as e:  # noqa: BLE001 -- notebook-scoped fallback
        print(f"Transaction detail view unavailable; falling back to gold/transactions: {e}")
        df = load_tenant_hudi(transactions_path)
    return normalize_transactions_for_anomaly(df)
```

**Option B — catch only the ones that mean "not there":**

```python
from pyspark.sql.utils import AnalysisException

def load_anomaly_transactions():
    try:
        return normalize_transactions_for_anomaly(load_tenant_hudi(transaction_detail_view_path))
    except (AnalysisException, ValueError) as e:
        print(f"Transaction detail view unavailable; falling back to gold/transactions: {e}")
        return normalize_transactions_for_anomaly(load_tenant_hudi(transactions_path))
```

Non-blocking, but Option A is a small change with meaningful debug benefit.

#### Issue 2 — CodeRabbit nitpick still open: `event_date` will be null on the fallback path

**Severity: Minor (Data Integrity)**

CodeRabbit flagged (Trivial-severity nitpick) that on the raw `gold/transactions` fallback path, `event_date` may not exist even when `event_ts` does. As written, the fallback produces `event_date = NULL` when only `event_ts` (or `tx_event_ts`) is present.

```python
_transaction_col(df, "event_date", ("tx_event_date",), "date").alias("event_date"),
```

If any downstream cell partitions or groups by `event_date`, calibration/clustering runs against the fallback lineage will silently see everything on `NULL`. The fix CodeRabbit proposed is a one-line coalesce:

```python
F.coalesce(
    _transaction_col(df, "event_date", ("tx_event_date",), "date"),
    F.to_date(_transaction_col(df, "event_ts", ("tx_event_ts",), "timestamp")),
).alias("event_date"),
```

The author didn't address this nitpick in commit `84184af` (only the Major finding was resolved). Worth adding to the same PR — the fix is small and the failure mode is silent.

#### Issue 3 — PR title too vague (CodeRabbit's inconclusive title check)

**Severity: Informational (Process)**

CodeRabbit's pre-merge "Title check" flagged the title as inconclusive because "corrected small bug" doesn't describe what actually changed. This is cosmetic for the merge, but the commit will land on `dev` with a title that reads as low-signal in `git log`. Consider rewording to something like `fix(anomaly-notebook): route transactions through vw_transaction_detail for event-history lineage`.

Non-blocking.

#### Issue 4 — Ruff cells 14 and 35 still contain bare `from pyspark.sql import functions as F`

**Severity: Informational (Pre-existing)**

Cells 14 and 35 in the notebook contain nothing but `from pyspark.sql import functions as F` on their own. These were on `dev` before this PR (confirmed via `git show origin/dev:JupyterHub/notebooks/... `) — they exist as standalone marker cells so ruff's E402 doesn't fire (no assignments precede the import within those cells) and they don't produce F811 because they are the first `F` binding in their cells. Not introduced by this PR, and not blocking. Flagging only so future readers don't chase them.

[↑ Back to top](#pr-review-biar-121--fix-corrected-small-bug-in-anomaly-notebook-regarding-event-history-transactions-lineage-downstream-effect)

---

## Security Assessment

| Concern | Assessment |
|---------|-----------|
| Tenant scoping | The load path is `load_tenant_hudi(...)`, which filters by `TENANT_FILTER_VALUE` before returning. Both the primary (detail view) and fallback (raw table) go through the same tenant filter — no tenant leakage introduced. |
| Injection / dynamic column names | All column names in `_transaction_col` calls are static literals in the notebook; nothing is user-controlled. No injection risk. |
| Broad exception handler leaking info | The `print(f"... {e}")` in the fallback path could reveal internal path names in logs. For a Jupyter notebook running under an authenticated user session, this is acceptable — not a new exposure. |
| Auth / permissions | Not touched. |

No new security vulnerabilities introduced by this PR.

[↑ Back to top](#pr-review-biar-121--fix-corrected-small-bug-in-anomaly-notebook-regarding-event-history-transactions-lineage-downstream-effect)

---

## Test Coverage

- **What is tested:** Nothing added. There are no automated tests for this notebook in the repo, and none are added here. The PR body notes the author ran it "Locally" (checkbox ticked), verified notebook JSON validity, and ran `git diff --check`.
- **What is not tested:** The fallback path itself — i.e., what happens when `views/vw_transaction_detail` is absent and the raw `gold/transactions` schema drives the projection. Given `event_date` will be null on that path (Issue 2 above), the fallback branch has an undetected data-integrity gap.
- **PR checklist:** "Locally" is ticked. "Development Environment", "Not needed", "Husky", and "Unit tests" are unticked — expected for a notebook change.
- **CI evidence:** Python Lint green (E402 / F811 issues from earlier commit resolved by `720ff0a`); TypeScript Build/Lint/Test green; CodeQL, njsscan, hadolint, DCO, GPG, encoding, dependency-review, conventional commits all green. Docker Build Check IN_PROGRESS at fetch time — expected to pass.

If practical, a manual run of the notebook against a tenant where `views/vw_transaction_detail` is missing (to exercise the fallback branch) would validate the raw-schema projection before merge. Not a hard block.

[↑ Back to top](#pr-review-biar-121--fix-corrected-small-bug-in-anomaly-notebook-regarding-event-history-transactions-lineage-downstream-effect)

---

## CodeRabbit Activity

Two CodeRabbit review passes ran on this PR, plus a rate-limited final walkthrough.

### Pass 1 — Initial commit `f9bb3cd`

**Commit reviewed:** `f9bb3cd`
**Findings:** 1 Major + 1 Nitpick actionable

| Finding | Severity | Status |
|---------|----------|--------|
| `_transaction_col` uses case-sensitive `col_name in df.columns`; camelCase source columns (e.g. `tenantId`, `endToEndId`) would silently return NULL | Major (Functional Correctness) | ✅ Resolved in `84184af` — now uses a `{c.lower(): c for c in df.columns}` mapping and looks up `col_name.lower()`. CodeRabbit itself marked `✅ Addressed in commit 84184af`. |
| `event_date` projection can be null on the fallback path when only `event_ts` is present — suggest `F.coalesce(event_date, F.to_date(event_ts))` | Trivial nitpick (Data Integrity) | ❌ Not resolved — no code change in either follow-up commit, no author response in comments. Corroborated in [Issue 2](#issue-2--coderabbit-nitpick-still-open-event_date-will-be-null-on-the-fallback-path). |

### Pass 2 — `f9bb3cd..720ff0a` (rate-limited)

**Commit reviewed:** `84184af..720ff0a`
**Findings:** 0 actionable — CodeRabbit hit its per-developer PR review rate limit and only produced a walkthrough/pre-merge check summary. The one flagged pre-merge item was an *inconclusive* Title check (see [Issue 3](#issue-3--pr-title-too-vague-coderabbits-inconclusive-title-check)).

**Reconciliation summary:** the Major-severity finding is fully resolved and both sides agree on the fix. The Trivial-severity nitpick is unaddressed and I have promoted it to Minor severity in [Issue 2](#issue-2--coderabbit-nitpick-still-open-event_date-will-be-null-on-the-fallback-path) because the silent-null failure mode matters for the fallback path this PR explicitly introduces.

[↑ Back to top](#pr-review-biar-121--fix-corrected-small-bug-in-anomaly-notebook-regarding-event-history-transactions-lineage-downstream-effect)

---

## Summary and Verdict

**Verdict: Approve with minor cleanup requested**

Correct, well-scoped fix for a real lineage drift: the anomaly notebook now sources enrichment fields from `vw_transaction_detail` and gracefully falls back to raw `gold/transactions`, with a stable 13-column projection and case-insensitive column resolution. The three commits trace a clean fix → CodeRabbit-response → lint-response arc, and CI is green across the board.

Two loose ends are worth cleaning up in the same PR: (1) the `event_date` coalesce that CodeRabbit flagged as a data-integrity nitpick would prevent a silent-null failure mode on the fallback path this PR itself introduces, and (2) the `except Exception` in `load_anomaly_transactions` catches too broadly and can mask projection-side bugs. Both are one-line fixes. Everything else (PR title vagueness, pre-existing standalone F-import cells) is informational.

### Blocking

None.

### Non-blocking but recommended

1. **Coalesce `event_date` from `event_ts`** — apply the CodeRabbit-proposed one-liner to guarantee `event_date` is populated on the raw `gold/transactions` fallback path. Silent-null grouping is exactly the kind of drift this PR is trying to prevent.
2. **Narrow the exception handler in `load_anomaly_transactions`** — either restrict the `try` to the load call (Option A above) or catch `(AnalysisException, ValueError)` explicitly, so a bug in `normalize_transactions_for_anomaly` doesn't silently degrade to the raw table.
3. **Retitle the PR** to describe the actual change — e.g. `fix(anomaly-notebook): route transactions through vw_transaction_detail for event-history lineage`. Not required for the merge.

[↑ Back to top](#pr-review-biar-121--fix-corrected-small-bug-in-anomaly-notebook-regarding-event-history-transactions-lineage-downstream-effect)

---

## GitHub Review Comment

````markdown
**Approve with minor cleanup requested**

Correct fix for the event-history lineage drift: routing transactions through `vw_transaction_detail` with a raw-table fallback and a stable 13-column projection is the right approach, and the case-insensitive `_transaction_col` (post `84184af`) is exactly what the fallback needs to survive camelCase Ozone columns. CI green, CodeRabbit's Major finding resolved.

Two small follow-ups in the same PR would tighten this up. Neither blocks merge but both prevent silent-degradation modes on the very fallback path this PR introduces.

---

### Non-blocking (please address in this PR if possible)

**1. Coalesce `event_date` from `event_ts` on the fallback path**

`JupyterHub/notebooks/Anomaly_Detection_And_Rule_Calibration.ipynb`, cell 8, inside `normalize_transactions_for_anomaly`:

```diff
-        _transaction_col(df, "event_date", ("tx_event_date",), "date").alias("event_date"),
+        F.coalesce(
+            _transaction_col(df, "event_date", ("tx_event_date",), "date"),
+            F.to_date(_transaction_col(df, "event_ts", ("tx_event_ts",), "timestamp")),
+        ).alias("event_date"),
```

If the raw `gold/transactions` fallback branch fires and the table only has `event_ts` (or `tx_event_ts`), the current projection produces `event_date = NULL` for every row. Any downstream cell that groups or partitions by `event_date` will collapse the whole result silently. This is CodeRabbit's still-open nitpick — worth taking.

**2. Narrow `except Exception` in `load_anomaly_transactions`**

Same cell 8. Either narrow the scope so a bug in `normalize_transactions_for_anomaly` isn't caught:

```python
def load_anomaly_transactions():
    try:
        df = load_tenant_hudi(transaction_detail_view_path)
    except Exception as e:  # noqa: BLE001 -- notebook-scoped fallback
        print(f"Transaction detail view unavailable; falling back to gold/transactions: {e}")
        df = load_tenant_hudi(transactions_path)
    return normalize_transactions_for_anomaly(df)
```

…or catch the actual "not there" exceptions:

```python
from pyspark.sql.utils import AnalysisException

def load_anomaly_transactions():
    try:
        return normalize_transactions_for_anomaly(load_tenant_hudi(transaction_detail_view_path))
    except (AnalysisException, ValueError) as e:
        print(f"Transaction detail view unavailable; falling back to gold/transactions: {e}")
        return normalize_transactions_for_anomaly(load_tenant_hudi(transactions_path))
```

**3. Optional: retitle the PR**

`fix: corrected small bug` doesn't describe the change; the CodeRabbit pre-merge Title check flagged the same. Something like `fix(anomaly-notebook): route transactions through vw_transaction_detail for event-history lineage` reads cleanly in `git log` on `dev`.
````

[↑ Back to top](#pr-review-biar-121--fix-corrected-small-bug-in-anomaly-notebook-regarding-event-history-transactions-lineage-downstream-effect)
