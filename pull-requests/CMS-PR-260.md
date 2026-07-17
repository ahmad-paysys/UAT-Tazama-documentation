# PR Review: CMS #260 — fix: case ageing investigator scope to owned and assigned cases only

**Repo:** tazama-lf/case-management-system
**Branch:** `paysys/investigator-scope-258` → `dev`
**Author:** MuhammadAli-Paysys (Muhammad Ali)
**Date Reviewed:** 2026-07-17
**Label:** bug
**Size:** +84 / -3 lines across 2 files
**Commits:** 1 (399a2d98)
**State:** OPEN
**HEAD SHA verified:** `399a2d98e3c86f9c8abf5792cb73da5cb0fce855`
**Existing approvals:** none — CodeRabbit COMMENTED (walkthrough only, no actionable findings), no human approval yet

## Table of Contents

- [Overview](#overview)
- [What Changed (Detailed)](#what-changed-detailed)
  - [1. backend/src/modules/report/report.service.ts — add `applyOwnedWorkScope` helper and switch three `getCaseAgeing` call sites to it](#1-backendsrcmodulesreportreportservicets--add-applyownedworkscope-helper-and-switch-three-getcaseageing-call-sites-to-it)
  - [2. backend/test/report.service.spec.ts — three tests locking the new scope in place](#2-backendtestreportservicespects--three-tests-locking-the-new-scope-in-place)
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

This PR narrows the investigator scope on the Case Ageing Report from four arms down to two, exactly as scoped by parent issues [#258](https://github.com/tazama-lf/case-management-system/issues/258) (dev sub-issue) and [#259](https://github.com/tazama-lf/case-management-system/issues/259) (QA sub-issue) under Case Ageing quirk 6 (#217).

The existing shared helper `applyInvestigatorScope` (report.service.ts:173) unions four arms for an investigator: task-assigned, case-owner, every `STATUS_00_DRAFT`/`STATUS_02_READY_FOR_ASSIGNMENT` case regardless of owner (claimable pool), and every unowned `STATUS_01_PENDING_CASE_CREATION_APPROVAL` case. Arms 3 and 4 polluted the Case Ageing widgets — an investigator saw the entire tenant's claimable backlog counted into their own average age, tier counts, and details table. This PR introduces a narrower helper `applyOwnedWorkScope` that keeps only the first two arms (task-assigned OR case-owner) and swaps it in at the three `getCaseAgeing` call sites (`openWhere`, `closedWhere`, `trendWhere`). The original `applyInvestigatorScope` is intentionally preserved for the ~10 other report methods (dashboard, case-status, evidence-findings, investigator-workload) that legitimately show the claimable pool.

Targets `dev` — correct for this repo's flow. All CI checks pass (CodeQL, njsscan, dependency-review, node-ci build/style/tests, DCO, GPG, hadolint, conventional-commits).

| File | Nature of Change |
|------|-----------------|
| `backend/src/modules/report/report.service.ts` | Adds `applyOwnedWorkScope` helper (18 lines) and swaps three `applyInvestigatorScope` calls to it inside `getCaseAgeing`. No other methods touched. |
| `backend/test/report.service.spec.ts` | Adds three tests: (a) investigator query has exactly two owned-work arms with no claimable/unowned leakage; (b) supervisor/admin path leaves where-clause unwrapped; (c) a claimable STATUS_02 case owned by another investigator is excluded by construction. |

[↑ Back to top](#pr-review-cms-260--fix-case-ageing-investigator-scope-to-owned-and-assigned-cases-only)

---

## What Changed (Detailed)

### 1. `backend/src/modules/report/report.service.ts` — add `applyOwnedWorkScope` helper and switch three `getCaseAgeing` call sites to it

New helper inserted between `applyInvestigatorScope` and `avgResolutionDays`:

```typescript
private applyOwnedWorkScope(baseFilters: any, requestingUserId?: string): any {
  if (!requestingUserId) return baseFilters;

  return {
    AND: [
      baseFilters,
      {
        OR: [
          // Tasks assigned to the user
          { tasks: { some: { assigned_user_id: requestingUserId } } },
          // Case owner is the user
          { case_owner_user_id: requestingUserId },
        ],
      },
    ],
  };
}
```

Three call-site swaps inside [getCaseAgeing](repos/case-management-system/backend/src/modules/report/report.service.ts#L1096):

```diff
- const openWhere = this.applyInvestigatorScope(
+ const openWhere = this.applyOwnedWorkScope(
    { ...commonFilters, status: { notIn: ReportsService.CLOSED_STATUSES } },
    filters?.requestingUserId,
  );
```

```diff
- const closedWhere = this.applyInvestigatorScope(
+ const closedWhere = this.applyOwnedWorkScope(
    {
      ...commonFilters,
      status: { in: ReportsService.CLOSED_STATUSES },
      updated_at: { gte: startDate, lte: endDate },
    },
    filters?.requestingUserId,
  );
```

```diff
- const trendWhere = this.applyInvestigatorScope(
+ const trendWhere = this.applyOwnedWorkScope(
    {
      ...commonFilters,
      status: { in: ReportsService.CLOSED_STATUSES },
      updated_at: { gte: trendMonths[0] },
    },
    filters?.requestingUserId,
  );
```

**Behavioural effect.** For investigator sessions (controller sets `requestingUserId = userId` only when `isInvestigatorOnly(userClaims)` — [report.controller.ts:561](repos/case-management-system/backend/src/modules/report/report.controller.ts#L561)):

- Open backlog (avgCaseAge, casesOver15/30Days, ageingByStatus, ageingDistribution, caseDetails): only cases the investigator owns or has a task on.
- Closed throughput (avgResolutionTime, caseTypeResolution): only cases the investigator owns/has a task on that were closed in the window.
- Resolution trend (6-month p25/median/p75): same narrowing over the 6-month window.

For supervisor/admin sessions (`requestingUserId === undefined`) the helper returns `baseFilters` verbatim — no behaviour change.

Note the parent's [#258](https://github.com/tazama-lf/case-management-system/issues/258) explicit scope guardrail: `applyInvestigatorScope` is used by ~10 other report methods and must **not** be changed globally. This PR honours that: [grep confirms](repos/case-management-system/backend/src/modules/report/report.service.ts) `applyInvestigatorScope` still appears at lines 302, 303, 427, 503, 520, 535 (createdWhere/closedWhere in one report, three sites in the case-status report, and three sites in the dashboard). Only the three Case Ageing sites (1138, 1218, 1257) migrated.

### 2. `backend/test/report.service.spec.ts` — three tests locking the new scope in place

Three tests added inside `describe('getCaseAgeing', …)`:

```typescript
it('scopes an investigator request to exactly the two owned-work arms (no claimable/unowned pool)', async () => {
  await service.getCaseAgeing('last30', { tenantId: 'tenant-123', requestingUserId: 'user-123' });

  const whereClauses = prismaService.case.findMany.mock.calls.map(([args]: [any]) => args.where);
  expect(whereClauses.length).toBeGreaterThanOrEqual(3);
  whereClauses.forEach((where) => {
    // Wrapped in AND[baseFilters, OR[...]] with exactly two arms.
    expect(where.AND).toBeDefined();
    const orClause = where.AND.find((clause: any) => clause.OR !== undefined);
    expect(orClause).toBeDefined();
    expect(orClause.OR).toHaveLength(2);
    expect(orClause.OR).toEqual(
      expect.arrayContaining([
        { tasks: { some: { assigned_user_id: 'user-123' } } },
        { case_owner_user_id: 'user-123' },
      ]),
    );
    // No claimable/unowned arms leak in.
    const orJson = JSON.stringify(orClause.OR);
    expect(orJson).not.toContain('STATUS_00_DRAFT');
    expect(orJson).not.toContain('STATUS_02_READY_FOR_ASSIGNMENT');
    expect(orJson).not.toContain('STATUS_01_PENDING_CASE_CREATION_APPROVAL');
  });
});
```

Key design choices worth flagging:

- `whereClauses.forEach(...)` asserts on **every** `findMany` call, not just one. This is the "every-mock-call assertion" pattern from the review instructions' 3.1 parallel-siblings hunt — it protects against a future refactor where one of the three call sites silently reverts to `applyInvestigatorScope`.
- The status-name string-contains assertion doubles as an inverted parallel-siblings check: if the claimable pool leaks back in through a different mechanism (e.g. a nested OR under `commonFilters`), the JSON scan still catches it.

Second test — supervisor/admin unwrapping:

```typescript
it('leaves the where-clause unwrapped for a supervisor/admin request (requestingUserId undefined)', async () => {
  await service.getCaseAgeing('last30', { tenantId: 'tenant-123' });

  const whereClauses = prismaService.case.findMany.mock.calls.map(([args]: [any]) => args.where);
  expect(whereClauses.length).toBeGreaterThanOrEqual(3);
  whereClauses.forEach((where) => {
    const json = JSON.stringify(where);
    expect(json).not.toMatch(/"tasks":\s*\{\s*"some":\s*\{\s*"assigned_user_id"/);
  });
});
```

Third test — construction-level exclusion of another investigator's claimable case:

```typescript
it('excludes a STATUS_02_READY_FOR_ASSIGNMENT case owned by another investigator from the investigator result', async () => {
  prismaService.case.findMany.mockResolvedValue([]);
  await service.getCaseAgeing('last30', { tenantId: 'tenant-123', requestingUserId: 'user-123' });

  const openCall = prismaService.case.findMany.mock.calls[0];
  expect(openCall).toBeDefined();
  const where = (openCall[0] as any).where;
  const orClause = where.AND.find((clause: any) => clause.OR !== undefined);
  expect(orClause.OR).toEqual([
    { tasks: { some: { assigned_user_id: 'user-123' } } },
    { case_owner_user_id: 'user-123' },
  ]);
});
```

This one deliberately asserts on where-shape rather than result contents (since the prisma mock returns whatever it's handed, a result-based assertion would be vacuous). The reasoning is spelled out in the inline comment.

[↑ Back to top](#pr-review-cms-260--fix-case-ageing-investigator-scope-to-owned-and-assigned-cases-only)

---

## Code Quality Analysis

### Strengths

- **Faithful to the design in #258.** The helper's arms, name, and placement match the code sketch in the issue verbatim. The three call-site swaps land at exactly the three lines the issue cited.
- **Scope discipline.** The PR resists the temptation to also convert other reports (evidence findings, investigator workload, case status) that share `applyInvestigatorScope`. Those reports intentionally include the claimable pool; leaving them alone is correct per #258's explicit guardrail.
- **Parallel-siblings safety.** All three Case Ageing sites converted together — no orphan call still using the wide helper (grep confirms).
- **Test coverage locks the intended behaviour.** The primary test asserts on every `findMany` call, not just one — the exact pattern needed to catch a future partial regression. It also enumerates the negative arms by status-name string, so a leak through a different code path still trips it.
- **Supervisor/admin path is separately tested.** The unwrapped-clause test guarantees the fix doesn't accidentally scope supervisor views.
- **Small, focused diff.** 1 helper, 3 call-site swaps, 3 tests. No incidental refactor, no unrelated churn.

### Issues and Observations

#### Issue 1 — `applyOwnedWorkScope` has no JSDoc explaining why it exists alongside `applyInvestigatorScope`

**Severity: Minor (Maintainability)**

The existing `applyInvestigatorScope` has a substantial JSDoc block ([report.service.ts:158-172](repos/case-management-system/backend/src/modules/report/report.service.ts#L158-L172)) explaining its four arms and design intent. The new `applyOwnedWorkScope` has no comment, so a future reader hitting it in isolation will not know:

1. Why there are two near-identical helpers.
2. That this one is deliberately narrower and used only by the Case Ageing Report.
3. That the claimable/unowned arms of `applyInvestigatorScope` are omitted by design, not by oversight.

Without that context, a future well-intentioned change will "unify" the two helpers, quietly re-widening Case Ageing.

**Suggested fix** — port the JSDoc sketch from issue #258:

```typescript
/**
 * Ageing-scoped investigator filter: explicit ownership only.
 * Drops the claimable/unowned arms of applyInvestigatorScope so the
 * Case Ageing Report measures how long each investigator's *own* work
 * has sat. The claimable pool belongs on a separate unassigned-queue
 * view, not summed into any one investigator's backlog. Do not use
 * for any other report — those intentionally include the claimable pool.
 */
private applyOwnedWorkScope(baseFilters: any, requestingUserId?: string): any {
```

#### Issue 2 — `any` on `baseFilters` and return type; `Prisma.CaseWhereInput` is already the file's convention

**Severity: Minor (Code Quality)**

`applyOwnedWorkScope(baseFilters: any, requestingUserId?: string): any` is newly-authored code in this PR and introduces two fresh `any`s into a file that otherwise types every case-where value as `Prisma.CaseWhereInput`. Confirmed by grep:

- `Prisma.CaseWhereInput` is already imported at [report.service.ts:3](repos/case-management-system/backend/src/modules/report/report.service.ts#L3) (`import { …, Prisma, … } from '@prisma/client-cms'`).
- It is the declared type at lines 84, 103, 121, 130-132, 150, 506, 510, 514, and 702 of the same file — including on the sibling helper `withNonContainerCaseFilter(where: Prisma.CaseWhereInput = {}): Prisma.CaseWhereInput` ([report.service.ts:121](repos/case-management-system/backend/src/modules/report/report.service.ts#L121)), and on the shape returned by `buildCommonCaseFilters` ([report.service.ts:150](repos/case-management-system/backend/src/modules/report/report.service.ts#L150)), which is precisely what the caller feeds into `applyOwnedWorkScope`.

So the tighter type is already the norm in this file and directly available at the call sites — there is no import to add and no adjacent code to change. The `any` on `applyOwnedWorkScope` mirrors the `any` on `applyInvestigatorScope` above it, but `applyInvestigatorScope` is pre-existing and out of PR scope; the new helper isn't.

**Suggested fix — in scope, three lines:**

```typescript
private applyOwnedWorkScope(
  baseFilters: Prisma.CaseWhereInput,
  requestingUserId?: string,
): Prisma.CaseWhereInput {
  if (!requestingUserId) return baseFilters;

  return {
    AND: [
      baseFilters,
      {
        OR: [
          { tasks: { some: { assigned_user_id: requestingUserId } } },
          { case_owner_user_id: requestingUserId },
        ],
      },
    ],
  };
}
```

**Compatibility with the three call sites.** All three call sites already pass a `Prisma.CaseWhereInput`-shaped object literal, and each is assigned to a `where` field passed straight to `this.prisma.case.findMany({ where: … })`. The object spread `{ ...commonFilters, status: {…} }` produces a `Prisma.CaseWhereInput` since `buildCommonCaseFilters` returns that type. `Prisma.CaseWhereInput` has an optional `AND?: CaseWhereInput | CaseWhereInput[]` field and an optional `OR?: CaseWhereInput[]` field, so the returned `{ AND: [baseFilters, { OR: […] }] }` object is a valid `CaseWhereInput` under the Prisma-generated type. No call site needs to change and no `as` cast is required.

**Compatibility with the tests.** The test file accesses `where.AND` via `(openCall[0] as any).where` ([report.service.spec.ts:808](repos/case-management-system/backend/test/report.service.spec.ts)) — it already casts to `any` at the assertion boundary, so a tighter return type on the helper does not force any test change.

**Trade-off.** The only "cost" is a five-line inconsistency with `applyInvestigatorScope` immediately above (still `any` in and `any` out). That's a pre-existing wart, and fixing it here is out of scope. Two nearby helpers with different type strictness is a mild inconsistency, but the alternative — a new `any`-typed helper — permanently widens the file. Prefer the local tightening.

[↑ Back to top](#pr-review-cms-260--fix-case-ageing-investigator-scope-to-owned-and-assigned-cases-only)

---

## Security Assessment

| Concern | Assessment |
|---------|-----------|
| Tenant scoping on the narrowed queries | Preserved — `commonFilters` (from `buildCommonCaseFilters(filters)`) carries `tenantId` from the auth token via the controller, and the AND-wrap preserves it in every branch. Not weakened. |
| Role-based scope helper | `applyOwnedWorkScope` correctly reads `requestingUserId` (set by the controller only when `isInvestigatorOnly(userClaims)` — [report.controller.ts:561](repos/case-management-system/backend/src/modules/report/report.controller.ts#L561)). Supervisor/admin path unchanged. |
| Auth guards on `GET /reports/case-ageing` | Unchanged — the controller decorator stack was not modified. |
| Injection / SSRF / XSS | None. No new user input reaches a data or command layer; `requestingUserId` is a UUID from a validated JWT claim, never a query param. Prisma parameterises. |
| PII in logs / responses | Unchanged — no new logging, no new response fields. |
| Direction of change | **Tightening**: investigators now see *fewer* cases than before (loses claimable arms). No path to see *more*. So this change reduces data exposure, not increases it. |
| CI security scanners | CodeQL, njsscan, dependency-review all SUCCESS on HEAD `399a2d98`. |

No new security vulnerabilities introduced by this PR. If anything, this narrows a data-visibility surface for the investigator role.

[↑ Back to top](#pr-review-cms-260--fix-case-ageing-investigator-scope-to-owned-and-assigned-cases-only)

---

## Test Coverage

The PR adds three targeted tests inside the existing `describe('getCaseAgeing', ...)` block. Coverage assessment:

- **New helper is exercised at all three call sites.** The primary test asserts the AND/OR shape on **every** `findMany` mock call (`.forEach`), which locks in that all three of `openWhere`, `closedWhere`, `trendWhere` share the two-arm scope. This is exactly the "every-mock-call assertion" pattern the review guidance recommends for shared filters, and it defends against a future partial revert.
- **Negative arms explicitly locked out.** The `orJson.not.toContain('STATUS_00_DRAFT'|'STATUS_02_READY_FOR_ASSIGNMENT'|'STATUS_01_PENDING_CASE_CREATION_APPROVAL')` assertion means a regression that re-adds any of the claimable/unowned arms fails the test.
- **Supervisor/admin path is covered.** Second test asserts the unwrapped shape and the absence of any `tasks.some.assigned_user_id` fragment.
- **Cross-investigator exclusion is asserted.** Third test's shape check is a valid stand-in for a mock result assertion (the prisma mock is not a real query engine, so a shape-level check is the correct assertion level here).
- **CI status:** `node-ci / check tests` is SUCCESS on HEAD.

Gaps:

- The `caseTypeResolution` and `resolutionTrend` outputs are not asserted end-to-end for the investigator case (i.e. no test that supplies a claimable STATUS_02 case in the mock and verifies it doesn't appear in the closed-throughput / trend widgets). The where-shape assertions imply this by construction, but a result-level test would make the connection to the user-visible widgets explicit. Non-blocking — the where-shape is the right assertion level for a Prisma-mocked service, and the intent is already captured in the third test's inline comment.
- No integration test at the controller layer confirming that supervisors calling `/reports/case-ageing` still receive the full backlog. This is out of scope for this PR (the unit-level assertion in test #2 is sufficient for the where-shape guarantee).

The PR description body is empty; no author-supplied test checklist to compare against. Not blocking, but worth a follow-up to add a short test-plan note to the PR description referencing #259's QA scenarios.

[↑ Back to top](#pr-review-cms-260--fix-case-ageing-investigator-scope-to-owned-and-assigned-cases-only)

---

## CodeRabbit Activity

**Independent analysis was drafted before opening CodeRabbit** — nothing here anchored my findings. Reconciling:

### Pass 1 — walkthrough only

**Commit reviewed:** `399a2d98`
**Findings:** 0 actionable comments

CodeRabbit posted only a walkthrough / change summary — no line-level findings, suggestions, or nitpicks. Pre-merge checks all passed (Docstring Coverage skipped, Linked Issues skipped, Out-of-Scope skipped, Description skipped, Title passed). Estimated complexity: 2/10.

| Finding | Severity | Status |
|---------|----------|--------|
| (none — summary-only pass) | — | — |

Nothing to reconcile against my independent findings. CodeRabbit did not flag Issue 1 (JSDoc gap) or Issue 2 (`any` typing on the new helper), but both are locally-scoped nits rather than defects.

[↑ Back to top](#pr-review-cms-260--fix-case-ageing-investigator-scope-to-owned-and-assigned-cases-only)

---

## Summary and Verdict

**Verdict: Approved**

This PR is a clean, minimal, and well-scoped fix to Case Ageing Report quirk 6. It faithfully implements the design in issue [#258](https://github.com/tazama-lf/case-management-system/issues/258): a new `applyOwnedWorkScope` helper with only the owned + task-assigned arms, applied at exactly the three `getCaseAgeing` call sites (open backlog, closed throughput, resolution trend). The wider `applyInvestigatorScope` is preserved unchanged for the ~10 other report methods that legitimately show the claimable pool. Test coverage is stronger than the existing spec: every `findMany` mock call is asserted, the negative arms are explicitly locked out by status-name string, and both the investigator and supervisor/admin paths are covered. All CI checks pass on HEAD `399a2d98`. Direction of change is data-tightening (investigators see fewer cases, never more), so security posture is not weakened.

### Blocking

*(none)*

### Non-blocking but recommended

1. **Add a JSDoc block to `applyOwnedWorkScope`** — explain why it exists alongside `applyInvestigatorScope` and note it must not be used by other reports, so a future consolidation attempt doesn't quietly re-widen Case Ageing. Suggested wording ports the sketch from #258 (see Issue 1).
2. **Type `applyOwnedWorkScope` with `Prisma.CaseWhereInput` instead of `any`** — the type is already imported and is the file's convention for every other case-where value (10+ existing uses, including the sibling helper `withNonContainerCaseFilter`). The three call sites and the tests are already compatible, so this is a three-line, zero-risk change local to the new helper. (See Issue 2.)

[↑ Back to top](#pr-review-cms-260--fix-case-ageing-investigator-scope-to-owned-and-assigned-cases-only)

---

## GitHub Review Comment

````markdown
**Approved**

Clean and minimal fix for Case Ageing quirk 6 (#217 / #258). New `applyOwnedWorkScope` helper (owned + task-assigned only) swapped in at exactly the three `getCaseAgeing` call sites; the wider `applyInvestigatorScope` is correctly left untouched for the other reports that intentionally include the claimable pool. Tests assert the two-arm shape on **every** `findMany` mock call (defends against a future partial revert) and cover the supervisor/admin unwrap path. All CI green on `399a2d98`. No blocking items.

---

### Non-blocking (please address in this PR if possible)

**1. Add a JSDoc block to `applyOwnedWorkScope`**

`applyInvestigatorScope` (report.service.ts:158-172) has a full docblock; `applyOwnedWorkScope` (report.service.ts:207) has none. Without it, a future reader will not know the two helpers are deliberately different and may "unify" them, silently re-widening Case Ageing. Suggested wording (ported from #258):

```typescript
/**
 * Ageing-scoped investigator filter: explicit ownership only.
 * Drops the claimable/unowned arms of applyInvestigatorScope so the
 * Case Ageing Report measures how long each investigator's *own* work
 * has sat. The claimable pool belongs on a separate unassigned-queue
 * view, not summed into any one investigator's backlog. Do not use
 * for any other report — those intentionally include the claimable pool.
 */
private applyOwnedWorkScope(baseFilters: any, requestingUserId?: string): any {
```

**2. Type `applyOwnedWorkScope` with `Prisma.CaseWhereInput` instead of `any`**

The new helper introduces two fresh `any`s into a file that otherwise types every case-where value as `Prisma.CaseWhereInput`. The type is already imported at [report.service.ts:3](backend/src/modules/report/report.service.ts#L3) and used at lines 84, 103, 121, 130-132, 150, 506, 510, 514, and 702 — including on the sibling helper `withNonContainerCaseFilter` and on the return of `buildCommonCaseFilters` (which is exactly what feeds `applyOwnedWorkScope`). All three call sites and the tests are already compatible, so no downstream change is needed:

```typescript
private applyOwnedWorkScope(
  baseFilters: Prisma.CaseWhereInput,
  requestingUserId?: string,
): Prisma.CaseWhereInput {
  if (!requestingUserId) return baseFilters;

  return {
    AND: [
      baseFilters,
      {
        OR: [
          { tasks: { some: { assigned_user_id: requestingUserId } } },
          { case_owner_user_id: requestingUserId },
        ],
      },
    ],
  };
}
```

(The pre-existing `applyInvestigatorScope` above it is also `any`-typed but is out of scope for this PR — happy to leave that for a follow-up.)
````

[↑ Back to top](#pr-review-cms-260--fix-case-ageing-investigator-scope-to-owned-and-assigned-cases-only)
