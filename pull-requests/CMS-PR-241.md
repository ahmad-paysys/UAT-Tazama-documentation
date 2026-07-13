# PR Review: CMS #241 — Paysys/Add Rule properties inside Typology in Visualization

**Repo:** tazama-lf/case-management-system
**Branch:** `paysys/addRulesPropsInTypology` → `dev`
**Author:** MAdeel95 (Muhammad Adeel)
**Date Reviewed:** 2026-07-13
**Label:** _none_
**Size:** +37 / -281 lines across 6 files
**Commits:** 8 (latest `05ab1c3`)
**State:** OPEN (mergeStateStatus: BLOCKED, mergeable: MERGEABLE)
**Existing approvals:** none

## Table of Contents

- [Overview](#overview)
- [What Changed (Detailed)](#what-changed-detailed)
  - [1. `backend/src/modules/gold-lakehouse/alerts-lakehouse.service.ts` — surface new rule metadata](#1-alerts-lakehouse-service)
  - [2. `backend/src/modules/gold-lakehouse/types/raw-rule-row.types.ts` — extend raw row type](#2-raw-rule-row-types)
  - [3. `backend/src/modules/alert/alert.service.ts` — drop payload logging](#3-alert-service)
  - [4. `frontend/.../alertnavigator/types/index.ts` — extend `RuleDetailDto`](#4-frontend-types)
  - [5. `frontend/.../alertnavigator/AlertNavigatorTab.tsx` — render new fields, remove loading branch](#5-alertnavigatortab)
  - [6. `backend/package-lock.json` — resolved/integrity fields stripped](#6-lockfile)
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

The PR enriches the Alert Navigator visualization so each triggered rule inside a typology can display its description and matched-band reason. This is a straight-through change: the lakehouse SQL aggregation gathers three new columns (`rule_desc`, `matched_band_reason`, `matched_rule_reason`), the backend mapping forwards them, the frontend `RuleDetailDto` accepts them, and `AlertNavigatorTab.tsx` renders the two most useful (`ruleDesc`, `matched_band_reason`) inside the per-rule details block. It also removes a verbose `loggerService.log` line in `AlertService.getTransactionHistory` that stringified full transaction payloads, and — as a side change — deletes the loading-state early-return branch from `AlertNavigatorTab`.

Targets `dev` — correct for this repo's flow.

CI status: three failing checks —
- `conventional-commits / validate-pr-title` (PR title `Paysys/Add Rule properties inside Typology in Visualization` doesn't match the required Conventional Commits format, e.g. `feat: …`)
- `node-ci / check style` (lint failure — several diff hunks introduce trailing whitespace and stray blank lines)
- `node-ci / check tests`
These must be green before merge.

| File | Nature of Change |
|------|------------------|
| `backend/src/modules/gold-lakehouse/alerts-lakehouse.service.ts` | SQL `json_build_object` gains 3 fields; `mappedRules` forwards `ruleDesc`/`matched_band_reason`/`matched_rule_reason`. |
| `backend/src/modules/gold-lakehouse/types/raw-rule-row.types.ts` | Adds `rule_desc`, `matched_band_reason`, `matched_rule_reason`, and `band_reasons_with_sub_rule_refs_json` (unused by the mapper). |
| `backend/src/modules/alert/alert.service.ts` | Removes a debug `loggerService.log` line that serialized full transaction data. |
| `frontend/.../alertnavigator/types/index.ts` | Extends `RuleDetailDto` with 8 optional fields; only 3 are actually used. |
| `frontend/.../alertnavigator/AlertNavigatorTab.tsx` | Renders `ruleDesc` and `matched_band_reason` under each rule; removes the `if (loading) …` render branch. |
| `backend/package-lock.json` | 133 `resolved` URLs and matching `integrity` SHAs deleted across dev sub-trees. |

[↑ Back to top](#pr-review-cms-241--paysysadd-rule-properties-inside-typology-in-visualization)

---

## What Changed (Detailed)

### 1. `backend/src/modules/gold-lakehouse/alerts-lakehouse.service.ts` — surface new rule metadata <a id="1-alerts-lakehouse-service"></a>

Two hunks. First, the SQL aggregation adds three new JSON fields per rule row:

```diff
                         'rule_independent_variable', anr.rule_independent_variable,
                         'rule_sub_ref',              anr.rule_sub_ref,
                         'rule_processing_time_ms',   anr.rule_processing_time_ms,
-                        'rule_tenant_id',            anr.rule_tenant_id
+                        'rule_tenant_id',            anr.rule_tenant_id,
+                        'rule_desc',                 anr.rule_desc,
+                        'matched_band_reason', anr.matched_band_reason,
+                        'matched_rule_reason', anr.matched_rule_reason
                     )
                 ) AS rules
             FROM alert_navigator_rules anr
```

Second, the mapping over `triggeredRulesData` now forwards these three fields to the frontend payload:

```diff
-          const rulesString = JSON.stringify(
-            triggeredRulesData.map((r) => ({
+          const mappedRules = triggeredRulesData.map((r) => ({
               ruleId: r.rule_id,
               ruleWeight: r.rule_weight,
               subRef: r.rule_sub_ref,
               independentVariable: r.rule_independent_variable,
-            })),
-          );
+              ruleDesc: r.rule_desc,
+              matched_band_reason: r.matched_band_reason,
+              matched_rule_reason: r.matched_rule_reason,
+            }));
+          const rulesString = JSON.stringify(mappedRules);
```

Notes:
- Inline formatting: two of the three new SQL fields lost their column-alignment (spaces collapsed), and the mapped-object block has trailing whitespace and a stray blank line — this is what is failing `check style`.
- The intermediate `mappedRules` variable is introduced without a reason (the previous inline form was equally readable). Not a blocker, but pure churn.
- Naming mismatch: `ruleId`/`ruleWeight`/`subRef`/`independentVariable`/`ruleDesc` are camelCase; `matched_band_reason`/`matched_rule_reason` are snake_case. This mismatch is echoed through the frontend DTO. See Issue 3.

---

### 2. `backend/src/modules/gold-lakehouse/types/raw-rule-row.types.ts` — extend raw row type <a id="2-raw-rule-row-types"></a>

```diff
   rule_sub_ref: string | null;
+  band_reasons_with_sub_rule_refs_json: string[] | null;
   rule_processing_time_ms: number | null;
   rule_tenant_id: string | null;
+  rule_desc: string | null;
+  matched_rule_reason: string | null;
+  matched_band_reason: string | null;
```

- The three fields consumed by `mappedRules` are correctly typed `string | null`.
- `band_reasons_with_sub_rule_refs_json: string[] | null` is added to the type but is not selected in the SQL `json_build_object` above, so at runtime it will always be `undefined` — never populated. Adding it to the type is misleading. Either drop it here or select it in the query. See Issue 4.
- The last line has trailing whitespace after `null;`.

---

### 3. `backend/src/modules/alert/alert.service.ts` — drop payload logging <a id="3-alert-service"></a>

```diff
     if (!transactionData.data) {
       throw new InternalServerErrorException(`Transaction history data not found for AlertId ${alertId}`);
     }
-    this.loggerService.log(`Fetched transaction data for Alert ID ${alertId}: ${JSON.stringify(transactionData)}`, AlertService.name);
     return { transactionData };
   }
```

Legitimate cleanup — dumping full transaction payloads into logs is a real PII/secret-leak risk (PANs, MSISDNs, amounts, counterparties). Removing it is a small security win. Unrelated to the Typology-visualization headline change, but low-risk enough to bundle.

---

### 4. `frontend/.../alertnavigator/types/index.ts` — extend `RuleDetailDto` <a id="4-frontend-types"></a>

```diff
   independentVariable?: string;
   data?: unknown;
+  ruleDesc?: string;
+  band_reasons_with_sub_rule_refs_json?: string[];
+  matched_band_reason?: string | null;
+  exit_condition_reasons_json?: string[];
+  matched_exit_condition_reason?: string | null;
+  matched_rule_reason?: string | null;
+  band_count?: number | null;
+  exit_condition_count?: number | null;
 }
```

Only three of these eight properties are actually emitted by the backend (`ruleDesc`, `matched_band_reason`, `matched_rule_reason`) and only two are consumed in the render (`ruleDesc`, `matched_band_reason`). The other five are pure type-surface bloat — they don't exist on the wire today and will read as `undefined`. See Issue 5.

Also, `ruleDesc?: string` doesn't include `null`, but the backend maps `ruleDesc: r.rule_desc` where `r.rule_desc: string | null`. CodeRabbit flagged this correctly — see Issue 2.

---

### 5. `frontend/.../alertnavigator/AlertNavigatorTab.tsx` — render new fields, remove loading branch <a id="5-alertnavigatortab"></a>

Three material changes.

**(a) Loading branch removed:**

```diff
-  if (loading) {
-    return (
-      <div className="flex items-center justify-center py-12">
-        <div className="h-8 w-8 animate-spin rounded-full border-4 border-gray-300 border-t-blue-600"></div>
-        <span className="ml-3 text-gray-600">Loading...</span>
-      </div>
-    );
-  }
```

But the `loading` state is still declared at line 28, initialised to `true`, set to `true` before each fetch (line 45), and back to `false` in `finally` (line 63). Because it is no longer read, the render path on first mount / refetch is:

```
data === null, error === null, alertId truthy → falls through to `if (!data)` → "No data available"
```

That's a UX regression: users see "No data available" during the fetch instead of a spinner. This is exactly what CodeRabbit flagged as Major. See Issue 1.

**(b) Rule detail rendering — two new lines:**

```diff
                               {rule.independentVariable}
                                 </div>
                               )}
+                              {rule.ruleDesc != null && (
+                                <div className="text-xs text-gray-500 mt-0.5">
+                                  Rule Description: {' '}
+                                  {rule.ruleDesc}
+                                </div>
+                              )}
+                              {rule.matched_band_reason != null && (
+                                <div className="text-xs text-gray-500 mt-0.5">
+                                  Band Reason: {' '}
+                                  {rule.matched_band_reason}
+                                </div>
+                              )}
```

Straightforward and correctly guarded with `!= null` (handles both `null` and `undefined`). This is the actual headline feature.

**(c) Stray whitespace-only additions:**

```diff
         setData(result);
+
 
         // Keep all typologies collapsed initially
```

and

```diff
                         typology.rules.map((rule, idx: number) => (
                           <div key={idx} className="flex items-start gap-2">
+
                             <div className="flex-shrink-0 mt-1">
```

Both are pure whitespace-noise diff hunks — they contribute nothing and are among the causes of `check style` failing. See Issue 6.

---

### 6. `backend/package-lock.json` — resolved/integrity fields stripped <a id="6-lockfile"></a>

The diff removes 266 lines from `backend/package-lock.json`. Spot-checking the removals shows the pattern is identical everywhere:

```diff
     "node_modules/@angular-devkit/schematics-cli/node_modules/@angular-devkit/core": {
       "version": "19.2.24",
-      "resolved": "https://registry.npmjs.org/@angular-devkit/core/-/core-19.2.24.tgz",
-      "integrity": "sha512-Kd49warf6U/EyWe5BszF/eebN3zQ3bk7tgfEljAw8q/rX95UUtriJubWvp6pgzHfzBA4jwq8f+QiNZB8eBEXPA==",
       "dev": true,
```

Count of `"resolved":` entries: `origin/dev` = 214, this PR = 81 — 133 packages have had their tarball URL and SHA-512 integrity hash removed. Every one of these is a dev sub-tree.

This is not the shape of a normal `npm install`. `npm i` populates `resolved`/`integrity` on any package it touches; it doesn't strip them. Possible causes: an `npm i --no-save --package-lock-only` variant, an editor auto-format that dropped fields, or a merge/rebase that lost data.

Consequences of merging as-is:
- `npm ci` on the affected trees no longer has a tarball URL to fetch from nor a hash to verify against — installs must fall back to resolving through the registry, and integrity verification is skipped for those entries.
- The GitHub `Dependency Review` action currently passes, but it operates on top-level `package.json` diffs; it does not re-verify integrity on transitive dev entries.
- This is functionally a downgrade of supply-chain guarantees for CI/CD. See Issue 7.

[↑ Back to top](#pr-review-cms-241--paysysadd-rule-properties-inside-typology-in-visualization)

---

## Code Quality Analysis

### Strengths

- Backend/frontend contract updated together in one PR — no risk of the frontend rendering fields the backend doesn't send.
- Removal of `loggerService.log(... JSON.stringify(transactionData) ...)` is a genuine privacy improvement.
- Render-side guards use `!= null`, which correctly covers both `null` and `undefined`.
- Change is genuinely small in surface area (2 files of real logic, 2 of type surface).

### Issues and Observations

#### Issue 1 — Loading UX regression: "No data available" flashes during fetch

**Severity: Major (Bug / UX regression)**

The `if (loading) return <spinner/>` branch was removed from `AlertNavigatorTab.tsx` (lines 86-94 of the old file), but the `loading` state and its `setLoading(true/false)` calls remain. Now the render path is:

```
initial mount → loading=true, data=null → falls through to `if (!data)` (line 118) →
renders "No data available"
```

The user sees "No data available" for the entire duration of the initial fetch and every refetch. Corroborated by CodeRabbit ("Outside diff range comments (1)").

Fix — either restore the spinner branch (preferred, one line hoist above `if (!data)`):

```tsx
if (loading) {
  return (
    <div className="flex items-center justify-center py-12">
      <div className="h-8 w-8 animate-spin rounded-full border-4 border-gray-300 border-t-blue-600"></div>
      <span className="ml-3 text-gray-600">Loading...</span>
    </div>
  );
}
```

Or, if the loading UI is intentionally being removed (please state why), also remove `const [loading, setLoading] = useState(true)` and the three `setLoading(...)` calls so dead state doesn't linger.

#### Issue 2 — `RuleDetailDto.ruleDesc` type doesn't match backend contract

**Severity: Minor (Data Integrity)**

Backend `RawRuleRow.rule_desc: string | null` is passed straight through as `ruleDesc: r.rule_desc`, but the frontend type declares `ruleDesc?: string` (line 7 of `types/index.ts`). When the backend emits `null` — which it demonstrably will, given the `| null` on the raw row — the frontend type is violated. The sibling fields `matched_band_reason` / `matched_rule_reason` correctly use `string | null`.

Corroborated by CodeRabbit's inline comment on lines 7-14.

Fix:

```diff
-  ruleDesc?: string;
+  ruleDesc?: string | null;
```

#### Issue 3 — Inconsistent camelCase / snake_case in the wire payload

**Severity: Minor (Code Quality / Maintainability)**

The mapped rule object mixes conventions: `ruleId`, `ruleWeight`, `subRef`, `independentVariable`, `ruleDesc` are camelCase; `matched_band_reason`, `matched_rule_reason` are snake_case. Since this is the shape of the JSON that crosses the wire, it will lock in the inconsistency for every consumer. If snake_case is not the deliberate convention on this project (the surrounding TS is camelCase-only), rename before merge — it's cheap now and expensive later.

```diff
-              matched_band_reason: r.matched_band_reason,
-              matched_rule_reason: r.matched_rule_reason,
+              matchedBandReason: r.matched_band_reason,
+              matchedRuleReason: r.matched_rule_reason,
```

…and mirror in `RuleDetailDto` and the render.

#### Issue 4 — `band_reasons_with_sub_rule_refs_json` added to type but never selected

**Severity: Minor (Code Quality)**

`RawRuleRow.band_reasons_with_sub_rule_refs_json: string[] | null` is declared in `raw-rule-row.types.ts` but the SQL `json_build_object` doesn't include it, so `r.band_reasons_with_sub_rule_refs_json` is always `undefined` at runtime. Either drop from the type or add to the query.

#### Issue 5 — Five type-surface additions with no runtime data

**Severity: Minor (Code Quality)**

`RuleDetailDto` gains 8 optional fields, but the backend only emits 3 (`ruleDesc`, `matched_band_reason`, `matched_rule_reason`), and the render only consumes 2 (`ruleDesc`, `matched_band_reason`). The other 5 — `band_reasons_with_sub_rule_refs_json`, `exit_condition_reasons_json`, `matched_exit_condition_reason`, `band_count`, `exit_condition_count` — will always be `undefined`. They read like copy-paste from an aspirational schema.

Either wire them through (SQL + mapper + render) or drop them. Speculative type surface rots.

#### Issue 6 — Trailing whitespace and stray blank lines cause `check style` failure

**Severity: Minor (Code Quality)**

Lint is failing because the PR introduces:
- trailing whitespace after `null;` in `raw-rule-row.types.ts` line for `matched_band_reason`
- trailing whitespace on `ruleDesc: r.rule_desc,`, `matched_band_reason: …,`, `matched_rule_reason: …,` in `alerts-lakehouse.service.ts`
- a stray blank line inside the `mappedRules` object literal
- a stray blank line after `setData(result);` in `AlertNavigatorTab.tsx`
- a stray blank line before the `<div className="flex-shrink-0 mt-1">` block

Running the project's lint auto-fix (`npm run lint -- --fix`) will clear these. `check style` will pass on the next push.

#### Issue 7 — 133 `resolved`/`integrity` entries stripped from `backend/package-lock.json`

**Severity: Major (Supply-chain / Data Integrity)**

`origin/dev` has 214 `"resolved":` entries; this PR has 81. The 133 removed entries lose both the tarball URL and the SHA-512 integrity hash. `npm ci` will no longer verify these packages against a pinned hash — it must re-resolve through the registry, and integrity is skipped.

This is unlikely to be intentional and is not related to the Typology visualization change. Regenerate the lockfile cleanly (`rm -f backend/package-lock.json && npm i --package-lock-only` from a clean state at the repo's declared Node/npm version), commit the result, and confirm the `resolved` count matches `origin/dev` (± any deliberate dep changes — of which this PR has none).

If the field-stripping is a project convention I'm not aware of, please confirm and I'll withdraw this item.

#### Issue 8 — PR title doesn't match Conventional Commits

**Severity: Minor (Code Quality — CI blocker)**

`conventional-commits / validate-pr-title` is failing. Repo memory: under `solutions/` the project uses `feat: `/`fix: ` prefixes. Rename to e.g. `feat: add rule description and matched-band reason to Alert Navigator visualization`. This unblocks the check.

#### Issue 9 — `check tests` is failing

**Severity: Major (Test Coverage — CI blocker)**

`node-ci / check tests` failed on the current head. I did not run the suite locally; whichever test is red must be fixed (or the removed loading branch may have broken a component test that asserted the spinner rendered). Investigate and resolve before merge.

[↑ Back to top](#pr-review-cms-241--paysysadd-rule-properties-inside-typology-in-visualization)

---

## Security Assessment

| Concern | Assessment |
|---------|-----------|
| Sensitive data in logs | **Improved.** `AlertService.getTransactionHistory` no longer serializes full transaction payloads into the log stream. |
| SQL injection / query construction | No change. Fields added to `json_build_object` are static column names against `alert_navigator_rules anr`; no user input flows into the SQL. |
| XSS via new fields | `ruleDesc` and `matched_band_reason` are rendered as text children inside JSX — React auto-escapes. No `dangerouslySetInnerHTML`, no template concatenation. Safe. |
| AuthZ / tenant scoping | Unchanged. `alertsLakehouse` continues to scope by `tenantId`; no new endpoints or claim changes. |
| Supply-chain integrity | **Worsened.** 133 lockfile entries lost their `integrity` hash — see Issue 7. |
| Input validation on the new fields | Backend passes `rule_desc`/`matched_*` through unmodified. If these columns can contain untrusted user-controlled content elsewhere in the pipeline, React escaping still protects the DOM, but this is worth confirming with the data-producer. |

Net: one clear win (log-scrubbing), one clear regression (lockfile integrity), no new injection surfaces.

[↑ Back to top](#pr-review-cms-241--paysysadd-rule-properties-inside-typology-in-visualization)

---

## Test Coverage

- **What is tested by this PR:** nothing new. No unit tests, no snapshot updates, no lakehouse-service test additions.
- **What is not tested:** the SQL aggregation change (new fields in `json_build_object`), the `mappedRules` transformation, the two new render branches, and the loading-branch removal. There are no assertions that guard against the "No data available flashes during fetch" regression documented in Issue 1.
- **CI status:** `node-ci / check tests` is **FAILING** on the current head — see Issue 9.
- **PR checklist:** the PR description is empty. No boxes to check. No coverage screenshot.

Given that this PR both changes a rendered branch and removes an existing render branch, at minimum a component-level test that renders `<AlertNavigatorTab loading data={null}>` and asserts a spinner (or a graceful placeholder) — instead of "No data available" — is warranted.

[↑ Back to top](#pr-review-cms-241--paysysadd-rule-properties-inside-typology-in-visualization)

---

## CodeRabbit Activity

### Pass 1 — Initial review on `05ab1c3`

**Commit reviewed:** `05ab1c3282a5f8a4158ada40121a69d70dc0faab`
**Findings:** 1 actionable (Major), 1 inline (Minor), 2 nitpicks

| Finding | Severity | Status |
|---------|----------|--------|
| Removed loading branch causes "No data available" flash — restore or remove `loading` state | Major | ❌ Not resolved — **corroborated as Issue 1** |
| `ruleDesc?: string` should be `ruleDesc?: string \| null` to match backend contract | Minor | ❌ Not resolved — **corroborated as Issue 2** |
| Naming inconsistency between `ruleId`/`ruleWeight`/`ruleDesc` (camelCase) and `matched_band_reason`/`matched_rule_reason` (snake_case) | Trivial (nit) | ❌ Not resolved — **corroborated as Issue 3** (raised to Minor because it's a wire-format decision) |
| `loading` state is dead code if the loading branch stays removed | Trivial (nit) | ❌ Not resolved — **covered under Issue 1** |

CodeRabbit missed: the lockfile integrity strip (Issue 7), the type-surface bloat (Issue 5), the phantom `band_reasons_with_sub_rule_refs_json` type field (Issue 4), the whitespace/lint issues driving `check style` failure (Issue 6), the failing `check tests` (Issue 9), and the Conventional Commits title failure (Issue 8).

[↑ Back to top](#pr-review-cms-241--paysysadd-rule-properties-inside-typology-in-visualization)

---

## Summary and Verdict

**Verdict: Changes Requested**

The headline feature — surfacing `ruleDesc` and `matched_band_reason` in each expanded rule — is small and correct, and the accompanying log-scrubbing in `AlertService.getTransactionHistory` is a nice side-improvement. But the PR is not ready to merge: it removes a loading UI branch while leaving the `loading` state wired up, which makes the component flash "No data available" during every fetch; it strips 133 `integrity`/`resolved` entries from `backend/package-lock.json`, weakening supply-chain guarantees for CI; the PR title fails Conventional Commits validation; and both `check style` and `check tests` are red on the current head.

The type-level and naming issues (Issues 2-5) are individually minor but compound into a payload contract that is inconsistent (camelCase/snake_case mixed) and inflated with five unused optional fields — worth fixing while everything is being touched.

### Blocking

1. **"No data available" flashes during fetch (Issue 1)** — restore the `if (loading)` render branch, or remove the `loading` state entirely and choose an intentional loading UX.
2. **Lockfile `resolved`/`integrity` fields stripped (Issue 7)** — regenerate `backend/package-lock.json` cleanly and restore the 133 missing entries so `npm ci` can verify integrity.
3. **`node-ci / check tests` failing (Issue 9)** — investigate and fix; do not merge red.
4. **`node-ci / check style` failing (Issue 6)** — run lint --fix on the touched files.
5. **PR title fails Conventional Commits (Issue 8)** — rename to `feat: …` (or equivalent).

### Non-blocking but recommended

6. **`RuleDetailDto.ruleDesc` type widening (Issue 2)** — accept `string | null`.
7. **Wire-format naming consistency (Issue 3)** — camelCase the two `matched_*` keys or accept the mixed form deliberately and document it.
8. **Trim unused type fields (Issue 5) and phantom `band_reasons_with_sub_rule_refs_json` (Issue 4)** — either wire them through end-to-end or drop from the types.
9. **Add a component test for the loading / no-data render paths** — the branch just changed and has no coverage.

[↑ Back to top](#pr-review-cms-241--paysysadd-rule-properties-inside-typology-in-visualization)

---

## GitHub Review Comment

````markdown
**Changes Requested**

The headline change (surfacing `ruleDesc` and `matched_band_reason` under each rule) is small and correct, and dropping the transaction-payload `logger.log` is a nice side win. But the PR is not ready to merge: the loading UI was removed while the `loading` state stayed wired up, the lockfile has lost 133 integrity hashes, three CI checks are red, and there are several type/contract inconsistencies worth cleaning up while everything is being touched.

---

### Blocking

**1. "No data available" flashes during fetch — `frontend/src/features/cases/components/view/visualizations/alertnavigator/AlertNavigatorTab.tsx`**

The `if (loading) return <spinner/>` branch was removed, but `loading` is still initialised to `true`, set to `true` before each fetch (line 45), and cleared in `finally` (line 63). On initial mount and every refetch, `data` is `null` while `loading` is `true`, so the component falls through to `if (!data)` at line 118 and renders **"No data available"** instead of a loading indicator. This is a UX regression on the current happy path.

Please either restore the spinner branch above the `!alertId` check:

```tsx
if (loading) {
  return (
    <div className="flex items-center justify-center py-12">
      <div className="h-8 w-8 animate-spin rounded-full border-4 border-gray-300 border-t-blue-600"></div>
      <span className="ml-3 text-gray-600">Loading...</span>
    </div>
  );
}
```

…or, if the removal is deliberate, also remove `const [loading, setLoading] = useState(true)` and its three setters so the dead state doesn't linger.

**2. `backend/package-lock.json` lost 133 `resolved`/`integrity` entries**

Comparing against `origin/dev`, this PR drops the tarball URL and SHA-512 integrity hash from 133 packages (dev sub-trees). `origin/dev` has 214 `"resolved":` entries; this PR has 81. Example:

```diff
     "node_modules/@angular-devkit/schematics-cli/node_modules/@angular-devkit/core": {
       "version": "19.2.24",
-      "resolved": "https://registry.npmjs.org/@angular-devkit/core/-/core-19.2.24.tgz",
-      "integrity": "sha512-Kd49warf6U/EyWe5BszF/eebN3zQ3bk7tgfEljAw8q/rX95UUtriJubWvp6pgzHfzBA4jwq8f+QiNZB8eBEXPA==",
       "dev": true,
```

This weakens supply-chain guarantees — `npm ci` can no longer verify these packages by hash. Please regenerate the lockfile cleanly (`rm -f backend/package-lock.json && npm i --package-lock-only`) at the repo's declared Node/npm version and commit that.

**3. `node-ci / check tests` is failing** — please investigate the red job; the removed loading branch may have broken a component test that asserted on the spinner.

**4. `node-ci / check style` is failing** — the diff introduces trailing whitespace and stray blank lines (e.g. after `setData(result);` in `AlertNavigatorTab.tsx`, on the new mapped-rule lines in `alerts-lakehouse.service.ts`, and at the end of `matched_band_reason: string | null;` in `raw-rule-row.types.ts`). `npm run lint -- --fix` should clear them.

**5. PR title fails Conventional Commits** — rename to something like `feat: add rule description and matched-band reason to Alert Navigator visualization` to unblock `conventional-commits / validate-pr-title`.

---

### Non-blocking (please address in this PR if possible)

**6. `RuleDetailDto.ruleDesc` type mismatch — `frontend/.../alertnavigator/types/index.ts`**

The backend maps `ruleDesc: r.rule_desc` where `r.rule_desc: string | null`, but the frontend declares `ruleDesc?: string`. When the backend sends `null` the type is violated. The sibling `matched_*` fields already use `string | null`.

```diff
-  ruleDesc?: string;
+  ruleDesc?: string | null;
```

**7. Wire-format naming inconsistency — `backend/src/modules/gold-lakehouse/alerts-lakehouse.service.ts`**

`mappedRules` mixes camelCase (`ruleId`, `ruleWeight`, `subRef`, `independentVariable`, `ruleDesc`) with snake_case (`matched_band_reason`, `matched_rule_reason`). Since this shape crosses the wire, the inconsistency will lock in for every consumer. Consider:

```diff
-              matched_band_reason: r.matched_band_reason,
-              matched_rule_reason: r.matched_rule_reason,
+              matchedBandReason: r.matched_band_reason,
+              matchedRuleReason: r.matched_rule_reason,
```

…and mirror in `RuleDetailDto` and the render.

**8. Trim unused type surface**

`RuleDetailDto` grew by 8 optional fields, but only 3 are emitted by the backend (`ruleDesc`, `matched_band_reason`, `matched_rule_reason`) and only 2 are rendered. The other 5 (`band_reasons_with_sub_rule_refs_json`, `exit_condition_reasons_json`, `matched_exit_condition_reason`, `band_count`, `exit_condition_count`) will always be `undefined`. Similarly, `RawRuleRow.band_reasons_with_sub_rule_refs_json` is declared but never selected in the SQL `json_build_object`. Either wire these fields through end-to-end or drop from the types.

**9. No test coverage for the changed render branch**

A small component test that renders `<AlertNavigatorTab loading data={null}>` and asserts on the loading indicator (or the intentional replacement) would guard against regressions like #1 in future.
````

[↑ Back to top](#pr-review-cms-241--paysysadd-rule-properties-inside-typology-in-visualization)
