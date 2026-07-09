# PR Review: BIAR #107 — fix: changes in notebooks

**Repo:** tazama-lf/biar
**Branch:** `fixes` → `dev`
**Author:** hassanrizwan-paysys (Hassan Rizwan)
**Date Reviewed:** 2026-07-09
**Label:** bug
**Size:** +1638 / −37794 lines across 16 files (net churn dominated by deleted notebook output cells and the removal of two obsolete dashboard notebooks)
**Commits:** 9 (2a00fe7, 3632ff1, 942faf7, d2cc03d, c6731af, 6774f8b, d4bf427, b71bf10, 84b894c)
**State:** OPEN
**Existing approvals:** None. CodeRabbit has commented four times (2026-07-06 ×2, 2026-07-08, 2026-07-09) with nitpicks/minor findings; no human review yet.

---

## Table of Contents

- [Overview](#overview)
- [What Changed (Detailed)](#what-changed-detailed)
  - [1. Account_HolderETL.py — credttm epoch-ms conversion](#1-account_holderetlpy--credttm-epoch-ms-conversion)
  - [2. CommentsETL.py — created_at/updated_at epoch-ms conversion](#2-commentsetlpy--created_atupdated_at-epoch-ms-conversion)
  - [3. TypologiesETL.py — permissive JSON parse, drop typology_name, getField access](#3-typologiesetlpy--permissive-json-parse-drop-typology_name-getfield-access)
  - [4. alert_navigator.py — join bronze rules metadata into alerts_nav_rules](#4-alert_navigatorpy--join-bronze-rules-metadata-into-alerts_nav_rules)
  - [5. lakehouse_query_api.py — path registry additions and typologies gold correction](#5-lakehouse_query_apipy--path-registry-additions-and-typologies-gold-correction)
  - [6. Query.ipynb — tenant-scoped Hudi loader](#6-queryipynb--tenant-scoped-hudi-loader)
  - [7. Lakehouse_Catalog.ipynb — inline documentation comments](#7-lakehouse_catalogipynb--inline-documentation-comments)
  - [8. Dashboard notebooks — regen, tenant filters, deletions](#8-dashboard-notebooks--regen-tenant-filters-deletions)
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

This PR bundles several loosely related fixes to the biar analytics stack:

1. **Sub-rule-ref / band-reason surfacing for CMS.** `alert_navigator.py` now loads the `bronze/rule` layer, parses each rule's `config.bands` and `config.exitConditions`, joins the metadata onto every alert-rule row, and derives `matched_band_reason`, `matched_exit_condition_reason`, and `matched_rule_reason` columns. This is the substantive functional change and the one aligned with the stated PR objective ("Band and exit condition reasons and sub rule ref for CMS visualizations").
2. **Epoch-milliseconds bug fixes.** `Account_HolderETL.credttm` and `CommentsETL.created_at/updated_at` were being converted assuming raw seconds and microseconds respectively; both now divide by `1_000` (milliseconds → seconds) before `F.to_timestamp`. This matches the convention already established in `EntityETL.py` (which explicitly documents `credttm` as epoch ms).
3. **TypologiesETL cleanup.** JSON schema inference is switched to `PERMISSIVE` mode, nested-field access moves from dotted string paths (`typology_obj.rules`) to `.getField("rules")`, and `typology_name` is dropped from Silver/Gold projections.
4. **Path registry corrections.** `lakehouse_query_api.py` fixes a mis-routed `"typologies"` entry that was pointing at the bronze path, and adds three new bronze entries (`bronze_alerts`, `rules_bronze`, `typologies_bronze`).
5. **Notebook housekeeping.** Two notebooks are deleted (`Dashboard_Metrics.ipynb`, `Executive_Overview_Dashboard-bk.ipynb`); nine remaining notebooks are re-saved with cleared outputs, inline explanatory comments, and a tenant-filter cell in `Query.ipynb`.

| File | Nature of Change |
|------|------------------|
| `automation-orchestrator/Table_ETLs/Account_HolderETL.py` | Fix: divide `credttm` by 1_000 before `to_timestamp` (epoch ms → seconds) |
| `automation-orchestrator/Table_ETLs/CommentsETL.py` | Fix: divide `created_at`/`updated_at` by 1_000 (was 1_000_000). **Stale comment left in place.** |
| `automation-orchestrator/Table_ETLs/TypologiesETL.py` | PERMISSIVE JSON parse, `.getField(...)` access, drop `typology_name` from Silver+Gold, whitespace-only churn on most other lines |
| `automation-orchestrator/Table_ETLs/alert_navigator.py` | Add `_prepare_rule_metadata`, extend `_build_rules(a, b_rules)`, join bronze rules, derive matched band/exit reasons, extend `pk` inputs |
| `datalakehouse-api/lakehouse_query_api.py` | Fix mis-routed `"typologies"` gold path; add `bronze_alerts`, `rules_bronze`, `typologies_bronze` entries |
| `JupyterHub/notebooks/Query.ipynb` | Add `load_tenant_hudi` helper with hardcoded `TENANT_FILTER_VALUE = "TAZAMA"`; rework example cell |
| `JupyterHub/notebooks/Lakehouse_Catalog.ipynb` | Add 5 inline comments before existing helpers/loads (no code change) |
| `JupyterHub/notebooks/Dashboard_Metrics.ipynb` | **Deleted** (1990 lines) |
| `JupyterHub/notebooks/Executive_Overview_Dashboard-bk.ipynb` | **Deleted** (6493 lines) — backup file removed |
| `JupyterHub/notebooks/Anomaly_Detection_And_Rule_Calibration.ipynb`, `Case_Management_Trend_Dashboard.ipynb`, `Case_Tracking_Analysis_Dashboard.ipynb`, `Executive_Overview_Dashboard.ipynb`, `Fraud_Trend_Analysis_Dashboard.ipynb`, `Fraud_Typology_Effectiveness_Dashboard.ipynb`, `TMS_Performance_Dashboard.ipynb` | Cleared cell outputs, added `TENANT_FILTER_VALUE` filtering plumbing and inline comments |

[↑ Back to top](#pr-review-biar-107--fix-changes-in-notebooks)

---

## What Changed (Detailed)

### 1. Account_HolderETL.py — credttm epoch-ms conversion

```diff
 def silver(self) -> str:
     df = self.spark.read.format("hudi").load(self.bronze_path)
+    account_holder_created_ts = F.to_timestamp(
+        F.col("credttm").cast("double") / F.lit(1_000)
+    )
     silver = (
         df
         .withColumn("tenant_id",        F.col("tenantid"))
-        .withColumn("event_ts",         F.to_timestamp(F.col("credttm")))
-        .withColumn("event_date",       F.to_date(F.to_timestamp(F.col("credttm"))))
+        .withColumn("event_ts",         account_holder_created_ts)
+        .withColumn("event_date",       F.to_date(account_holder_created_ts))
```

**What / why:** `credttm` is stored as epoch milliseconds (confirmed by `EntityETL.py:7,77-78` in the same repo, which documents this convention and does the same conversion). Passing epoch-ms directly to `F.to_timestamp` would either return `NULL` or a wildly wrong far-future timestamp depending on Spark's handling; dividing by 1_000 first is correct. The intermediate variable also removes a redundant `F.to_timestamp(F.col("credttm"))` reconstruction on the `event_date` line — small readability win.

**Correctness:** ✅ Matches EntityETL convention. No caller change needed; downstream just consumes `event_ts` / `event_date`.

**Nit:** No comment left explaining "divide by 1_000 because credttm is epoch ms." EntityETL has such a comment (line 77). Not blocking.

---

### 2. CommentsETL.py — created_at/updated_at epoch-ms conversion

```diff
 # created_at / updated_at are stored as epoch microseconds
-created_ts = F.to_timestamp((F.col("created_at").cast("double") / F.lit(1_000_000)))
-updated_ts = F.to_timestamp((F.col("updated_at").cast("double") / F.lit(1_000_000)))
+created_ts = F.to_timestamp((F.col("created_at").cast("double") / F.lit(1_000)))
+updated_ts = F.to_timestamp((F.col("updated_at").cast("double") / F.lit(1_000)))
```

**What / why:** The commit message (`fix: fixed epoch ms conversion in comments for created at and updated at columns`) confirms the fields are actually epoch milliseconds, not microseconds. The divisor is corrected.

**Issue — the inline comment above still says `microseconds` and now contradicts the code.** CodeRabbit flagged this on 2026-07-09; the head commit `84b894c` did not update the comment. See [Issue 1](#issue-1--stale-inline-comment-in-commentsetlpy).

---

### 3. TypologiesETL.py — permissive JSON parse, drop typology_name, getField access

Three functional deltas plus wide whitespace-only reformatting.

**3a. Permissive JSON schema inference:**

```diff
-typ_schema = self.spark.read.json(
-    b.select("configuration_json").where(F.col("configuration_json").isNotNull()).rdd.map(lambda r: r[0])
+typ_schema = self.spark.read.option("mode", "PERMISSIVE").json(
+    b.select("configuration_json")
+     .where(F.col("configuration_json").isNotNull())
+     .rdd.map(lambda r: r[0])
 ).schema
```

`PERMISSIVE` is Spark's default mode, so this is a no-op semantically. Making the mode explicit is fine, but it does not "tighten" JSON handling as the release notes claim.

**3b. `.getField(...)` migration:** All `F.col("typology_obj.foo.bar")` accessors become `F.col("typology_obj").getField("foo").getField("bar")`. Semantically equivalent, more verbose. Justifiable if a downstream Spark version was throwing on the dotted form, otherwise cosmetic.

**3c. `typology_name` dropped from Silver+Gold:**

```diff
-.withColumn("typology_name",       F.col("typology_obj.typology_name"))
```

```diff
 F.col("typology_id_in_json").cast("string"),
 F.col("typology_cfg_in_json").cast("string"),
-F.col("typology_name").cast("string"),
 F.col("flow_processor").cast("string"),
```

**Verify:** I grepped the biar repo for `typology_name` — only two hits, both the removed lines above. No downstream ETL in this repo reads it. However, this column is exposed via the Hudi Gold table — any external consumer (CMS, dashboards, Trino/Athena queries) that projected `typology_name` will now break. The PR description does not mention this schema removal. See [Issue 3](#issue-3--typology_name-silently-removed-from-gold-schema).

**3d. Trailing newline missing:** The file ends `return self.gold_path` with no final newline (`\ No newline at end of file` in the diff). Minor.

---

### 4. alert_navigator.py — join bronze rules metadata into alerts_nav_rules

This is the core functional change. New method `_prepare_rule_metadata` parses `bronze/rule` `configuration_json` per (tenant, rule_id, rule_cfg), keeps only the latest by `ingested_at_ts`, extracts `bands` and `exitConditions` arrays and several JSON-serialised projections (band reasons, sub-rule refs, exit-condition reasons, and combined structs).

`_build_rules` gains a `b_rules` parameter and, when metadata is available, left-joins it onto the rules view with a tenant-fallback rule:

```python
(F.col("rule_id") == F.col("cfg_rule_id"))
& (F.col("rule_cfg") == F.col("cfg_rule_cfg"))
& (
    (F.col("tenant_id_norm") == F.col("cfg_tenant_id_norm"))
    | (F.col("cfg_tenant_id_norm") == F.lit("DEFAULT"))
)
```

Then ranks matches — tenant-exact = 0, `DEFAULT` = 1, other = 2 — and takes the top rank per `pk` via a row-number window. Finally derives:

```python
matched_band_reason           = element_at(transform(filter(bands_arr, x -> x.subRuleRef = rule_sub_ref), x -> x.reason), 1)
matched_exit_condition_reason = element_at(transform(filter(exit_conditions_arr, x -> x.subRuleRef = rule_sub_ref), x -> x.reason), 1)
matched_rule_reason           = coalesce(matched_band_reason, matched_exit_condition_reason)
```

The `pk` for rules is also broadened to include `typology_cfg` and `rule_cfg`:

```diff
 F.col("alert_id"),
 F.col("typology_id"),
+F.coalesce(F.col("typology_cfg"), F.lit("")),
 F.col("rule_id"),
+F.coalesce(F.col("rule_cfg"), F.lit("")),
 F.coalesce(F.col("rule_sub_ref"), F.lit("")),
```

**Correctness observations:**

- The `DEFAULT` fallback is scoped by upper-cased tenant literal `"DEFAULT"`, which is fine as a convention but should be documented. There is no code path that populates a `DEFAULT` tenant here — this is future-proofing.
- The rank-1 filter over the `pk` window is correct: same alert × typology × rule × sub-ref only keeps the best-matching metadata row.
- If `bronze/rule` doesn't exist (`_safe_load` returns `None`), `_prepare_rule_metadata` short-circuits to `None` and `_build_rules` returns without the join. Good defensive handling.
- **Schema change on `pk`.** Existing Hudi rows in the `alerts_nav_rules` view already have a `pk` built from `(alert_id, typology_id, rule_id, rule_sub_ref)`. New writes will use `(alert_id, typology_id, typology_cfg, rule_id, rule_cfg, rule_sub_ref)`. On the same alert+rule combination, the two hashes will not collide — meaning a historical row and a new row will co-exist as two separate keys. This may or may not be intentional. See [Issue 4](#issue-4--pk-change-on-alerts_nav_rules-creates-hudi-key-drift).
- **JSON columns are emitted but never consumed here.** `band_reasons_json`, `band_sub_rule_refs_json`, `band_reasons_with_sub_rule_refs_json`, `exit_condition_reasons_json`, `exit_condition_sub_rule_refs_json`, `exit_condition_reasons_with_sub_rule_refs_json` are all computed on `rule_meta` and joined — but they are NOT dropped before the final result, so the view schema grows by six string columns. If the CMS reads them directly, this is fine. If not, they're dead payload. The PR description says the CMS wants sub-rule refs and reasons, so this is likely intentional — worth confirming.
- Also emitted: `rule_desc`, `band_count`, `exit_condition_count`. Again silently added to the view schema.

**Trailing newline fix:** the file previously ended without a final newline; the PR adds one. ✅

---

### 5. lakehouse_query_api.py — path registry additions and typologies gold correction

```diff
-typologies_bronze_path     = f"{WAREHOUSE_ROOT}/bronze/typologies"
+typologies_gold_path     = f"{WAREHOUSE_ROOT}/gold/typologies"
+rules_bronze_path         = f"{WAREHOUSE_ROOT}/bronze/rule"
+typologies_bronze_path         = f"{WAREHOUSE_ROOT}/bronze/typologies"
+alerts_bronze_path         = f"{WAREHOUSE_ROOT}/bronze/alerts"
```

```diff
     "rule":                            rules_gold_path,
-    "typologies":                      typologies_bronze_path,
+    "typologies":                      typologies_gold_path,
     ...
+    "bronze_alerts":                      alerts_bronze_path,
+    "rules_bronze":                       rules_bronze_path,
+    "typologies_bronze":                  typologies_bronze_path,
```

**What / why:** The `"typologies"` key in `GOLD_PATHS` was mistakenly pointing at the bronze path. Now it correctly points at gold. Three new bronze aliases are added.

**Consistency nit:** The naming pattern for bronze aliases is inconsistent — `bronze_alerts` (prefix `bronze_`) vs `rules_bronze` and `typologies_bronze` (suffix `_bronze`). Cosmetic; pick one. See [Issue 5](#issue-5--inconsistent-bronze-alias-naming-in-gold_paths).

---

### 6. Query.ipynb — tenant-scoped Hudi loader

Adds a new cell:

```python
TENANT_FILTER_VALUE = "TAZAMA"
TENANT_FILTER_COLUMNS = ("tenant_id", "tx_tenant_id", "tenantid")

def load_tenant_hudi(path, **options):
    reader = spark.read.format("hudi")
    for key, value in options.items():
        reader = reader.option(key, value)
    df = reader.load(path)
    tenant_col = next((c for c in TENANT_FILTER_COLUMNS if c in df.columns), None)
    if tenant_col is None:
        raise ValueError(
            f"Hudi table at {path} has no tenant filter column; "
            f"expected one of {TENANT_FILTER_COLUMNS}"
        )
    return df.filter(df[tenant_col] == TENANT_FILTER_VALUE)
```

And reworks the example cell to build the table path from `WAREHOUSE_ROOT` and use `load_tenant_hudi`. Correct in isolation.

**Issue:** `TENANT_FILTER_VALUE = "TAZAMA"` is hardcoded. `WAREHOUSE_ROOT` in the same notebook uses `os.environ.get(...)`. CodeRabbit flagged this on 2026-07-08 and no follow-up commit addressed it. See [Issue 2](#issue-2--tenant_filter_value-hardcoded-in-query-and-dashboard-notebooks).

---

### 7. Lakehouse_Catalog.ipynb — inline documentation comments

Five one-liner comments added before existing function definitions and Hudi loads. No code change. ✅

---

### 8. Dashboard notebooks — regen, tenant filters, deletions

- `Dashboard_Metrics.ipynb` (1990 lines) is deleted. Release notes call this "obsolete."
- `Executive_Overview_Dashboard-bk.ipynb` (6493 lines) is deleted — a backup file that should not have been in the repo.
- Seven active dashboard notebooks are re-saved with cleared outputs, `execution_count: null`, added leading blank lines in cells, and `TENANT_FILTER_VALUE` plumbing (per commit `d4bf427`, `fix: added multi tenancy filters for all dashboard notebooks`). The functional filter additions are indistinguishable from the mass cell-metadata churn without opening each notebook.

Because these are notebooks and rendering the JSON diff line-by-line is not productive, the review does not enumerate every cell change. The important architectural change — a per-notebook hardcoded tenant — is captured in [Issue 2](#issue-2--tenant_filter_value-hardcoded-in-query-and-dashboard-notebooks).

[↑ Back to top](#pr-review-biar-107--fix-changes-in-notebooks)

---

## Code Quality Analysis

### Strengths

- **Root-cause epoch-ms fix.** `Account_HolderETL` and `CommentsETL` now match the convention already established in `EntityETL.py`. The `Account_HolderETL` change also hoists the timestamp expression into a local variable, avoiding a duplicate `F.to_timestamp` call.
- **Defensive rules-metadata handling.** `_prepare_rule_metadata` returns `None` when bronze rules are absent, and `_build_rules` treats the join as optional — the view still builds if no metadata is available.
- **De-duplication before join.** `rule_meta` is row-numbered by `ingested_at_ts DESC` per (tenant, rule_id, rule_cfg) before the join, avoiding fan-out from historical rule versions.
- **Path registry bug fixed.** `"typologies"` in `GOLD_PATHS` was pointing at the bronze path; now correctly points at gold.
- **Housekeeping.** Deleting the `-bk.ipynb` backup file is right. Cleared notebook outputs mean cleaner future diffs.

### Issues and Observations

#### Issue 1 — Stale inline comment in CommentsETL.py

**Severity: Minor (Code Quality)**

`automation-orchestrator/Table_ETLs/CommentsETL.py:58` still says:

```python
# created_at / updated_at are stored as epoch microseconds
created_ts = F.to_timestamp((F.col("created_at").cast("double") / F.lit(1_000)))
updated_ts = F.to_timestamp((F.col("updated_at").cast("double") / F.lit(1_000)))
```

The divisor changed from `1_000_000` to `1_000`, but the comment still claims "microseconds." CodeRabbit flagged this on 2026-07-09 and the head commit `84b894c` did not update it. Trivial fix:

```diff
-        # created_at / updated_at are stored as epoch microseconds
+        # created_at / updated_at are stored as epoch milliseconds
```

#### Issue 2 — TENANT_FILTER_VALUE hardcoded in Query and dashboard notebooks

**Severity: Minor (Maintainability)**

`JupyterHub/notebooks/Query.ipynb` defines `TENANT_FILTER_VALUE = "TAZAMA"` as a literal. The same notebook derives `WAREHOUSE_ROOT` from an env var. Deployments serving a different tenant must edit every notebook. CodeRabbit flagged this on the Query notebook on 2026-07-08 and it applies to all seven dashboard notebooks touched in commit `d4bf427`.

Fix pattern (already used for `WAREHOUSE_ROOT`):

```python
TENANT_FILTER_VALUE = os.environ.get("TENANT_FILTER_VALUE", "TAZAMA")
```

Non-blocking because deployments today are single-tenant, but the whole point of the "multi tenancy filters" commit is undermined by hardcoding the tenant.

#### Issue 3 — `typology_name` silently removed from Gold schema

**Severity: Major (Breaking Change Risk / Data Integrity)**

`TypologiesETL.gold()` no longer projects `F.col("typology_name")`. The PR description does not mention this schema change. Any downstream consumer of the `gold/typologies` Hudi table that reads `typology_name` (CMS, an Athena/Trino view, an operator query) will fail after this deploys.

I grepped the biar repo — no other Python file references `typology_name`, so within biar this is safe. But `gold/typologies` is intended to be queried via `lakehouse_query_api` and beyond (case-management-system, other services). This needs to be verified with the owning team before merge, or the column should be preserved.

If the removal is deliberate, the PR description should say so explicitly and note the schema version bump.

**Fix (if removal not deliberate):** Restore both lines:

```python
# in silver()
.withColumn("typology_name", F.col("typology_obj").getField("typology_name"))

# in gold() select
F.col("typology_name").cast("string"),
```

#### Issue 4 — `pk` change on alerts_nav_rules creates Hudi key drift

**Severity: Minor (Data Integrity)**

`_build_rules` extends the `pk` inputs by `typology_cfg` and `rule_cfg`. Because Hudi upserts on `pk` (the record key), the same logical (alert, typology, rule) row will now be written under a **different** key than any pre-existing row. On the next run against a table that already has rows from the old scheme, the view will have two entries for the same underlying alert-rule: one with the old `pk`, one with the new. Downstream consumers deduplicating by `pk` will see duplicates until the old rows are purged or a compaction rewrites them.

If the view is fully rebuilt on each run (which the ETL name "AlertNavigatorETL" suggests, but I could not confirm from the diff alone), this is a non-issue. If it's an upsert-in-place table, a one-shot cleanup or a schema-version comment is warranted.

**Ask author:** is `alerts_nav_rules` rebuilt from scratch each run, or upserted?

#### Issue 5 — Inconsistent bronze alias naming in GOLD_PATHS

**Severity: Informational (Code Quality)**

```python
"bronze_alerts":     alerts_bronze_path,
"rules_bronze":      rules_bronze_path,
"typologies_bronze": typologies_bronze_path,
```

Prefix vs suffix — pick one and stick with it. Not blocking, but future keys will inherit whichever pattern is set.

#### Issue 6 — Whitespace-only churn in TypologiesETL.py

**Severity: Informational**

Roughly half the file's diff is trailing-whitespace and blank-line reflows that add nothing to the review. Preserving the churn buries the three real changes (permissive mode, `.getField(...)`, dropped `typology_name`). Not blocking, but making surgical PRs easier to review is worth a line in a style guide.

#### Issue 7 — Extra JSON columns on alerts_nav_rules view without documentation

**Severity: Informational**

`_prepare_rule_metadata` emits `band_reasons_json`, `band_sub_rule_refs_json`, `band_reasons_with_sub_rule_refs_json`, `exit_condition_reasons_json`, `exit_condition_sub_rule_refs_json`, `exit_condition_reasons_with_sub_rule_refs_json`, `rule_desc`, `band_count`, `exit_condition_count` — nine new columns on the resulting view. None are dropped before the final `return`. If the CMS-facing consumer only needs `matched_band_reason` and `matched_exit_condition_reason`, most of these are dead weight; if it needs the full JSON arrays, this is fine but should be called out.

[↑ Back to top](#pr-review-biar-107--fix-changes-in-notebooks)

---

## Security Assessment

| Concern | Assessment |
|---------|-----------|
| Injection (Spark SQL / JSON schema inference) | `configuration_json` is parsed via `F.from_json` against an inferred schema. This is Spark's normal data-plane pattern and is not user-facing. No new injection surface. |
| Tenant isolation | The new `load_tenant_hudi` helper hard-filters by `TENANT_FILTER_VALUE` at load time. This is a scoping helper, not an authorization boundary — anyone with notebook access can rewrite the constant. That is pre-existing behaviour of a Jupyter analytics environment and not introduced by this PR. |
| Rule-metadata `DEFAULT` fallback | Uses upper-cased literal `"DEFAULT"`. No path allows an attacker to inject a metadata row under `tenant_id = "DEFAULT"` unless they already have write access to the `bronze/rule` layer. Pre-existing trust model. |
| Notebook outputs cleared | Cleared cell outputs remove any previously-embedded data from the repo history going forward. Positive privacy signal, though prior commits still contain the outputs. |

No new security vulnerabilities introduced by this PR.

[↑ Back to top](#pr-review-biar-107--fix-changes-in-notebooks)

---

## Test Coverage

- **No unit tests added or modified.** The repository does not appear to have a Python test suite for `automation-orchestrator/Table_ETLs/` (a quick scan shows no `tests/` under that path). The epoch-ms fix, the rule-metadata join, and the `pk` change are all untested at merge time.
- **PR checklist:** `Development Environment` is checked; `Locally`, `Not needed`, `Husky successfully run`, and `Unit tests passing and Documentation done` are unchecked.
- **No CI evidence or coverage screenshot attached.**
- The `alert_navigator` join change is the largest functional delta. It has no test. Given the `pk` schema change and the possibility of downstream schema drift on `gold/typologies`, this warrants either (a) a manual verification script showing before/after row counts and `matched_rule_reason` populations, or (b) an integration test against a synthetic bronze/rule input.

Call it out: the two epoch-ms bugs would have been caught trivially by a unit test that fed one known epoch-ms value and asserted the resulting timestamp. The absence of such a test is why the same class of bug had to be fixed in two files in one PR.

[↑ Back to top](#pr-review-biar-107--fix-changes-in-notebooks)

---

## CodeRabbit Activity

### Pass 1 — Initial scan of commit `c6731af`

**Commit reviewed:** `c6731af1eaa6fa15d6f14bff8841e79404f51b02` (2026-07-06)
**Findings:** 1 actionable + 2 nitpicks

| Finding | Severity | Status |
|---------|----------|--------|
| Hardcoded `EFRuP@1.0.0` in `ALertsETL.py` | Nitpick (Maintainability) | ❌ Not resolved — file not touched in this PR |
| Cross-join risk on `all` granularity in `ConditionsTimelineView.py` | Nitpick (Performance) | ❌ Not resolved — file not touched (out of scope) |
| Substring-vs-token match on `event_types_csv` in `ConditionsTimelineView.py` | Actionable | ❌ Not resolved — file not touched (out of scope) |

*Both `ALertsETL.py` and `ConditionsTimelineView.py` are pre-existing files that CodeRabbit noticed in the change set of an unrelated merge commit. They are legitimately out of scope for this PR.*

### Pass 2 — Duplicate posting of Pass 1 (~15s later)

Same findings; posted twice due to a GitHub rate-limit error message ("Inline review comments failed to post"). No new items.

### Pass 3 — Review of commit `d4bf427` (multi-tenancy filters)

**Commit reviewed:** `d4bf4271d0a497dc852ed40abcd1f456bb89899f` (2026-07-08)
**Findings:** 1 nitpick

| Finding | Severity | Status |
|---------|----------|--------|
| `TENANT_FILTER_VALUE = "TAZAMA"` hardcoded in `Query.ipynb` | Nitpick (Maintainability) | ❌ Not resolved — see [Issue 2](#issue-2--tenant_filter_value-hardcoded-in-query-and-dashboard-notebooks) |

### Pass 4 — Review of commit `84b894c` (CommentsETL epoch-ms fix)

**Commit reviewed:** `84b894c86ff15fd703d4228deecf859ac0f952a2` (2026-07-09)
**Findings:** 1 minor

| Finding | Severity | Status |
|---------|----------|--------|
| Stale `epoch microseconds` comment in `CommentsETL.py` | Minor (Code Quality) | ❌ Not resolved — see [Issue 1](#issue-1--stale-inline-comment-in-commentsetlpy) |

[↑ Back to top](#pr-review-biar-107--fix-changes-in-notebooks)

---

## Summary and Verdict

**Verdict: Changes Requested**

The two epoch-ms fixes and the alert-navigator rule-metadata join are welcome and mostly correct. However, the PR silently removes the `typology_name` column from `gold/typologies` without mentioning it in the description — a real risk of breaking external consumers. Two open CodeRabbit findings (stale comment, hardcoded tenant) also remain unaddressed on the head commit, and the "multi-tenancy" filter is functionally single-tenant because the value is a literal.

The rule-metadata join changes the `pk` composition of `alerts_nav_rules`; this needs a one-line answer from the author about whether the view is rebuilt each run or upserted, before merge.

### Blocking

1. **`typology_name` silently dropped from Gold** — either restore the column or confirm no external consumer reads it and call out the schema change in the PR description. See [Issue 3](#issue-3--typology_name-silently-removed-from-gold-schema).
2. **Confirm `alerts_nav_rules` rebuild behaviour** — the `pk` inputs changed. If the table is upsert-in-place, old and new rows will co-exist under different keys. See [Issue 4](#issue-4--pk-change-on-alerts_nav_rules-creates-hudi-key-drift).

### Non-blocking but recommended

3. **Stale comment in CommentsETL.py:58** — change "microseconds" → "milliseconds". Trivial. See [Issue 1](#issue-1--stale-inline-comment-in-commentsetlpy).
4. **Read `TENANT_FILTER_VALUE` from env var** in Query.ipynb and the seven dashboard notebooks. Matches the existing `WAREHOUSE_ROOT` pattern. See [Issue 2](#issue-2--tenant_filter_value-hardcoded-in-query-and-dashboard-notebooks).
5. **Naming consistency for bronze aliases** in `lakehouse_query_api.py`. See [Issue 5](#issue-5--inconsistent-bronze-alias-naming-in-gold_paths).
6. **Drop unused JSON payload columns** from the alerts_nav_rules view, or document why they are exposed. See [Issue 7](#issue-7--extra-json-columns-on-alerts_nav_rules-view-without-documentation).
7. **Complete the PR checklist** (unit tests, husky, docs). At minimum, add a note in the PR description covering the schema deltas on `gold/typologies` and `alerts_nav_rules`.

[↑ Back to top](#pr-review-biar-107--fix-changes-in-notebooks)

---

## GitHub Review Comment

````markdown
**Changes Requested**

Two epoch-ms fixes and the new alert-navigator rule-metadata join are mostly good, but the PR silently drops the `typology_name` column from `gold/typologies` and changes the `pk` composition of `alerts_nav_rules`, and two prior CodeRabbit findings are still unaddressed on the head commit.

---

### Blocking

**1. `typology_name` silently dropped from `gold/typologies`**

`automation-orchestrator/Table_ETLs/TypologiesETL.py` removes `typology_name` from both the Silver `withColumn` chain and the Gold `select`:

```diff
-.withColumn("typology_name",       F.col("typology_obj.typology_name"))
...
-F.col("typology_name").cast("string"),
```

Nothing in `biar` still references it, but this table is exposed via `lakehouse_query_api.GOLD_PATHS["typologies"]` and is intended to be read by CMS / query consumers. The PR description does not mention the schema change. Either:

- restore the column (and use `.getField("typology_name")` to match the new access pattern), or
- confirm with the CMS / query-API consumers that nothing reads `typology_name`, and call the removal out explicitly in the PR description.

**2. `pk` composition changed on `alerts_nav_rules` — confirm rebuild behaviour**

`_build_rules` in `automation-orchestrator/Table_ETLs/alert_navigator.py` now includes `typology_cfg` and `rule_cfg` in the `pk` inputs:

```diff
 F.col("alert_id"),
 F.col("typology_id"),
+F.coalesce(F.col("typology_cfg"), F.lit("")),
 F.col("rule_id"),
+F.coalesce(F.col("rule_cfg"), F.lit("")),
 F.coalesce(F.col("rule_sub_ref"), F.lit("")),
```

If `alerts_nav_rules` is a Hudi upsert-in-place target, the same logical row will now be written under a new key and coexist with the old one. Please confirm the view is rebuilt from scratch each run, or add a cleanup / schema-version note.

---

### Non-blocking (please address in this PR if possible)

**3. Stale comment in `automation-orchestrator/Table_ETLs/CommentsETL.py:58`**

You changed the divisor from `1_000_000` to `1_000`, but the comment above still says "microseconds":

```diff
-        # created_at / updated_at are stored as epoch microseconds
+        # created_at / updated_at are stored as epoch milliseconds
         created_ts = F.to_timestamp((F.col("created_at").cast("double") / F.lit(1_000)))
         updated_ts = F.to_timestamp((F.col("updated_at").cast("double") / F.lit(1_000)))
```

CodeRabbit flagged this on 2026-07-09.

**4. `TENANT_FILTER_VALUE` is hardcoded across all "multi-tenancy" notebooks**

`JupyterHub/notebooks/Query.ipynb` and the seven dashboard notebooks in commit `d4bf427` all set:

```python
TENANT_FILTER_VALUE = "TAZAMA"
```

`WAREHOUSE_ROOT` in the same notebook already uses the env-var pattern. Apply it here too so the "multi-tenancy" filter actually supports multiple tenants:

```python
TENANT_FILTER_VALUE = os.environ.get("TENANT_FILTER_VALUE", "TAZAMA")
```

**5. Inconsistent bronze alias naming in `datalakehouse-api/lakehouse_query_api.py`**

```python
"bronze_alerts":     alerts_bronze_path,
"rules_bronze":      rules_bronze_path,
"typologies_bronze": typologies_bronze_path,
```

Pick one convention (prefix or suffix) so future entries are predictable.

**6. Unused JSON payload columns on `alerts_nav_rules`**

`_prepare_rule_metadata` emits `band_reasons_json`, `band_sub_rule_refs_json`, `band_reasons_with_sub_rule_refs_json`, `exit_condition_reasons_json`, `exit_condition_sub_rule_refs_json`, `exit_condition_reasons_with_sub_rule_refs_json`, `rule_desc`, `band_count`, `exit_condition_count` — none are dropped before the join result is returned. If the CMS consumes them directly, great; if not, please drop them or document that they are intentional payload.

**7. PR checklist / description**

Please tick the boxes that apply and add a note about the two schema deltas (`gold/typologies` losing `typology_name`, `alerts_nav_rules` `pk` composition) in the description.
````

[↑ Back to top](#pr-review-biar-107--fix-changes-in-notebooks)
