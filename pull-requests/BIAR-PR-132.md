# PR Review: BIAR #132 — fix: Implement regex-validation in SQL query for EFRuP version.

**Repo:** tazama-lf/biar
**Branch:** `paysys/fixEFRuP` → `dev`
**Author:** MAdeel95 (Muhammad Adeel)
**Date Reviewed:** 2026-07-17
**Label:** bug
**Size:** +9 / -14 lines across 2 files
**Commits:** 3 (6bdec9a6, 4e9cd251, 90ea36d8)
**State:** OPEN
**HEAD SHA verified:** `90ea36d838e7d18146baf649a94a1de7548a3989`
**Existing approvals:** none — CodeRabbit walkthrough posted but full review was rate-limited (next review in ~27min at the time of walkthrough); no human approval yet

## Table of Contents

- [Overview](#overview)
- [What Changed (Detailed)](#what-changed-detailed)
  - [1. `automation-orchestrator/Table_ETLs/ALertsETL.py` — replace exact-version equality with anchored RLIKE](#1-automation-orchestratortable_etlsalertsetlpy--replace-exact-version-equality-with-anchored-rlike)
  - [2. `JupyterHub/notebooks/Rule_Discovery.ipynb` — import reshuffle and `except:` → `except Exception:`](#2-jupyterhubnotebooksrule_discoveryipynb--import-reshuffle-and-except--except-exception)
  - [3. Deleted commit — Spark ETL regression tests added then removed](#3-deleted-commit--spark-etl-regression-tests-added-then-removed)
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

Prior to this PR, the alerts ETL (`AlertsETL._flatten_silver`) extracted the EFRuP subRuleRef using an exact-version equality check: `r.id = 'EFRuP@1.0.0'`. Any bumped EFRuP version (e.g. `EFRuP@2.0.0`) would silently fall out of the filter, and `efrup_subruleref` would be `null` for those alerts — feeding a downstream null into the gold layer that other systems (see the sibling CMS PR 254) then read.

This PR replaces the equality with `r.id RLIKE '^EFRuP@[0-9]+\.[0-9]+\.[0-9]+$'` — anchored, strict semver (three numeric groups). Any `EFRuP@X.Y.Z` will now be matched; identifiers with pre-release/build suffixes (`EFRuP@1.0.0-beta`), two-part versions (`EFRuP@1.0`), or lookalike prefixes (`EFRuP2@1.0.0`) are still rejected.

The PR also bundles two unrelated changes: an import reshuffle in `Rule_Discovery.ipynb` and a lint-driven `except: → except Exception:` sweep in the same notebook. A short-lived third change added Spark-based regression tests for the RLIKE (commit `6bdec9a6`) but the second commit (`4e9cd251`, "delete test files") removed all of them — the fix ships without direct test coverage.

Targets `dev` — correct for this repo's flow. All 16 CI checks are `SUCCESS` on HEAD `90ea36d8` (CodeQL, njsscan, nodejsscan, dependency-review, TS build/lint/test, Python lint, docker build, hadolint, encoding, DCO, GPG-verify, conventional-commit title, CodeRabbit status).

| File | Nature of Change |
|------|-----------------|
| `automation-orchestrator/Table_ETLs/ALertsETL.py` | Replace `AND r.id = 'EFRuP@1.0.0'` with `AND r.id RLIKE '^EFRuP@[0-9]+\.[0-9]+\.[0-9]+$'` in the `efrup_subruleref` extraction. |
| `JupyterHub/notebooks/Rule_Discovery.ipynb` | Trim unused pyspark.sql.types imports; move `from IPython.display import HTML` next to its consumer; replace six bare `except:` clauses with `except Exception:`. No functional change to notebook logic. |

**Companion PR:** [tazama-lf/case-management-system#254](https://github.com/tazama-lf/case-management-system/pull/254) — same subject on the CMS side. Note the semantics differ: CMS uses prefix match (`startsWith('EFRuP@')` / `LIKE 'EFRuP@%'`) with no numeric constraint, while BIAR uses strict-semver regex. See Issue 2.

[↑ Back to top](#pr-review-biar-132--fix-implement-regex-validation-in-sql-query-for-efrup-version)

---

## What Changed (Detailed)

### 1. `automation-orchestrator/Table_ETLs/ALertsETL.py` — replace exact-version equality with anchored RLIKE

```diff
             transform(
                 filter(
                     t.ruleResults,
                     r -> r is not null
-                        AND r.id = 'EFRuP@1.0.0'
+                        AND r.id RLIKE '^EFRuP@[0-9]+\\.[0-9]+\\.[0-9]+$'
                 ),
                 r -> r.subRuleRef
             )
```

Context — the change sits inside the `efrup_subruleref` column expression on the silver-alerts write:

```python
.withColumn(
"efrup_subruleref",
F.expr("""
    element_at(
        flatten(
            transform(
                filter(
                    alert_data_obj.tadpResult.typologyResult,
                    t -> t is not null
                ),
                t -> transform(
                    filter(
                        t.ruleResults,
                        r -> r is not null
                            AND r.id RLIKE '^EFRuP@[0-9]+\\.[0-9]+\\.[0-9]+$'
                    ),
                    r -> r.subRuleRef
                )
            )
        ),
        1
    )
""")
)
```

Regex analysis — the pattern `^EFRuP@[0-9]+\.[0-9]+\.[0-9]+$` is a strict semver-triplet match:

| Input | Matches? | Notes |
|-------|----------|-------|
| `EFRuP@1.0.0` | ✅ | Baseline |
| `EFRuP@2.0.0` | ✅ | Target of the fix |
| `EFRuP@10.20.30` | ✅ | Multi-digit versions supported |
| `EFRuP@1.0` | ❌ | Two-part versions rejected |
| `EFRuP@1.0.0-beta` | ❌ | Pre-release suffix rejected |
| `EFRuP@1.0.0.1` | ❌ | Four-part versions rejected |
| `EFRuP2@1.0.0` | ❌ | Prefix drift rejected |
| `xEFRuP@1.0.0` | ❌ | Anchors prevent substring match |

Escaping — the Python triple-quoted string `'^EFRuP@[0-9]+\\.[0-9]+\\.[0-9]+$'` becomes `^EFRuP@[0-9]+\.[0-9]+\.[0-9]+$` in the SQL expression that Spark evaluates via `java.util.regex.Pattern`. `\.` matches a literal `.`, which is what's wanted (an unescaped `.` would also match under strict-semver structure because `[0-9]+` on either side leaves no room for a non-`.` character, but the explicit escape is correct and readable).

Runtime characteristics — `RLIKE` on Spark uses Java regex. The pattern is anchored and non-backtracking (`[0-9]+` runs are greedy against a bounded character class), so there is no catastrophic-backtracking exposure. Compiled once per query, applied per rule row in the transform.

### 2. `JupyterHub/notebooks/Rule_Discovery.ipynb` — import reshuffle and `except:` → `except Exception:`

Two categories of unrelated cleanup:

**Import trimming:**
```diff
 from pyspark.sql import functions as F
-from pyspark.sql.window import Window
-from pyspark.sql.types import (
-    StructType, StructField, StringType, LongType,
-    TimestampType, DoubleType
-)
+from pyspark.sql.types import TimestampType
 import pandas as pd
```

`Window`, `StructType`, `StructField`, `StringType`, `LongType`, `DoubleType` were imported but not referenced in the notebook. Also moves `from IPython.display import HTML` from mid-cell (line ~532 in the original) up to the top of the cell where the display helpers live. Purely cosmetic; no functional impact.

**Bare `except:` → `except Exception:`:**
```diff
-    except:
+    except Exception:
         pass
```

Six occurrences, all in `try` blocks that were catching *any* exception (including `KeyboardInterrupt` and `SystemExit`) and swallowing it. Replacing with `except Exception:` narrows the catch to non-system-exit exceptions — a strictly-better pattern and typically a linter (`bare-except` / `E722`) requirement. Behaviour under normal exceptions is unchanged.

One case also drops the unused binding `as e`:
```diff
-    except Exception as e:
+    except Exception:
         return pd.DataFrame(), 0, f"BirthDt value '{birth_dt_raw}' is a placeholder — real date of birth not available in synthetic data (exit condition .x01)"
```

`e` was unused in the return string, so the rename is safe.

### 3. Deleted commit — Spark ETL regression tests added then removed

Commit `6bdec9a6` initially added:

```
automation-orchestrator/pytest.ini               |   3 +
automation-orchestrator/requirements-test.txt    |   3 +
automation-orchestrator/tests/conftest.py        |  28 ++
automation-orchestrator/tests/test_alerts_etl_efrup.py | 181 +++++++++
```

The added test file explicitly targeted the regex fix — a docstring on the module reads:

> Regression tests for the efrup_subruleref extraction ... Covers the GitHub issue scenarios for the RLIKE fix at ALertsETL.py:225 ... which replaced an exact-version match that only recognised EFRuP@1.0.0 and missed EFRuP@2.0.0 (and any other version).

Commit `4e9cd251` ("delete test files") then removed all four files. The final PR therefore ships the RLIKE fix without any spec-level coverage of the regex behaviour. See Issue 3 in Code Quality Analysis and the Test Coverage section — this is the single most consequential defect of the PR.

[↑ Back to top](#pr-review-biar-132--fix-implement-regex-validation-in-sql-query-for-efrup-version)

---

## Code Quality Analysis

### Strengths

- **Regex is correct and anchored.** `^EFRuP@[0-9]+\.[0-9]+\.[0-9]+$` covers the bumped-version case, rejects prefix drift, and is not backtracking-vulnerable.
- **Escape choice matches the Spark SQL string context.** `\\.` in the Python triple-quoted string is the right level of escaping for a Java-regex `\.`.
- **Bare-except sweep is a genuine hygiene improvement.** Six occurrences of `except:` were catching `KeyboardInterrupt` and `SystemExit`; narrowing to `except Exception:` is standard best-practice and typically a lint requirement.
- **Import trimming is legitimate.** The dropped pyspark types were unreferenced; leaving them in place would rot.

### Issues and Observations

#### Issue 1 — `Rule_Discovery.ipynb:510` still uses exact-version equality on `EFRuP@1.0.0`

**Severity: Major (Bug — parallel-siblings drift)**

The notebook this PR touches contains a sibling comparison that was **not** updated:

```python
# JupyterHub/notebooks/Rule_Discovery.ipynb:510 (inside extract_rule_results)
for r in t.get("ruleResults", []):
    rule_id = r["id"]

    # Extract EFRuP separately instead of just skipping it
    if rule_id == "EFRuP@1.0.0":
        efrp_ref = r.get("subRuleRef", "none")
        continue

    # Keep highest weight version if rule appears in multiple typologies
    if rule_id not in all_rules or r.get("wght", 0) > all_rules[rule_id]["weight"]:
        all_rules[rule_id] = { ... }
```

Two consequences when the environment sees an `EFRuP@2.0.0` (or any bumped) rule:

1. **`efrp_ref` stays `"none"`.** `display_efrp_status(efrp_ref)` (defined a few cells later) renders the EFRuP status header from this value; with the exact-match still in place, the notebook UI will misreport EFRuP status as absent.
2. **The bumped EFRuP row leaks into `all_rules`** because the `continue` is not reached. It is then rendered in the generic rule-results table, which is not the notebook's intended presentation for EFRuP (which has a dedicated status cell above the table).

This is exactly the scenario the parallel-siblings hunt (Section 3.1) catches: the PR touched this same file for other cleanup, so a reader might reasonably expect all EFRuP references to have been considered, but this one was missed.

**Fix — mirror the ETL semantics with `.startswith("EFRuP@")` (looser) or a regex-equivalent check:**

```python
# Option A — prefix match, matches CMS PR 254 semantics (recommended for consistency)
if rule_id.startswith("EFRuP@"):
    efrp_ref = r.get("subRuleRef", "none")
    continue

# Option B — strict-semver regex, matches this PR's ETL semantics exactly
import re
_EFRUP_RE = re.compile(r"^EFRuP@[0-9]+\.[0-9]+\.[0-9]+$")
...
if _EFRUP_RE.match(rule_id):
    efrp_ref = r.get("subRuleRef", "none")
    continue
```

Option A is preferred — see Issue 2 for why strict-semver-vs-prefix should be aligned across the platform.

#### Issue 2 — Semantic divergence with the sibling CMS PR (#254)

**Severity: Minor (Consistency / Breaking-change risk)**

The sibling PR [tazama-lf/case-management-system#254](https://github.com/tazama-lf/case-management-system/pull/254) uses a **prefix match** for the same conceptual identifier: `LIKE 'EFRuP@%'` in SQL and `.startsWith('EFRuP@')` in TS. This PR uses a **strict-semver regex**. That means the two systems disagree on the acceptance of at least these identifiers:

| Identifier | BIAR (this PR) | CMS PR 254 |
|------------|----------------|------------|
| `EFRuP@1.0.0-beta` | ❌ rejected | ✅ accepted |
| `EFRuP@1.0` | ❌ rejected | ✅ accepted |
| `EFRuP@2.0.0-rc1` | ❌ rejected | ✅ accepted |
| `EFRuP@2` | ❌ rejected | ✅ accepted |

If the rule-engine ever ships an EFRuP identifier that BIAR rejects but CMS accepts, BIAR's silver layer will null the `efrup_subruleref` for that alert, while the CMS UI will still try to render an EFRuP badge from the rule results — producing inconsistent user-facing state. Prefix matching is the safer floor.

**Recommendation:** loosen this PR's regex to a prefix check to align with CMS, or if strict semver is a deliberate contract, coordinate with the CMS side to tighten it there too. Either way, the two implementations should not disagree.

If keeping the strict form is deliberate (e.g. to reject an accidental `EFRuP@debug` string), document that contract in a comment above the RLIKE.

#### Issue 3 — No test coverage for the RLIKE fix (regression tests were added and then deleted)

**Severity: Major (Test Coverage — for the load-bearing change)**

The initial commit `6bdec9a6` added `automation-orchestrator/tests/test_alerts_etl_efrup.py` (181 lines) — a Spark-based regression suite whose module docstring explicitly said:

> Regression tests for the efrup_subruleref extraction ... Covers the GitHub issue scenarios for the RLIKE fix at ALertsETL.py:225 ... which replaced an exact-version match that only recognised EFRuP@1.0.0 and missed EFRuP@2.0.0.

The second commit `4e9cd251` deleted the tests along with `pytest.ini`, `requirements-test.txt`, and `conftest.py`. The commit message ("delete test files") does not explain the rationale.

The PR now ships the load-bearing regex change with **no spec-level assertion** that:

- `EFRuP@2.0.0` produces a non-null `efrup_subruleref` (the whole point of the fix),
- `EFRuP@1.0.0` still works (regression on the baseline),
- `EFRuP@1.0.0-beta` is rejected (regex contract),
- prefix drift like `EFRuP2@1.0.0` is rejected (regex contract).

If a future maintainer relaxes the regex to `EFRuP@.*` or tightens it to `EFRuP@1\..*` there is nothing to fail the build.

**Ask:** either restore the deleted test file (the diff is preserved in commit `6bdec9a6`, so `git show 6bdec9a6:automation-orchestrator/tests/test_alerts_etl_efrup.py` yields the whole file), or explain in the PR description why Spark-based tests are inappropriate for this environment and provide an alternative — a pure-Python regex-contract unit test at minimum:

```python
# automation-orchestrator/tests/test_efrup_regex.py
import re

EFRUP_PATTERN = r"^EFRuP@[0-9]+\.[0-9]+\.[0-9]+$"

class TestEfrupRegex:
    def test_baseline_version_matches(self):
        assert re.match(EFRUP_PATTERN, "EFRuP@1.0.0")

    def test_bumped_version_matches(self):
        assert re.match(EFRUP_PATTERN, "EFRuP@2.0.0")

    def test_multi_digit_version_matches(self):
        assert re.match(EFRUP_PATTERN, "EFRuP@10.20.30")

    def test_prerelease_rejected(self):
        assert not re.match(EFRUP_PATTERN, "EFRuP@1.0.0-beta")

    def test_two_part_version_rejected(self):
        assert not re.match(EFRUP_PATTERN, "EFRuP@1.0")

    def test_prefix_drift_rejected(self):
        assert not re.match(EFRUP_PATTERN, "EFRuP2@1.0.0")

    def test_substring_rejected(self):
        assert not re.match(EFRUP_PATTERN, "xEFRuP@1.0.0")
```

This keeps the contract in place even if a Spark test harness isn't available.

#### Issue 4 — Scope-expansion: unrelated notebook cleanup bundled with the ETL bug fix

**Severity: Informational (Maintainability)**

The notebook changes (import reshuffle, `except:` → `except Exception:`) are unrelated to the RLIKE fix. Two separate PRs would have been cleaner: (a) the ETL regex fix (with tests), (b) notebook lint cleanup. Bundling them makes a future `git bisect` land on the same PR for two very different classes of change. Not blocking; noted for future.

#### Issue 5 — PR title uses trailing period, contra conventional-commit style commonly used in the repo

**Severity: Informational (Style)**

`fix: Implement regex-validation in SQL query for EFRuP version.` — the trailing period and capitalised `Implement` after `fix:` are minor style drift versus the pattern used in most other merges (`fix: implement ...` — lowercase, no trailing period). The conventional-commit CI check passes, so this is not blocking. Optional cleanup on rebase.

[↑ Back to top](#pr-review-biar-132--fix-implement-regex-validation-in-sql-query-for-efrup-version)

---

## Security Assessment

| Concern | Assessment |
|---------|-----------|
| SQL injection | The regex literal is a compile-time constant embedded in a Spark SQL string; no user input is interpolated. Safe. |
| Regex denial-of-service (ReDoS) | The pattern is anchored, non-alternating, and uses bounded character-class quantifiers (`[0-9]+`). No catastrophic-backtracking exposure on Java regex. Safe. |
| Tenant / role scoping | N/A — this change modifies a data-transformation filter inside an ETL job that already runs under the pipeline's tenancy model. |
| Auth-guard decorators | N/A — no new endpoint. |
| URL construction / SSRF | N/A. |
| HTML rendering / XSS | The notebook writes an EFRuP status header via `IPython.display.HTML(...)` using a `f"<p>...{efrp_ref}...</p>"` template. This is unchanged by the PR, so pre-existing. In the context of a JupyterHub notebook viewed by internal analysts consuming their own bronze/gold alerts, the exposure is contained. Flagging only because Issue 1's fix would flow through the same rendering path — a proper regex-mirror in the notebook does not introduce new HTML injection surface either, since `efrp_ref` still comes from the alert JSON and the notebook is not a multi-tenant web app. |
| Secrets / PII in logs | Unchanged. |
| CI security scanners | CodeQL (both variants), njsscan, nodejsscan, dependency-review all `SUCCESS`. |

No new security vulnerabilities introduced.

[↑ Back to top](#pr-review-biar-132--fix-implement-regex-validation-in-sql-query-for-efrup-version)

---

## Test Coverage

**What is tested:** Nothing — the regression tests that were originally added for this fix (commit `6bdec9a6`, `automation-orchestrator/tests/test_alerts_etl_efrup.py`) were removed in commit `4e9cd251`. The final PR contains no new spec-level coverage.

**What is not tested (that should be):**

- The regex accepting `EFRuP@2.0.0` (the whole point of the fix).
- The regex still accepting `EFRuP@1.0.0` (baseline regression guard).
- The regex rejecting pre-release / two-part / prefix-drift variants (regex-contract regression guard).
- The notebook's `extract_rule_results` behaviour when an `EFRuP@2.0.0` row is present (would surface Issue 1 above; there's no existing test for `extract_rule_results` either, but it's an area that would benefit).

**PR checklist:** The PR body has no explicit checklist. The linked issue is not referenced in the description.

Called out per Section 3.1 test-hunt: **every `Major` in Issues must have a matching test.** Issue 1 (parallel-siblings drift in Rule_Discovery.ipynb) and Issue 3 (regex contract) both need locking-in tests. Their absence elevates each of them into the Blocking list — see Summary and Verdict.

[↑ Back to top](#pr-review-biar-132--fix-implement-regex-validation-in-sql-query-for-efrup-version)

---

## CodeRabbit Activity

### Pass 1 — walkthrough only, review rate-limited on commit `6bdec9a6`

**Commit reviewed:** `6bdec9a6` (only for the walkthrough summary)
**Findings:** 0 actionable comments (the review itself was rate-limited)

CodeRabbit posted the change-stack walkthrough and pre-merge check summary but was not able to run its usual line-by-line review — the rate-limit notice reads *"Next review available in: 27 minutes"*. No further passes have run after subsequent commits `4e9cd251` and `90ea36d8`. The walkthrough is informational only; it did not flag the two Majors below.

**Reconciliation with independent hunts:**

- **Parallel-siblings hunt.** Ran `grep -rn "EFRuP@" repos/biar --include="*.py" --include="*.ipynb"`. Two production sites match: (a) `automation-orchestrator/Table_ETLs/ALertsETL.py:225` (fixed by this PR) and (b) `JupyterHub/notebooks/Rule_Discovery.ipynb:510` (**not fixed**, see Issue 1). The hunt caught the miss that CodeRabbit's walkthrough did not surface. Adding this to the record because CodeRabbit here is not a review substitute — the walkthrough only *summarises* the changed files, so a sibling in an unchanged region of a touched file is invisible to it.
- **Type/prop drift hunt.** N/A — no TypeScript or type-shape changes in this PR.
- **Label/boundary drift hunt.** No numeric boundary or label change (the regex is a strict subset of the previous exact-string logic; downstream string display of `efrup_subruleref` values is unaffected in terms of format). No drift.
- **Guard/scope asymmetry hunt.** No endpoint changes. N/A.

**Note on rate-limited reviews.** Because the actual CodeRabbit review didn't run, this file's Issues section is the primary safety net. The Section 3.1 hunts caught the two Majors independently; had CodeRabbit run, it likely would have flagged Issue 1 as well.

[↑ Back to top](#pr-review-biar-132--fix-implement-regex-validation-in-sql-query-for-efrup-version)

---

## Summary and Verdict

**Verdict: Changes Requested**

The core regex change in `ALertsETL.py` is correct and safe — anchored, escaped, non-backtracking. The `except: → except Exception:` sweep is a genuine improvement. But two blocking problems drop the verdict:

1. A **parallel EFRuP comparison** in the same touched notebook still hard-codes `"EFRuP@1.0.0"`. It short-circuits EFRuP handling in the Rule_Discovery notebook exactly the way the ETL bug used to short-circuit the silver layer.
2. The **regression tests were added and then deleted** in the second commit with no explanation, so the load-bearing regex ships without a single spec-level assertion of its contract.

Once those two are addressed (fix the notebook sibling and restore or replace the test file), this PR is a clean approval.

### Blocking

1. **Fix the `rule_id == "EFRuP@1.0.0"` sibling in `JupyterHub/notebooks/Rule_Discovery.ipynb:510`.** Replace with `rule_id.startswith("EFRuP@")` (preferred, to align with CMS PR 254) or a regex mirror of the ETL pattern. The current state defeats the fix for the notebook path.
2. **Restore or replace the regression tests deleted by commit `4e9cd251`.** Either resurrect `automation-orchestrator/tests/test_alerts_etl_efrup.py` from the first commit, or provide a lighter pure-Python regex-contract test (example shown in Issue 3) — the fix must not merge without spec-level coverage of the RLIKE contract.

### Non-blocking but recommended

3. **Align semver semantics with CMS PR 254** (Issue 2). CMS uses prefix match; this PR uses strict semver. If the divergence is intentional, comment the SQL to say so; otherwise loosen the regex to a prefix check.
4. **Split unrelated notebook cleanup into a separate PR** (Issue 4). Import reshuffle and bare-except sweep are hygiene changes orthogonal to the ETL regex fix.
5. **PR title tidy-up** (Issue 5). `fix: implement regex-validation in SQL query for EFRuP version` (lowercase, no trailing period) matches the convention used elsewhere.

[↑ Back to top](#pr-review-biar-132--fix-implement-regex-validation-in-sql-query-for-efrup-version)

---

## GitHub Review Comment

````markdown
**Changes Requested**

The core RLIKE regex in `ALertsETL.py` is correct and safe (anchored, escaped, non-backtracking) and the `except: → except Exception:` sweep is a valid hygiene improvement. Two blockers, however:

- The same fix wasn't applied to a sibling comparison in `Rule_Discovery.ipynb` — a bumped EFRuP version will still be mishandled by the notebook.
- The regression test file added in commit `6bdec9a6` was deleted in commit `4e9cd251` without explanation, so the load-bearing regex ships without a spec.

Both are quick to address.

---

### Blocking

**1. `JupyterHub/notebooks/Rule_Discovery.ipynb:510` still hard-codes `EFRuP@1.0.0`**

Inside `extract_rule_results`, the EFRuP extraction is still an exact-string equality:

```python
for r in t.get("ruleResults", []):
    rule_id = r["id"]
    if rule_id == "EFRuP@1.0.0":     # ← same bug this PR fixes in the ETL
        efrp_ref = r.get("subRuleRef", "none")
        continue
```

Consequences when an `EFRuP@2.0.0` alert arrives:
- `efrp_ref` stays `"none"`, so `display_efrp_status` renders the wrong EFRuP status header.
- The bumped EFRuP row leaks into `all_rules` and is displayed as if it were a normal rule.

Recommended fix — mirror the CMS PR #254 semantics with a prefix check (also fixes Issue 3 below):

```python
if rule_id.startswith("EFRuP@"):
    efrp_ref = r.get("subRuleRef", "none")
    continue
```

**2. Restore or replace the regression tests deleted by commit `4e9cd251`**

The first commit added `automation-orchestrator/tests/test_alerts_etl_efrup.py` (181 lines) whose docstring named this exact fix. The second commit removed it along with `pytest.ini`, `requirements-test.txt`, and `conftest.py`. The RLIKE change ships without any assertion that it accepts `EFRuP@2.0.0`, still accepts `EFRuP@1.0.0`, or rejects `EFRuP@1.0.0-beta`.

At minimum, a pure-Python regex-contract test covers this without needing the Spark harness:

```python
# automation-orchestrator/tests/test_efrup_regex.py
import re

EFRUP_PATTERN = r"^EFRuP@[0-9]+\.[0-9]+\.[0-9]+$"

def test_baseline_version_matches():
    assert re.match(EFRUP_PATTERN, "EFRuP@1.0.0")

def test_bumped_version_matches():
    assert re.match(EFRUP_PATTERN, "EFRuP@2.0.0")

def test_prerelease_rejected():
    assert not re.match(EFRUP_PATTERN, "EFRuP@1.0.0-beta")

def test_prefix_drift_rejected():
    assert not re.match(EFRUP_PATTERN, "EFRuP2@1.0.0")
```

Or resurrect the deleted Spark test from `git show 6bdec9a6:automation-orchestrator/tests/test_alerts_etl_efrup.py`.

---

### Non-blocking (please address in this PR if possible)

**3. Align semver semantics with CMS PR #254**

CMS uses prefix match (`startsWith('EFRuP@')` / `LIKE 'EFRuP@%'`). This PR uses strict semver. The two disagree on identifiers like `EFRuP@1.0.0-beta`, `EFRuP@1.0`, `EFRuP@2` — BIAR would null the silver row while CMS would still render a badge from the rule results. Either loosen to a prefix here, or leave a comment explaining the strict-semver contract if it's deliberate.

**4. Split the notebook cleanup into a separate PR next time**

The import reshuffle and `except:` sweep in `Rule_Discovery.ipynb` are unrelated to the ETL regex fix. Bundling them muddies future `git bisect`. Non-blocking for this PR — flagging for the next similar situation.
````

[↑ Back to top](#pr-review-biar-132--fix-implement-regex-validation-in-sql-query-for-efrup-version)
