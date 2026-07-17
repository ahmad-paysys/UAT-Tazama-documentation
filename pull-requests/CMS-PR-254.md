# PR Review: CMS #254 — fix: match EFRuP rule id by prefix instead of pinning to EFRuP@1.0.0

**Repo:** tazama-lf/case-management-system
**Branch:** `paysys/dashboard-fixes` → `dev`
**Author:** sobia-rizwan1567 (Sobia Rizwan)
**Date Reviewed:** 2026-07-17
**Label:** bug
**Size:** +110 / -5 lines across 4 files
**Commits:** 2 (56be11ef, f304ec9a)
**State:** OPEN
**HEAD SHA verified:** `f304ec9ac16a7d51358b223c22a2c2caab73fad9`
**Existing approvals:** none — CodeRabbit COMMENTED (one Trivial nitpick, addressed by the second commit); no human approval yet

## Table of Contents

- [Overview](#overview)
- [What Changed (Detailed)](#what-changed-detailed)
  - [1. `backend/src/modules/gold-lakehouse/alerts-lakehouse.service.ts` — prefix-match EFRuP in SQL and TS](#1-backendsrcmodulesgold-lakehousealerts-lakehouseservicets--prefix-match-efrup-in-sql-and-ts)
  - [2. `frontend/src/features/alerts/components/AlertsDetailModal.tsx` — prefix-match EFRuP in the modal extractor](#2-frontendsrcfeaturesalertscomponentsalertsdetailmodaltsx--prefix-match-efrup-in-the-modal-extractor)
  - [3. Test additions — bumped version + weighted-EFRuP coverage](#3-test-additions--bumped-version--weighted-efrup-coverage)
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

Prior to this PR, three sites hard-coded the exact string `'EFRuP@1.0.0'` — a SQL `WHERE` clause in the lakehouse aggregation, a `rulesData.find()` in the backend typology mapper, and a `ruleResults.find()` in the frontend `AlertsDetailModal`. Any bump to `EFRuP@2.0.0` would silently fail-closed at all three: the SQL row would drop out of the aggregation, the backend flow-processor would resolve to `undefined`, and the modal badge would simply not render. This PR replaces all three with prefix matches on `'EFRuP@'` (via `LIKE 'EFRuP@%'` in SQL and `.startsWith('EFRuP@')` in TS), and additionally hardens the backend by excluding EFRuP from `triggeredRulesData` by prefix rather than trusting `rule_weight === 0` — so that a future non-zero-weighted EFRuP row does not leak into the triggered-rules list.

Targets `dev` — correct for this repo's flow. All 15 CI checks are `SUCCESS` on HEAD `f304ec9a` (CodeQL, njsscan, nodejsscan, dependency-review, node-ci build/style/tests, hadolint, encoding, DCO, GPG-verify, conventional-commit title, CodeRabbit status).

| File | Nature of Change |
|------|-----------------|
| `backend/src/modules/gold-lakehouse/alerts-lakehouse.service.ts` | Introduce `EFRUP_RULE_ID_PREFIX = 'EFRuP@'` const in `getAlertNavigatorData`; use in SQL `LIKE`, `flowProcessorRule` find, and new `triggeredRulesData` exclusion. |
| `backend/test/alerts-lakehouse.service.spec.ts` | Add two tests: bumped-version EFRuP@2.0.0 still surfaces `flowProcessorData`; weighted EFRuP@1.0.0 is excluded from triggered rules. |
| `frontend/src/features/alerts/components/AlertsDetailModal.tsx` | Rename `EFRUP_RULE_ID` → `EFRUP_RULE_ID_PREFIX`; switch `.find(rule.id === EFRUP_RULE_ID)` to `.startsWith(EFRUP_RULE_ID_PREFIX)` with a type guard. |
| `frontend/src/features/alerts/components/__tests__/AlertsDetailModal.test.tsx` | Add EFRuP@2.0.0 display test. |

[↑ Back to top](#pr-review-cms-254--fix-match-efrup-rule-id-by-prefix-instead-of-pinning-to-efrup100)

---

## What Changed (Detailed)

### 1. `backend/src/modules/gold-lakehouse/alerts-lakehouse.service.ts` — prefix-match EFRuP in SQL and TS

```diff
   async getAlertNavigatorData(alertId: number, tenantId = 'DEFAULT', userJwt?: string): Promise<AlertNavigatorDataResponse> {
+    const EFRUP_RULE_ID_PREFIX = 'EFRuP@';
     try {
       // ... SQL construction ...
       WHERE anr.alert_id  = ${safeAlertId}
         AND anr.tenant_id = '${safeTenantId}'
         AND (
           anr.rule_weight > 0
-          OR anr.rule_id = 'EFRuP@1.0.0'
+          OR anr.rule_id LIKE '${EFRUP_RULE_ID_PREFIX}%'
         )
```

And in the mapping step:

```diff
-  const flowProcessorRule = rulesData.find((r) => r.rule_id === 'EFRuP@1.0.0');
-  const triggeredRulesData = rulesData.filter((r) => (r.rule_weight ?? 0) > 0);
+  const flowProcessorRule = rulesData.find((r) => r.rule_id?.startsWith(EFRUP_RULE_ID_PREFIX));
+  const triggeredRulesData = rulesData.filter((r) => (r.rule_weight ?? 0) > 0 && !r.rule_id?.startsWith(EFRUP_RULE_ID_PREFIX));
```

Three coupled changes:

- **SQL predicate.** `LIKE 'EFRuP@%'` widens the aggregation clause so any EFRuP version row is kept alongside truly-triggered rules (`rule_weight > 0`).
- **Flow-processor lookup.** `find()` now matches by prefix, so a version bump does not drop the row on the read path.
- **Triggered-rules exclusion.** Previously the exclusion relied on the implicit fact that EFRuP rows carry `rule_weight = 0`. The PR adds an explicit `!startsWith(...)` guard, so even if a future EFRuP row leaks in with a non-zero weight (e.g. a misconfigured deployment), it will still be filtered out of the triggered-rules JSON blob rendered to the UI.

The constant `EFRUP_RULE_ID_PREFIX` is scoped inside the function; the second commit (`fix: EFRUP constant`, f304ec9a) moved it up so the SQL interpolation and the two TS predicates all reference the same identifier. This directly addresses CodeRabbit's nitpick — see the CodeRabbit Activity section.

SQL-injection surface: the constant is a compile-time string literal, not user input, so interpolation into `LIKE '${EFRUP_RULE_ID_PREFIX}%'` is safe. `safeTenantId` and `safeAlertId` continue to use the pre-existing `escapeSqlString` / `clampPositiveInteger` guards.

### 2. `frontend/src/features/alerts/components/AlertsDetailModal.tsx` — prefix-match EFRuP in the modal extractor

```diff
-const EFRUP_RULE_ID = 'EFRuP@1.0.0';
+const EFRUP_RULE_ID_PREFIX = 'EFRuP@';
 const extractFlowProcessorData = (
   alert: AlertWithAlertedTypologies,
 ): string | undefined => {
   // ...
   const flowProcessorRule = ruleResults.find(
-    (rule) => rule.id === EFRUP_RULE_ID,
+    (rule) => typeof rule.id === 'string' && rule.id.startsWith(EFRUP_RULE_ID_PREFIX),
   );
```

Cleanly done. The added `typeof rule.id === 'string'` guard is important — `rule` comes from `ruleResults.filter(isRecord)` which does not narrow the shape of `rule.id`, so calling `.startsWith` on a non-string would throw. The type guard makes the switch from `===` to `.startsWith(...)` safe.

### 3. Test additions — bumped version + weighted-EFRuP coverage

Backend (`alerts-lakehouse.service.spec.ts`):

- **`surfaces flowProcessorData for a bumped EFRuP rule version (EFRuP@2.0.0)`** — mocks a `rules` array containing `rule-001` (weight 250) and `EFRuP@2.0.0` (weight 0), asserts `flowProcessorData === 'Block'` and that the emitted `rules` JSON contains only `rule-001`.
- **`excludes EFRuP from triggered rules even if it carries a non-zero weight`** — the load-bearing new coverage for the second-order fix. Mocks an EFRuP row with `rule_weight: 5` (previously would have been included in triggered rules) and asserts it is filtered out anyway.

Frontend (`AlertsDetailModal.test.tsx`):

- **`displays the EFRuP subRuleRef when the rule id is a bumped version (EFRuP@2.0.0)`** — asserts the modal renders the `Block` subRuleRef when the rule id is `EFRuP@2.0.0` rather than `EFRuP@1.0.0`.

The existing negative test (`does not display an EFRuP badge when no rule has id EFRuP@1.0.0`) is left unchanged.

[↑ Back to top](#pr-review-cms-254--fix-match-efrup-rule-id-by-prefix-instead-of-pinning-to-efrup100)

---

## Code Quality Analysis

### Strengths

- **Full-stack fix in one PR.** SQL, backend mapper, and frontend extractor all move together, so a bumped rule version cannot slip past a single layer that was overlooked.
- **Second-order hardening.** The `!startsWith(...)` addition to `triggeredRulesData` filter is a genuine defensive improvement beyond the stated "prefix instead of exact match" bug — it decouples the exclusion of EFRuP from the coincidence of its zero weight. The new test-case `excludes EFRuP from triggered rules even if it carries a non-zero weight` locks this in.
- **Frontend adds a `typeof` guard.** Switching from `===` to `.startsWith(...)` on a value typed as `unknown`/`Record<string, unknown>` needs a type guard; the PR adds one. Easy to miss, done correctly here.
- **Constant lifted after CodeRabbit nit.** The second commit responds to CodeRabbit's suggestion — the SQL `LIKE '${EFRUP_RULE_ID_PREFIX}%'` now uses the same identifier as the two predicates below, so a future rename to `EFRuP2@` or similar is a single edit per file.
- **Tests exercise the real invariants, not just the changed lines.** The backend test on non-zero weight is a regression guard against a future maintainer removing the `!startsWith(...)` clause; the frontend test on `EFRuP@2.0.0` guards against a future maintainer re-hardcoding the version.

### Issues and Observations

#### Issue 1 — Constant is duplicated across backend and frontend; two independent copies of the same magic string

**Severity: Informational (Maintainability)**

`EFRUP_RULE_ID_PREFIX = 'EFRuP@'` is declared twice — once in `alerts-lakehouse.service.ts:30` (function-scoped) and once in `AlertsDetailModal.tsx:166` (module-scoped). If the rule-engine ever renames the family (e.g. to `EFRuP2@` or `FlowProc@`), a maintainer must update both. There is no shared TS types package cross-cutting backend and frontend for constants of this kind, so this duplication is consistent with existing patterns in the repo — but it is worth flagging that a wire-protocol identifier now lives in two places. No fix requested here; noted for future consideration.

#### Issue 2 — Backend constant is function-scoped rather than class-scoped

**Severity: Informational (Code Quality)**

The constant sits inside `getAlertNavigatorData`. CodeRabbit's suggested extraction was to service-level (`private static readonly` on the class). Function-scoping works — this method is the only intra-file consumer — but a class-level `private static readonly EFRUP_RULE_ID_PREFIX = 'EFRuP@'` would make the wire-protocol identifier discoverable at the top of the class, alongside the `escapeSqlString` / `clampPositiveInteger` helpers. Minor; not blocking.

#### Issue 3 — Stale test title in the existing negative test

**Severity: Informational (Test Hygiene)**

The pre-existing test at `AlertsDetailModal.test.tsx:457` still reads `"does not display an EFRuP badge when no rule has id EFRuP@1.0.0"`. Under the new prefix semantics, the accurate wording would be `"no rule id starts with EFRuP@"`. The test body still exercises the right behaviour (a non-EFRuP rule doesn't render the badge), but the title now mildly misleads a reader who greps for "EFRuP@1.0.0" hunting the exact-version dependency. Not touched by this PR; correctable in the same file with a one-line rename if desired.

[↑ Back to top](#pr-review-cms-254--fix-match-efrup-rule-id-by-prefix-instead-of-pinning-to-efrup100)

---

## Security Assessment

| Concern | Assessment |
|---------|-----------|
| SQL injection | Unchanged. `EFRUP_RULE_ID_PREFIX` is a compile-time string literal, safe to interpolate. `safeTenantId` / `safeAlertId` continue to use `escapeSqlString` / `clampPositiveInteger`. |
| Tenant / role scoping | Unchanged. The `WHERE anr.tenant_id = '${safeTenantId}'` clause continues to gate the aggregation; the EFRuP prefix widening is inside the `OR` that adds rows, but tenant scoping is `AND`-combined at a higher level and still holds. |
| Auth-guard decorators | N/A — this PR touches a service, not a new endpoint. Controller and guards are untouched. |
| URL construction / SSRF | N/A. |
| HTML rendering / XSS | The `AlertsDetailModal` change reads `rule.id` (already required to be a `string` by the added type guard) and calls `.startsWith(...)`. No new value flows into the DOM; the pre-existing `flowProcessorData` render path (a plain `{ subRuleRef }` text node) is unchanged. React escaping applies as before. |
| Secrets / PII in logs | Unchanged. |
| CI security scanners | CodeQL (both variants), njsscan, nodejsscan, dependency-review all `SUCCESS`. |

No new security vulnerabilities introduced. If anything, the explicit `!startsWith(...)` on `triggeredRulesData` makes the read-path harder to accidentally leak an internal flow-processor row into the UI's triggered-rules JSON — a minor confidentiality-adjacent improvement.

[↑ Back to top](#pr-review-cms-254--fix-match-efrup-rule-id-by-prefix-instead-of-pinning-to-efrup100)

---

## Test Coverage

**What is tested (new):**

- Backend: bumped version (`EFRuP@2.0.0`) still resolves `flowProcessorData`. Locks the SQL-and-mapper prefix logic against regression to `===`.
- Backend: weighted EFRuP (`rule_weight: 5`) is still excluded from triggered rules. Locks in the second-order hardening.
- Frontend: bumped version renders the `EFRuP:` badge with the correct `subRuleRef`. Locks the modal prefix logic against regression to `===`.

**Existing coverage preserved:**

- Frontend negative test at `AlertsDetailModal.test.tsx:457` continues to pass — no EFRuP rule → no badge.
- Backend `null values gracefully` test at `alerts-lakehouse.service.spec.ts` is not touched.

**What is not tested:**

- **SQL prefix path is not directly asserted.** The backend tests mock the HTTP layer and inject `rules` rows already shaped as-if-aggregated, so the `LIKE '${EFRUP_RULE_ID_PREFIX}%'` predicate in the query string is exercised only in the sense that the emitted SQL contains the literal. There is no assertion of the constructed SQL query. This is consistent with the existing test style for this file (no test asserts the emitted SQL string) — so it is a matter of consistency rather than a gap.
- **The pre-existing negative test title is stale** (see Issue 3) — not a coverage gap, a labelling nit.

**PR checklist:** Locally, Development Environment, Husky, Unit tests, Documentation are all ticked. "Not needed, changes very basic" is left unchecked — appropriate for a bug fix.

[↑ Back to top](#pr-review-cms-254--fix-match-efrup-rule-id-by-prefix-instead-of-pinning-to-efrup100)

---

## CodeRabbit Activity

### Pass 1 — full review on commit `56be11ef`

**Commit reviewed:** `56be11efbd98097e9a9887486665b1b0451b26b5`
**Findings:** 1 actionable comment (Trivial/nitpick)

| Finding | Severity | Status |
|---------|----------|--------|
| Extract `'EFRuP@'` into a class-level constant so it is reused in both TS predicates and the SQL query. | Trivial (Maintainability) | ✅ Resolved (partial) — second commit `f304ec9a` extracted a function-scoped `EFRUP_RULE_ID_PREFIX` and threaded it through the SQL `LIKE` and both TS predicates. Class-level extraction (CodeRabbit's stronger form) was not taken; the function-scoped form is still a real improvement over duplicated literals and matches the pattern of other helper constants nearby. |

### Pass 2 — rate-limited on commit `f304ec9a`

CodeRabbit posted a rate-limit notice on the second commit ("Next review available in: 51 minutes"). No further automated findings.

**Reconciliation with independent hunts:**

- **Parallel-siblings hunt.** Ran `grep -rn "EFRuP" backend/src frontend/src --include="*.ts" --include="*.tsx"`. Three production sites match the EFRuP prefix (SQL, backend .find, backend .filter — all in `alerts-lakehouse.service.ts`) plus one frontend extractor (`AlertsDetailModal.tsx`). All four were updated. Two `<span>EFRuP:</span>` labels are static display strings, not comparisons, so no drift. No missed siblings — CodeRabbit and this independent hunt agree.
- **Label/boundary drift hunt.** The stale test title in `AlertsDetailModal.test.tsx:457` (Issue 3) is a mild drift CodeRabbit did not flag — surfacing it here as Informational, not blocking. This is not a class currently in Section 3.1 that would have caught it; drift *within test titles specifically* is a narrow enough case that adding it to the hunt list is not warranted.
- **Guard/scope hunt.** No new endpoints; existing tenant scoping in the SQL is preserved. CodeRabbit and this independent hunt agree.

[↑ Back to top](#pr-review-cms-254--fix-match-efrup-rule-id-by-prefix-instead-of-pinning-to-efrup100)

---

## Summary and Verdict

**Verdict: Approved**

A small, well-scoped resiliency fix: the exact-version pinning on `EFRuP@1.0.0` is replaced with prefix matching at all three production sites (SQL `WHERE`, backend `.find`, frontend `.find`), and the backend gets a second-order improvement that no longer relies on `rule_weight === 0` to exclude EFRuP from triggered rules. The added tests exercise the real invariants (bumped version + weighted EFRuP), the frontend adds a necessary type guard, and CodeRabbit's constant-extraction nitpick was addressed by the second commit. No security regressions; all CI green.

### Blocking

None.

### Non-blocking but recommended

1. **(Optional) Lift the backend `EFRUP_RULE_ID_PREFIX` to a `private static readonly` on the class.** Function-scoping works, but a class-level constant sits with the other file-local helpers (`escapeSqlString`, `clampPositiveInteger`) and is more discoverable to future maintainers.
2. **(Optional) Rename the stale test title at `AlertsDetailModal.test.tsx:457`.** `"does not display an EFRuP badge when no rule has id EFRuP@1.0.0"` → `"...when no rule id starts with EFRuP@"`. The test body is correct; only the title lags the semantics.

[↑ Back to top](#pr-review-cms-254--fix-match-efrup-rule-id-by-prefix-instead-of-pinning-to-efrup100)

---

## GitHub Review Comment

````markdown
**Approved**

Clean full-stack resiliency fix: exact-version `'EFRuP@1.0.0'` pinning is replaced with prefix matching at every production site (SQL `WHERE`, backend mapper, frontend modal extractor), and the backend `triggeredRulesData` filter now explicitly excludes EFRuP by prefix instead of relying on `rule_weight === 0` — a nice second-order hardening. The frontend correctly adds a `typeof rule.id === 'string'` guard when switching from `===` to `.startsWith`, and the new tests lock both the bumped-version path (`EFRuP@2.0.0`) and the weighted-EFRuP-exclusion path. CodeRabbit's constant-extraction nit was addressed by the second commit (`fix: EFRUP constant`). All 15 CI checks green.

---

### Non-blocking (please address in this PR if possible)

**1. Optional: lift the backend `EFRUP_RULE_ID_PREFIX` to a class-level `private static readonly`**

Currently it sits inside `getAlertNavigatorData`. That's fine, but as a `private static readonly` on `AlertsLakehouseService` it would sit alongside `escapeSqlString` / `clampPositiveInteger` and be more discoverable to a future maintainer. Same string, one line moved:

```ts
export class AlertsLakehouseService extends GoldLakehouseService {
  private static readonly EFRUP_RULE_ID_PREFIX = 'EFRuP@';
  // ...
}
```

**2. Optional: rename the stale test title at `frontend/src/features/alerts/components/__tests__/AlertsDetailModal.test.tsx:457`**

The pre-existing negative test still reads `"does not display an EFRuP badge when no rule has id EFRuP@1.0.0"`. Under the new prefix semantics, `"...when no rule id starts with EFRuP@"` is more accurate. The test body is correct; only the title drifted.
````

[↑ Back to top](#pr-review-cms-254--fix-match-efrup-rule-id-by-prefix-instead-of-pinning-to-efrup100)
