# PR Review: CMS #249 — fix: surface Auto-closed cases in Cases List filters and improve Gold Lakehouse error visibility

**Repo:** tazama-lf/case-management-system
**Branch:** `paysys/dashboard-fixes` → `dev`
**Author:** sobia-rizwan1567 (Sobia Rizwan)
**Date Reviewed:** 2026-07-16
**Label:** bug
**Size:** +26 / -5 lines across 5 files
**Commits:** 2 (`59f5df3d`, `c2ca0301`)
**State:** OPEN
**HEAD SHA verified:** `c2ca0301247ef0274e5ddcfc0dd88b9cfcefbbe4`
**Existing approvals:** none — CodeRabbit posted the walkthrough only, no human approval yet

---

## Table of Contents

- [Overview](#overview)
- [What Changed (Detailed)](#what-changed-detailed)
  - [1. `backend/src/modules/gold-lakehouse/alerts-lakehouse.service.ts` — drop dead `anh.end_to_end_id` from Alert Navigator SQL](#1-backendsrcmodulesgold-lakehousealerts-lakehouseservicets--drop-dead-anhend_to_end_id-from-alert-navigator-sql)
  - [2. `backend/src/modules/gold-lakehouse/gold-lakehouse.service.ts` — log upstream response body on Axios failure](#2-backendsrcmodulesgold-lakehousegold-lakehouseservicets--log-upstream-response-body-on-axios-failure)
  - [3. `backend/test/gold-lakehouse.service.spec.ts` — add spec for new Axios logging branch; strip BOM](#3-backendtestgold-lakehouseservicespects--add-spec-for-new-axios-logging-branch-strip-bom)
  - [4. `frontend/src/features/cases/components/CaseFilters.tsx` — surface Auto-closed statuses in Cases List](#4-frontendsrcfeaturescasescomponentscasefilterstsx--surface-auto-closed-statuses-in-cases-list)
  - [5. `frontend/src/features/reports/components/CaseAgeingBarChart.tsx` — round X-axis tick values](#5-frontendsrcfeaturesreportscomponentscaseageingbarcharttsx--round-x-axis-tick-values)
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

This PR bundles four small, related dashboard bug fixes:

1. **Alert Navigator SQL fix.** Removes `anh.end_to_end_id` from the `SELECT`/`GROUP BY` in the Alert Navigator query on `alert_navigator_header`. The column was silently dropped upstream in BIAR on 2026-07-10 (see `docs/finding-biar-alert-navigator-end-to-end-id.md` for the full timeline), which broke the CMS query with a 500. The CMS response mapper never read the column, so deleting the reference is the correct fix.
2. **Gold Lakehouse error observability.** When `runSqlQuery` catches an Axios error with a response body, it now logs `error.response.data` alongside the message so operators can see what the upstream BIAR API actually returned instead of a bare `Request failed with status code 500`.
3. **Cases List Auto-closed filter parity.** Adds `STATUS_71_AUTOCLOSED_CONFIRMED` and `STATUS_72_AUTOCLOSED_REFUTED` to the frontend status dropdown and to the `CLOSED_STATUS_VALUES` set that governs which statuses appear when the Cases List is scoped to "Closed Cases". Backend (`case-query.service.ts`, `report.service.ts`, `triage.service.ts`, RBAC matrix, constants) already treats these two statuses as closed — this closes a UI/backend drift.
4. **Ageing chart tick rounding.** `CaseAgeingBarChart` X-axis ticks now render as `${Math.round(value)}%` instead of `${value}%`, preventing fractional-percent labels when Recharts auto-generates ticks.

The PR targets `dev`, which is correct for this repo's flow. All 17 CI checks (CodeQL, njsscan, DCO, GPG signature verify, encoding-check, hadolint, dependency-review, node-ci build/style/tests, conventional-commits, CodeRabbit) are green. `mergeStateStatus` is `BLOCKED` only because no human approver has reviewed yet.

| File | Nature of Change |
|------|-----------------|
| `backend/src/modules/gold-lakehouse/alerts-lakehouse.service.ts` | Remove `anh.end_to_end_id` from Alert Navigator SELECT + GROUP BY (2 lines) |
| `backend/src/modules/gold-lakehouse/gold-lakehouse.service.ts` | Import `isAxiosError`; log upstream response body on Axios failures |
| `backend/test/gold-lakehouse.service.spec.ts` | Add spec asserting upstream body is logged; strip UTF-8 BOM from file head |
| `frontend/src/features/cases/components/CaseFilters.tsx` | Add two AUTOCLOSED statuses to dropdown and CLOSED_STATUS_VALUES |
| `frontend/src/features/reports/components/CaseAgeingBarChart.tsx` | `Math.round` X-axis tick values |

[↑ Back to top](#pr-review-cms-249--fix-surface-auto-closed-cases-in-cases-list-filters-and-improve-gold-lakehouse-error-visibility)

---

## What Changed (Detailed)

### 1. `backend/src/modules/gold-lakehouse/alerts-lakehouse.service.ts` — drop dead `anh.end_to_end_id` from Alert Navigator SQL

Two identical removals — one in `SELECT`, one in `GROUP BY` — of the same column reference in `getAlertNavigatorData()`.

```diff
             anh.transaction_amount,
             anh.transaction_currency,
             anh.transaction_id,
-            anh.end_to_end_id,
             anh.block_or_override_status,
             anh.alert_date,
             COLLECT_LIST(
```

```diff
             anh.transaction_amount,
             anh.transaction_currency,
             anh.transaction_id,
-            anh.end_to_end_id,
             anh.block_or_override_status,
             anh.alert_date
       `;
```

**Why this fix is correct.** Per `docs/finding-biar-alert-navigator-end-to-end-id.md`, `alert_navigator_header` originally did not carry `end_to_end_id`. BIAR commit `035b8db` (2026-05-11, Hassan Rizwan) added it via a pacs002→pacs008 bridge join. BIAR commit `4d00a0a` (2026-07-10, Hassan Rizwan) dropped it again when the bridge join was rewritten from inner to outer join. The CMS query in `992bd5a2` (April 2026) had always carried `anh.end_to_end_id` in the SELECT/GROUP BY as a copy-paste artifact from a pre-refactor query that read from `transactions` — but the response mapper (`alertMetadata`, lines 154–165) never actually read `data.end_to_end_id`. So between May 11 and July 10, the column was selected but ignored; after July 10, the column vanished server-side and the query started returning a 500.

**Consumer verification.** I grepped every consumer of the Alert Navigator response:
- `alertMetadata` mapping (lines 154–165) — reads `alert_id`, `transaction_id`, `alert_timestamp`, `tx_type`, `transaction_amount`, `transaction_currency`, `alert_status`, `alert_reason`, `block_or_override_status`, `evaluation_id`. **Does not read `end_to_end_id`.**
- `AlertNavigatorDataResponse` type (`backend/src/modules/gold-lakehouse/types/alert-navigator.types.ts`) — no `endToEndId` field.
- Frontend `alert-navigator` feature — does not read an `endToEndId` from this response.

No consumer breaks. The `stripHudiMetadata` step (line 152) is unaffected.

### 2. `backend/src/modules/gold-lakehouse/gold-lakehouse.service.ts` — log upstream response body on Axios failure

New import:

```diff
 import { firstValueFrom } from 'rxjs';
+import { isAxiosError } from 'axios';
```

Modified catch block in `runSqlQuery()`:

```diff
     } catch (error: unknown) {
       const errorMessage = error instanceof Error ? error.message : 'Unknown error';
-      this.logger.error(`Error running SQL query: ${errorMessage}`);
+
+      if (isAxiosError(error) && error.response?.data !== undefined) {
+        this.logger.error(`Error running SQL query: ${errorMessage} - upstream response: ${JSON.stringify(error.response.data)}`);
+      } else {
+        this.logger.error(`Error running SQL query: ${errorMessage}`);
+      }

       if (error instanceof HttpException) {
         throw error;
```

**Effect.** When BIAR returns a 4xx/5xx with a JSON body (e.g. a Spark parser complaint or a table-not-found error), the operator now sees the body in logs instead of just `Request failed with status code 500`. The `isAxiosError` type guard from the axios package narrows `error.response` correctly for TypeScript. The non-Axios path is unchanged.

**Behaviour on non-Axios errors** (ECONNREFUSED, generic `Error`, `HttpException`) — falls through to the `else` branch identically to before. `HttpException` re-throw below the log is unchanged.

### 3. `backend/test/gold-lakehouse.service.spec.ts` — add spec for new Axios logging branch; strip BOM

The file previously began with a UTF-8 BOM (`﻿`) before the `import` statement. The diff shows it stripped (the encoding-check CI is now green on this file). The added spec:

```typescript
it('logs the upstream response body on an Axios failure', async () => {
  const axiosError = Object.assign(new Error('Request failed with status code 500'), {
    isAxiosError: true,
    response: { data: { error: 'Spark: table not found' }, status: 500 },
  });
  http.mockReturnValue(throwError(() => axiosError));
  const errorSpy = jest.spyOn(service['logger'], 'error');

  await expect(service.runSqlQuery('BAD')).rejects.toThrow(HttpException);

  expect(errorSpy).toHaveBeenCalledWith(expect.stringContaining('Spark: table not found'));
});
```

The `isAxiosError: true` marker is what axios's runtime `isAxiosError()` helper checks — so the mock exercises the new branch faithfully. The spec asserts the upstream body content reaches `logger.error`.

**Absence-of-line note.** There is no *negative* spec asserting the fallback branch (non-Axios error) does *not* include an "upstream response" phrase — but the prior `wraps generic errors` and `re-throws HttpException` specs already cover the non-Axios paths for behaviour, so this is a minor gap, not a real regression risk.

### 4. `frontend/src/features/cases/components/CaseFilters.tsx` — surface Auto-closed statuses in Cases List

Two additions in `CLOSED_STATUS_VALUES`:

```diff
 const CLOSED_STATUS_VALUES = [
   'STATUS_82_CLOSED_CONFIRMED',
   'STATUS_83_CLOSED_INCONCLUSIVE',
   'STATUS_81_CLOSED_REFUTED',
+  'STATUS_71_AUTOCLOSED_CONFIRMED',
+  'STATUS_72_AUTOCLOSED_REFUTED',
   'STATUS_99_ABANDONED',
 ];
```

Two additions in `statusOptions`:

```diff
     { value: 'STATUS_10_ASSIGNED', label: 'Assigned' },
+    { value: 'STATUS_71_AUTOCLOSED_CONFIRMED', label: 'Auto-closed - Confirmed' },
+    { value: 'STATUS_72_AUTOCLOSED_REFUTED', label: 'Auto-closed - Refuted' },
     { value: 'STATUS_82_CLOSED_CONFIRMED', label: 'Closed - Confirmed' },
```

**Backend parity confirmed.** The two statuses are already treated as closed in the backend at every relevant callsite:
- `case-query.service.ts` — the `closedOnly` branch includes both AUTOCLOSED statuses (lines 336–337), the `excludeClosed` branch mirrors it (lines 355–356), and status enum search covers them (lines 456–457).
- `report.service.ts` — `outcomeMap` includes `autoclosedConfirmed` / `autoclosedRefuted` and both branches of "confirmed vs refuted" bucketing include the AUTOCLOSED equivalents.
- `constants/case.constants.ts` — `CLOSED_STATUSES` list includes both.
- `rbac/permissionMatrix.json` — role-transition matrix covers both.

The frontend was the drift point. This PR closes it.

**Sort order.** The new entries are placed alphabetically ("Auto-closed …" between "Assigned" and "Closed …"), matching the existing ordering convention in the array.

**Case-type-filter interaction.** `filteredStatusOptions` (line 151) restricts the dropdown to `CLOSED_STATUS_VALUES` when `caseTypeFilter === 'closed'`. With the two new statuses added to that set, they now correctly appear only in the "Closed Cases" scope. The `useEffect` at line 165 that clears an invalid statusFilter when switching to "closed" also honours the widened set.

### 5. `frontend/src/features/reports/components/CaseAgeingBarChart.tsx` — round X-axis tick values

```diff
           <XAxis
             type="number"
             domain={[0, 100]}
-            tickFormatter={(value: number) => `${value}%`}
+            tickFormatter={(value: number) => `${Math.round(value)}%`}
             tick={{ fontSize: 11 }}
```

**Effect.** Recharts can auto-generate fractional tick values (e.g. `33.333`) when it chooses an axis interval that doesn't divide `[0, 100]` cleanly. Rounding to integer percent keeps labels tidy. The chart's underlying bar values are unchanged — this is purely a label-formatting cosmetic.

[↑ Back to top](#pr-review-cms-249--fix-surface-auto-closed-cases-in-cases-list-filters-and-improve-gold-lakehouse-error-visibility)

---

## Code Quality Analysis

### Strengths

- **Focused, minimal diffs.** Each change is the smallest edit that fixes the corresponding bug. No collateral refactors.
- **Root-cause fix for the Alert Navigator regression.** The dead column is deleted rather than papered over (e.g. no `try/catch` swallowing the 500 or fallback query). The `docs/finding-biar-alert-navigator-end-to-end-id.md` note that pairs with this PR captures the BIAR-side history so the "why the query used to work" is preserved.
- **Backend/UI parity restored** for AUTOCLOSED statuses without touching backend code. This is the right side of the drift to fix — backend has been the source of truth for closed-status membership across multiple files.
- **Error-logging improvement is properly type-guarded.** `isAxiosError(error) && error.response?.data !== undefined` is a clean narrowing that avoids logging `undefined` and avoids leaking the raw Axios request object.
- **New test covers the new branch.** The Axios-error case is exercised, and the assertion (`stringContaining('Spark: table not found')`) locks in the upstream-body-in-log behaviour rather than just structural expectations.
- **BOM removed from spec file**, unblocking the encoding-check CI on this file.
- **CI is green across the board** including CodeQL and njsscan.

### Issues and Observations

#### Issue 1 — PR title/scope covers only two of the four changes

**Severity: Informational (Code Quality)**

The PR title advertises "Auto-closed cases in Cases List filters and Gold Lakehouse error visibility," but the diff also contains the Alert Navigator `end_to_end_id` removal (the largest functional fix) and the Case Ageing chart tick rounding. Neither is mentioned in the title. This is not a blocker — the changes are small and all dashboard-related — but a future git-log reader searching for the Alert Navigator regression fix won't find it via `git log --oneline`. Consider mentioning both in the PR description before merge, or leaving a follow-up commit-message note.

#### Issue 2 — `JSON.stringify(error.response.data)` size + PII exposure risk in logs

**Severity: Minor (Security / Observability)**

The new log line is:

```typescript
this.logger.error(`Error running SQL query: ${errorMessage} - upstream response: ${JSON.stringify(error.response.data)}`);
```

Two small concerns:

1. **Unbounded size.** If BIAR ever returns a large error body (e.g. a stack trace with echoed rows), the log line grows unbounded. For a Spark/Hudi API this is unlikely to be more than a few KB per error, so this is more a hygiene note than a real risk. If log lines get truncated by the logger backend, a `.slice(0, 2000)` guard would be defensive.
2. **PII exposure.** Error bodies from Spark/Hudi typically contain the SQL that failed (and sometimes offending row data). This service already logs the full SQL earlier (`this.logger.log(finalSql)` at line 107), so the incremental exposure is small — but if the upstream ever changes to echo request payloads, the new log line will surface that too. Not blocking, but worth being aware of when reviewing the log destination's audience.

**Suggested cap** (non-blocking):

```typescript
const preview = JSON.stringify(error.response.data).slice(0, 2000);
this.logger.error(`Error running SQL query: ${errorMessage} - upstream response: ${preview}`);
```

#### Issue 3 — Ageing chart tick rounding is a display-only fix; no visible regression class

**Severity: Informational (Code Quality)**

`Math.round` on the axis label doesn't change the plotted geometry, and Recharts already picks tick positions that fit a `[0, 100]` domain reasonably. This is a low-signal fix but harmless. No follow-up needed.

#### Issue 4 — No test for the frontend filter change

**Severity: Minor (Test Coverage)**

The Cases List filter is the most user-visible change in this PR, and it has no accompanying spec (frontend or E2E). The change is trivial and mechanical (two entries added to a constant, two to an array), but a lightweight test that exercises `filteredStatusOptions` when `caseTypeFilter === 'closed'` and asserts both AUTOCLOSED entries appear would lock the fix in against a future accidental removal. Not blocking — the diff is inspectable end-to-end.

[↑ Back to top](#pr-review-cms-249--fix-surface-auto-closed-cases-in-cases-list-filters-and-improve-gold-lakehouse-error-visibility)

---

## Security Assessment

| Concern | Assessment |
|---------|-----------|
| SQL injection in Alert Navigator query | Unchanged — `safeAlertId` via `clampPositiveInteger`, `safeTenantId` via `escapeSqlString`. The removed column is a static identifier, not user input. No new injection surface. |
| SQL injection in other lakehouse queries | Unchanged — none of the other files touch SQL builders. |
| Authorization / tenant scoping | Unchanged — `getAlertNavigatorData` still enforces `anh.tenant_id = '${safeTenantId}'` at line 121 and in `rules_agg` at line 59. The removed column has no auth semantics. |
| Auth-guard decorators on new endpoints | N/A — no new endpoints. |
| URL construction / SSRF | N/A — no new URL construction. The existing `${this.apiUrl}/execute_sql` call is unchanged. |
| XSS / HTML rendering | N/A — no new `dangerouslySetInnerHTML` or DOM string injection. The new `statusOptions` labels are static strings rendered through React's default escaping. |
| Secrets / PII in error logs | New: `JSON.stringify(error.response.data)` logged when upstream fails. See Issue 2 — the surface is bounded by whatever BIAR includes in its error responses; SQL was already logged upstream in the same method, so the incremental exposure is small. No blocker. |
| Frontend status enum tampering | The two new frontend status values are only used as filter selections sent to the backend, which validates them via Prisma-typed enums in `case-query.service.ts`. An attacker sending an arbitrary status value already gets validated server-side. No new trust boundary. |
| Rate limits / DoS | Unchanged — the logging change is on the error path only, no new outbound calls. |
| CI security-scanner status | CodeQL (actions + javascript-typescript), njsscan, and dependency-review all pass. |

No new vulnerabilities introduced by this PR. The Alert Navigator SQL change actively reduces attack surface (one fewer column selected from upstream), though not for any security reason.

[↑ Back to top](#pr-review-cms-249--fix-surface-auto-closed-cases-in-cases-list-filters-and-improve-gold-lakehouse-error-visibility)

---

## Test Coverage

**Covered by this PR:**
- Backend: the new Axios-error logging branch has a dedicated spec that asserts the upstream response body reaches `logger.error`. The existing suite continues to cover the non-Axios error paths (`ECONNREFUSED`, generic `Error`, `HttpException` re-throw).
- CI: full `node-ci / check tests` pass at HEAD `c2ca0301`.

**Not covered by this PR:**
- **Frontend filter change** — no unit or E2E test for `CaseFilters.tsx` asserting that the two new AUTOCLOSED entries appear in the dropdown and are honoured by `filteredStatusOptions` when `caseTypeFilter === 'closed'`. See Issue 4. Non-blocking because the diff is a two-line data change.
- **Alert Navigator SQL fix** — no integration test asserts that the query executes cleanly against a `alert_navigator_header` view that lacks `end_to_end_id`. This would require a lakehouse fixture; the existing unit tests mock `runSqlQuery` and don't validate SQL text. The regression that motivated this fix is only observable end-to-end. Adding a snapshot test on the generated SQL would lock the column removal in place — worth considering as a follow-up.
- **Case Ageing chart tick formatter** — no spec asserts `Math.round(33.333)` produces `"33%"`. Trivial; not worth a dedicated test.

**Checklist state.** The PR description body is short (three-line summary, no checkboxes template in this repo). CodeRabbit's 5 pre-merge checks all pass (Docstring Coverage, Linked Issues, Out-of-Scope, Description, Title). No coverage report screenshot attached; not customary for this repo.

Every issue in this review is `Minor` or `Informational` — there is no `Major` to bind a regression test to.

[↑ Back to top](#pr-review-cms-249--fix-surface-auto-closed-cases-in-cases-list-filters-and-improve-gold-lakehouse-error-visibility)

---

## CodeRabbit Activity

CodeRabbit posted only its automated walkthrough / pre-merge-checks comment — no actionable findings, no inline code comments, no review at severity above informational. `gh api pulls/249/reviews` returns `[]` and `gh api pulls/249/comments` returns `[]`.

### Pass 1 — Auto-walkthrough

**Commit reviewed:** `c2ca0301` (via post-push walkthrough)
**Findings:** 0 actionable comments

| Finding | Severity | Status |
|---------|----------|--------|
| — | — | — |

Pre-merge checks:

| Check | Status |
|-------|--------|
| Docstring Coverage | ✅ Skipped (no functions in changed files require docstrings per CodeRabbit's rubric) |
| Linked Issues | ✅ Skipped (none linked) |
| Out of Scope Changes | ✅ Skipped (no linked issues) |
| Description | ✅ Skipped (CodeRabbit summary enabled) |
| Title | ✅ Accurate for the two headline changes |

Nothing to reconcile against my own findings.

[↑ Back to top](#pr-review-cms-249--fix-surface-auto-closed-cases-in-cases-list-filters-and-improve-gold-lakehouse-error-visibility)

---

## Summary and Verdict

**Verdict: Approved**

This is a clean, tightly-scoped set of four dashboard bug fixes. The Alert Navigator SQL change is the most important — it fixes a live 500 caused by an upstream BIAR schema change on 2026-07-10, and the fix (delete the dead column reference) is the correct one because CMS never read the column. The Cases List filter change closes a UI/backend drift that had been surfacing "Closed Cases" without two of the six closed-status variants. The error-logging change is a small but real observability win. The Case Ageing tick rounding is cosmetic. All four CI scanners are green, the new logging branch has a spec, and no security surface is expanded.

### Blocking

None.

### Non-blocking but recommended

1. **Consider capping the upstream-response log line** (see Issue 2) — a `.slice(0, 2000)` guard against pathological BIAR error bodies. Not required for merge.
2. **Consider a lightweight test for the AUTOCLOSED filter entries** (see Issue 4) — either a Jest render-test on `CaseFilters` or an E2E assertion that the "Closed Cases" scope shows six statuses. Locks the mechanical change in against future edits.
3. **Mention the Alert Navigator SQL fix in the PR description or a commit-message note** (see Issue 1) so the git history is greppable for the 2026-07-16 regression fix.

[↑ Back to top](#pr-review-cms-249--fix-surface-auto-closed-cases-in-cases-list-filters-and-improve-gold-lakehouse-error-visibility)

---

## GitHub Review Comment

````markdown
**Approved**

Four small, focused dashboard fixes bundled together: (a) remove dead `anh.end_to_end_id` from the Alert Navigator SQL (fixes the live 500 caused by BIAR dropping the column on 2026-07-10 — CMS never read the column, so deleting the reference is the correct fix), (b) add `STATUS_71_AUTOCLOSED_CONFIRMED` / `STATUS_72_AUTOCLOSED_REFUTED` to the Cases List filter and `CLOSED_STATUS_VALUES` (backend already treats them as closed in `case-query.service.ts`, `report.service.ts`, `constants/case.constants.ts`, and RBAC — this closes a UI drift), (c) log upstream response body on Axios failures in `runSqlQuery` for observability, (d) `Math.round` the ageing-chart X-axis ticks. CI is green (17/17 checks). No blockers.

---

### Non-blocking (please address in this PR if possible)

**1. Cap the upstream-response log line to guard against pathological error bodies**

`backend/src/modules/gold-lakehouse/gold-lakehouse.service.ts:137-141`. If BIAR ever returns a large error body, the log line grows unbounded. A defensive `.slice(0, 2000)` (or similar) keeps a preview without risking log-backend truncation of surrounding entries:

```typescript
if (isAxiosError(error) && error.response?.data !== undefined) {
  const preview = JSON.stringify(error.response.data).slice(0, 2000);
  this.logger.error(`Error running SQL query: ${errorMessage} - upstream response: ${preview}`);
} else {
  this.logger.error(`Error running SQL query: ${errorMessage}`);
}
```

Also note the full SQL is already logged upstream at line 107, so the incremental PII exposure of the response body is small — but worth being aware of when reviewing where these logs land.

**2. Consider a lightweight test for the AUTOCLOSED filter entries**

`frontend/src/features/cases/components/CaseFilters.tsx:14-21, 94-95`. The change is mechanical (two entries in a constant, two in an array) but it's the most user-visible fix in the PR and has no spec. A Jest render-test asserting that under `caseTypeFilter === 'closed'` the status dropdown offers six entries (both AUTOCLOSED plus the four existing closed statuses) would lock the fix against a future accidental deletion. Not required for merge.

**3. Mention the Alert Navigator SQL fix in the PR description**

Right now the PR title only advertises the Cases List filter + Gold Lakehouse error visibility. The dead-column removal in `alerts-lakehouse.service.ts` is the largest functional fix in the diff and fixes a live 500 — a one-line note in the description would make the change greppable in `git log` for anyone hunting the 2026-07-16 regression later. See `docs/finding-biar-alert-navigator-end-to-end-id.md` in the reviewer's workspace for the full BIAR timeline that motivated the fix.
```
````

[↑ Back to top](#pr-review-cms-249--fix-surface-auto-closed-cases-in-cases-list-filters-and-improve-gold-lakehouse-error-visibility)
