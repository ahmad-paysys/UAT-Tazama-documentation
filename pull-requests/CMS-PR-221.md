# PR Review: CMS #221 — fix: Transaction History Graph and Visualization

## Table of Contents

- [Initial Review (2026-06-30)](#initial-review-2026-06-30)
  - [Overview](#overview)
  - [What Changed (Detailed)](#what-changed-detailed)
  - [Issues and Observations](#issues-and-observations)
  - [Security Assessment](#security-assessment)
  - [Test Coverage](#test-coverage)
  - [Summary and Verdict](#summary-and-verdict)
- [Follow-up Review (2026-06-30)](#follow-up-review-2026-06-30)
  - [Changes Requested — Resolution Status](#changes-requested--resolution-status)
  - [New Issues Found in Updated Commits](#new-issues-found-in-updated-commits)
  - [Updated Verdict](#updated-verdict)

---

## Initial Review (2026-06-30)

**Repo:** tazama-lf/case-management-system  
**Branch:** `paysys/code-fixes` → `dev`  
**Author:** Sobia Rizwan  
**Date Reviewed:** 2026-06-30  
**Label:** bug  
**Size:** +994 / -907 lines across 4 files

---

## Overview

[↑ Back to top](#pr-review-cms-221--fix-transaction-history-graph-and-visualization)

This PR fixes broken graph rendering in the Transactional History view of the Case Management System. The changes are entirely within Jupyter notebooks (`.ipynb` files) served via Voila. Four files are touched:

| File | Nature of Change |
|------|-----------------|
| `notebooks/transaction-viz.ipynb` | Major rewrite of visualization logic |
| `notebooks/account-network.ipynb` | One-line fix: apply `numberFormatter` to Total Value stat |
| `notebooks/alert-history.ipynb` | EOF newline fix only |
| `notebooks/counterparty-network.ipynb` | EOF newline fix only |

The core fix is in `transaction-viz.ipynb`, which was producing incorrect or blank charts. The notebook has been substantially restructured with better data normalization, granularity-aware axis formatting, security hardening, and improved fallback rendering.

---

## What Changed (Detailed)

[↑ Back to top](#pr-review-cms-221--fix-transaction-history-graph-and-visualization)

### 1. `account-network.ipynb` — Formatting Fix

```diff
- document.getElementById('stat-total-value').innerText = stats.total || 0;
+ document.getElementById('stat-total-value').innerText = numberFormatter.format(stats.total || 0);
```

**Correct and minimal.** The `stat-total-tx` element already used `numberFormatter` but `stat-total-value` did not, causing inconsistent display of raw numeric values vs. locale-formatted numbers in the sidebar. This is an isolated, clearly correct fix.

---

### 2. `alert-history.ipynb` and `counterparty-network.ipynb` — EOF Fix

Both files remove a trailing newline after the closing `}`, adding a `\ No newline at end of file` marker. This is a cosmetic/formatting-only change with no behavioral impact. It is consistent cleanup.

---

### 3. `transaction-viz.ipynb` — Core Visualization Rewrite

This is the substantive change. Key improvements:

#### a. Security: URL Injection Prevention (New)
```python
# Before
url = f"{backend_url}/api/v1/jupyter/proxy/transaction-history/{id_value}"

# After
safe_id = quote(str(id_value), safe="")
url = f"{backend_url}/api/v1/jupyter/proxy/transaction-history/{safe_id}"
```
The `accountId` originates from the Voila URL query string (`parse_qs`). The old code interpolated it directly into the URL path, enabling path traversal or injection (e.g., `accountId = "../admin"`). The fix applies `urllib.parse.quote()` to URL-encode the ID before interpolation. Same fix applied to the Benford endpoint.

#### b. Security: HTML Injection Prevention (New)
```python
# Before
display(HTML(f"...{accountId}..."))
display(HTML(f"...Error: {str(e)}..."))

# After
from html import escape
display(HTML(f"...{escape(str(accountId))}..."))
display(HTML(f"...Error: {escape(str(e))}..."))
```
`accountId` and exception messages are now HTML-escaped before being rendered into inline HTML. This prevents XSS in the Voila-hosted notebook if a malicious ID is supplied.

#### c. Request Timeouts Added
```python
REQUEST_TIMEOUT_SECONDS = 30
response = requests.get(url, params=params, headers=headers, timeout=REQUEST_TIMEOUT_SECONDS)
```
The original code had no timeout on HTTP requests, meaning a slow or hung backend would block the notebook indefinitely. A 30-second timeout is now applied to all `requests.get()` calls. The Benford endpoint still hardcodes `timeout=20` separately — a minor inconsistency (see Issues below).

#### d. Double Fetch Fallback Removed
```python
# Before: fallback re-fetch with same params was tried if first fetch returned None
data = fetch_transaction_history(accountId, backendUrl, filter)
if not data:
    data = fetch_transaction_history(accountId, backendUrl)  # redundant retry

# After: single fetch, no redundant retry
data = fetch_transaction_history(accountId, backendUrl, filter)
```
The fallback was calling the same endpoint without the granularity filter — it provided no real fallback value and masked errors. Removing it is correct.

#### e. Date Parsing Hardening
```python
# Before
df_timeline['date'] = pd.to_datetime(df_timeline['date'])

# After
df_timeline['date'] = pd.to_datetime(df_timeline['date'], errors='coerce')
df_timeline = df_timeline.dropna(subset=['date'])
```
`errors='coerce'` converts unparseable dates to `NaT` instead of raising. Rows with `NaT` dates are then dropped, preventing downstream chart failures from malformed backend data. Same treatment applied to `df_cumulative`.

#### f. Granularity-Aware Visualization Architecture
The old chart used a single x-axis range and tick config for all three subplots. The new approach:

- **Row 1 (Transaction Timeline)** and **Row 2 (Cumulative Value)**: Use exact per-transaction timestamps with a `timestamp_xaxis_cfg` that adapts tick format and count to the active granularity (`day`/`month`/`year`).
- **Row 3 (Volume Distribution)**: Uses a derived `df_volume_derived` DataFrame built by grouping `df_timeline` by the selected period granularity, with categorical date strings on the x-axis (e.g., `"Jun 2026"`). This prevents bars from being pixel-thin when data spans a wide time range.

The dual-axis (secondary y) approach in Row 1 was also dropped. The old design overlaid a "Transaction Count" line on a secondary y-axis alongside the "Amount" line. The new design uses a simpler single y-axis for Row 1 (amount per transaction, color-coded by alert status), while transaction count is moved to Row 3 as a proper bar chart. This is a cleaner separation of concerns.

#### g. Alert Visualization Improvement
```python
# Old: separate traces for alert/normal periods
# New: single trace with per-point color coding
marker=dict(
    color=df_timeline['isAlerted'].map({True: '#EF4444', False: '#3B82F6'}),
    ...
),
customdata=df_timeline['isAlerted'].map({True: '⚠ ALERT', False: 'Normal'}),
```
Alert status is now encoded per-transaction as marker color and shown in the hover tooltip. Two legend-only ghost traces (`x=[None], y=[None]`) provide a color legend without adding phantom chart data.

#### h. Cumulative Chart: `cumulativeCount` Added
```python
df_cumulative = pd.DataFrame({
    'date': df_timeline['date'],
    'cumulativeAmount': df_timeline['amount'].cumsum(),
    'cumulativeCount': range(1, len(df_timeline) + 1)  # new
})
```
The cumulative chart hover now shows the transaction sequence number (`Txn #N`), which gives users useful context when reviewing the running total.

#### i. Recent Transactions Table: Null-Safe Formatting
```python
# Old: could crash on non-numeric amounts
display_df['amount'] = display_df['amount'].apply(lambda x: f"{x:,.2f}")

# New: safe coercion
amount = pd.to_numeric(display_df['amount'], errors='coerce')
display_df['amount'] = amount.apply(lambda x: f"{x:,.2f}" if pd.notnull(x) else '—')
```
Also added `.fillna('—')` for the date column after `pd.to_datetime(..., errors='coerce')`. These are defensive improvements that prevent the table from crashing on bad data from the backend.

---

## Issues and Observations

[↑ Back to top](#pr-review-cms-221--fix-transaction-history-graph-and-visualization)

### Bug: Duplicate line in `viz_metrics` cell
```python
total_vol = f"{summary.get('totalVolume', 0):,.2f}"
total_vol = f"{summary.get('totalVolume', 0):,.2f}"  # identical duplicate
```
`total_vol` is assigned twice with the same expression on consecutive lines. The second assignment is redundant. This is a minor copy-paste leftover.

### Code Quality: Large volume of commented-out code
The `viz_charts` cell contains many commented-out blocks — previous implementations of date range logic, `df_volume_derived` derivation, and x-axis config. While useful during development, this dead code should be removed before merge:
- Lines with `# if _filter == "day":` / `# df_volume_derived =` blocks
- The `# fig.update_xaxes(...)` block for Row 1 (spanning ~15 lines)
- The `# x_range_start / x_range_end` comments from the old approach

This is a notebook, so the visual clutter is especially disruptive to future readers.

### Minor: Indentation comment misalignment
```python
df_volume_derived['date_str'] = df_volume_derived['date'].dt.strftime(...)
    # This is already correct since groupby('period') uses the period column
    # but cast to date string for Plotly to treat as categorical
```
The inline comment block is indented with 4 extra spaces, making it appear to be part of a code block that doesn't exist. Minor but confusing.

### Minor: `filter` shadows Python built-in
```python
filter = None
```
`filter` is a Python built-in function. While this works without issue in the notebook context (it's a local variable and `filter` is never called as a built-in here), it is poor practice and could confuse linters or future readers. `time_filter` or `granularity` would be better names. This is pre-existing in the old code, not introduced by this PR.

### Minor: `fetch_json` helper defined but not called
```python
def fetch_json(endpoint: str, params: dict):
    url = f"{backendUrl}{endpoint}"
    resp = requests.get(url, params=params, headers=headers, timeout=REQUEST_TIMEOUT_SECONDS)
    resp.raise_for_status()
    return resp.json()
```
This helper is defined in the `fetch_data` cell but is not called anywhere in the notebook. It's dead code. Either it should be used (e.g., refactoring Benford fetch to use it) or removed.

### Observation: Benford timeout inconsistency
```python
# In fetch_transaction_history:
timeout=REQUEST_TIMEOUT_SECONDS  # 30s

# In Benford fetch:
timeout=20  # hardcoded, separate from the constant
```
The Benford request uses a hardcoded `20` while the main fetch uses the `REQUEST_TIMEOUT_SECONDS` constant. This makes the constant incomplete. Either `REQUEST_TIMEOUT_SECONDS` should be used everywhere, or separate named constants should be defined.

### Observation: `df_volume` from backend is now unused
The old code populated `df_volume` from the backend's `volumeDistribution` key and used it for Row 3. The new code derives `df_volume_derived` entirely from `df_timeline`, ignoring `df_volume` entirely. The normalization logic for `df_volume` (mapping `bucketStart`, `transactionCount`, `bucket_tx_count`, etc.) is still present in the `fetch_data` cell but no longer read by any chart code. This dead code should be cleaned up.

### Observation: Row 3 uses categorical string axis, not datetime
```python
x=df_volume_derived['date_str'],  # e.g., "Jun 2026"
```
Using string labels for the bar chart x-axis means Plotly treats them as categorical. This prevents zooming/panning the bar chart in sync with Rows 1 and 2 (which use `shared_xaxes=False` anyway). This is a deliberate tradeoff to avoid pixel-thin bars, and it is reasonable given the independent x-axes. However, it means the `_bar_x_start`/`_bar_x_end` variables computed in the granularity block are now unused — more dead code.

---

## Security Assessment

[↑ Back to top](#pr-review-cms-221--fix-transaction-history-graph-and-visualization)

| Concern | Old Code | New Code |
|---------|----------|----------|
| Path traversal via `accountId` in URL | Vulnerable | Fixed (URL encoding) |
| XSS via `accountId` in HTML output | Vulnerable | Fixed (`html.escape`) |
| XSS via exception message in HTML | Vulnerable | Fixed (`html.escape`) |
| Request timeout (DoS / hung backend) | Missing | Added (30s) |
| Token in Authorization header | Bearer token from env | Unchanged (correct) |

All four security improvements are valid and should have been present from the start. This PR represents a meaningful security improvement.

---

## Test Coverage

[↑ Back to top](#pr-review-cms-221--fix-transaction-history-graph-and-visualization)

No automated tests exist or are expected for Jupyter notebooks in this project (the PR checklist confirms "Unit tests passing" but this is likely referring to other project tests). The changes were validated locally and in the development environment per the PR description.

---

## Summary and Verdict

[↑ Back to top](#pr-review-cms-221--fix-transaction-history-graph-and-visualization)

**Verdict: Approve with minor cleanup requested**

The PR achieves its stated goal — fixing broken transaction history graphs. The underlying fixes are sound:
- The visualization architecture change (granularity-aware axes, per-transaction scatter vs. aggregated bars) directly addresses the root cause of charts not displaying correctly.
- The security fixes (URL encoding, HTML escaping, timeouts) are correct and necessary improvements that go beyond the stated scope.
- Data parsing hardening (`errors='coerce'`, `dropna`) prevents crashes on bad backend data.

The blocking items before merge are minor:
1. Remove the duplicate `total_vol` assignment.
2. Remove commented-out code blocks from `viz_charts`.
3. Remove unused `_bar_x_start`/`_bar_x_end` variables.
4. Either use `fetch_json` or remove it.
5. Either clean up the `df_volume` normalization block (now unused) or add a comment explaining it's kept for future backend integration.

None of these are defects — they are cleanliness issues. The functional changes are correct and the security improvements are a net positive.

---
---
---

## Follow-up Review (2026-06-30)

[↑ Back to top](#pr-review-cms-221--fix-transaction-history-graph-and-visualization)

**Reviewed commit:** `ef3404a5` (latest as of 2026-06-30)  
**Reviewed against:** CHANGES_REQUESTED on commit `ecc8218d` by `ahmad-paysys`  
**Developer response:** Sobia Rizwan posted: *"All of the issues are catered."*

The PR has received 6 total commits since the initial push, with the latest (`ef3404a5`) arriving after both the CHANGES_REQUESTED review and CodeRabbit's third round of comments. The review below verifies each requested change against the current notebook state.

---

## Changes Requested — Resolution Status

[↑ Back to top](#pr-review-cms-221--fix-transaction-history-graph-and-visualization)

### Item 1 — Remove the duplicate `total_vol` assignment

**Status: RESOLVED**

The `viz_metrics` cell now contains a single assignment:
```python
total_vol = f"{summary.get('totalVolume', 0):,.2f}"
```
The duplicate line is gone.

---

### Item 2 — Remove commented-out code blocks from `viz_charts`

**Status: PARTIALLY RESOLVED — still has issues**

Some commented-out code was removed, but the cleanup is incomplete. The following commented-out lines remain in the notebook:

**In cell 2 (`fetch_data`):**
```python
# df_timeline['date'] = pd.to_datetime(df_timeline['date'])
# df_cumulative['date'] = pd.to_datetime(df_cumulative['date'])
```
These are the old strict-parse lines sitting right above the new `errors='coerce'` equivalents. They serve as an implicit "before/after" comment, but they are dead code and should be removed.

**In cell 4 (`viz_charts`):**
```python
# df_timeline = df_timeline.sort_values('date')
```
Commented-out sort is dead code — the actual sort happens in `fetch_data` already.

The section-header comments (e.g., `# 1. Transaction Timeline - per-transaction amount`, `# 3. Volume Distribution — derived from df_timeline`, `# Layout & axes styling`, etc.) are retained and are fine — they serve as documentation for chart sections. The truly problematic dead code is the commented-out executable statements listed above.

**Overall: the bulk of the commented-out block clutter from the initial review has been removed, but a few stray commented-out code lines remain.**

---

### Item 3 — Remove unused `_bar_x_start`/`_bar_x_end` variables

**Status: NOT RESOLVED**

`_bar_x_start` and `_bar_x_end` are still assigned in all three branches of the granularity `if/elif/else` block in cell 4:

```python
if _filter == "day":
    ...
    _bar_x_start = (df_timeline['date'].min() - timedelta(days=1)).strftime('%Y-%m-%d')
    _bar_x_end   = (df_timeline['date'].max() + timedelta(days=1)).strftime('%Y-%m-%d')
elif _filter == "year":
    ...
    _bar_x_start = (df_timeline['date'].dt.to_period('Y').min()...).strftime('%Y-%m-%d')
    _bar_x_end   = (df_timeline['date'].dt.to_period('Y').max()...).strftime('%Y-%m-%d')
else:
    ...
    _bar_x_start = (df_timeline['date'].dt.to_period('M').min()...).strftime('%Y-%m-%d')
    _bar_x_end   = (df_timeline['date'].dt.to_period('M').max()...).strftime('%Y-%m-%d')
```

None of the `fig.update_xaxes(range=...)` calls reference `_bar_x_start` or `_bar_x_end`. The only `range=` calls in the chart are:
```python
range=[_tl_x_start, _tl_x_end]   # Row 1
range=[_cum_x_start, _cum_x_end]  # Row 2
```
Row 3 uses a categorical string axis (`date_str`) so no x-range is applied to it at all. These six assignments are dead code. **This item was not addressed.**

---

### Item 4 — Either use `fetch_json` or remove it

**Status: RESOLVED**

`fetch_json` has been fully removed from the notebook. No reference to it exists in the current commit.

---

### Item 5 — Clean up `df_volume` normalization block (unused) or comment it

**Status: RESOLVED**

The `df_volume` / `volumeDistribution` normalization block has been removed from the notebook. This aligns with a broader cleanup: the backend also removed the `volumeDistribution` field from `TransactionHistoryResponse` (both the TypeScript interface and the service mapping), and the corresponding test assertion was dropped. The notebook, backend service, and type definitions are now consistent — none of them reference `volumeDistribution` anymore.

---

## New Issues Found in Updated Commits

[↑ Back to top](#pr-review-cms-221--fix-transaction-history-graph-and-visualization)

The latest commits introduced backend changes that were not part of the original PR scope and warrant attention.

### Backend: `volumeDistribution` removal

Three backend files were modified to remove the `volumeDistribution` field:

| File | Change |
|------|--------|
| `backend/src/modules/gold-lakehouse/transaction-lakehouse.service.ts` | Removed mapping block |
| `backend/src/modules/gold-lakehouse/types/transaction-history-response.types.ts` | Removed type field |
| `backend/test/transaction-lakehouse.service.spec.ts` | Removed test assertion |

This is a **breaking API change**: any client consuming the `volumeDistribution` field from the transaction history endpoint will silently receive `undefined` instead of an array. Since the notebook was the only known consumer and it no longer uses the field, this is likely safe — but it should be confirmed that no other dashboard, report, or integration depends on this field before merging.

### CodeRabbit Round 3: `accountId` guard before Benford URL (open)

CodeRabbit's third review (on commit `19687b77`) flagged that the Benford URL construction could produce `/account/None` if `accountId` is falsy. The latest commit (`ef3404a5`) added a guard:
```python
if not accountId:
    display(HTML("..."))
else:
    benford_url = f"..."
```
This resolves CodeRabbit's concern. Confirmed fixed.

---

## Updated Verdict

[↑ Back to top](#pr-review-cms-221--fix-transaction-history-graph-and-visualization)

**Verdict: Still requires one fix before approval**

| # | Item | Status |
|---|------|--------|
| 1 | Remove duplicate `total_vol` | ✅ Resolved |
| 2 | Remove commented-out code blocks | ⚠️ Partial — a few stray lines remain |
| 3 | Remove unused `_bar_x_start`/`_bar_x_end` | ❌ Not resolved |
| 4 | Remove or use `fetch_json` | ✅ Resolved |
| 5 | Clean up `df_volume` normalization | ✅ Resolved |

Item 3 is the only clean blocker — six dead-code lines that are trivial to remove. Item 2 has a few stray commented-out executable statements left but is mostly addressed; it can be resolved in the same pass as item 3.

The developer should also confirm whether any other consumer of the `/transaction-history` API depended on `volumeDistribution` before the backend removal is merged.

Once item 3 (and the two remaining stray commented-out lines) is cleaned up, the PR is ready to approve.
