# PR Review: DEMS #43 — feat: Approver claims added for activation

**Repo:** tazama-lf/event-monitoring-service  
**Branch:** `feat-paysys-approver-claim` → `dev`  
**Author:** MuhammadAli-Paysys (Muhammad Ali)  
**Date Reviewed:** 2026-07-02  
**Label:** enhancement  
**Size:** +347 / -4 lines across 3 files  
**Commits:** 2 (`1e18a124`, `186c2c3b`)  
**State:** OPEN  
**Existing approvals:** MAdeel95 — Approved (2026-06-30)

---

## Table of Contents

- [Overview](#overview)
- [What Changed (Detailed)](#what-changed-detailed)
  - [1. src/auth/auth.decorator.ts — New claim and decorator](#1-srcauthauthdecoratorts--new-claim-and-decorator)
  - [2. src/config-notify/config-notify.controller.ts — Endpoint re-gating](#2-srcconfig-notifyconfig-notifycontrollerts--endpoint-re-gating)
  - [3. tests/dems-engine/dems-engine.service.spec.ts — Test expansion](#3-testsdems-enginedems-engineservicespects--test-expansion)
- [Authorization Architecture Analysis](#authorization-architecture-analysis)
  - [How the guard enforces claims](#how-the-guard-enforces-claims)
  - [Impact of the decorator swap](#impact-of-the-decorator-swap)
- [Code Quality Analysis](#code-quality-analysis)
  - [Strengths](#strengths)
  - [Issues and Observations](#issues-and-observations)
- [Security Assessment](#security-assessment)
- [Test Coverage](#test-coverage)
- [CodeRabbit Activity](#coderabbit-activity)
  - [Pass 1 — Decorator change review](#pass-1--decorator-change-review)
  - [Pass 2 — Test expansion review](#pass-2--test-expansion-review)
- [Summary and Verdict](#summary-and-verdict)
- [GitHub Review Comment](#github-review-comment)

---

## Overview

[↑ Back to top](#pr-review-dems-43--feat-approver-claims-added-for-activation)

This PR makes two distinct changes:

1. **Authorization change** — `PATCH /config-notify/:id` (the endpoint that updates a config's `publishing_status` to `active`/`inactive`) is switched from requiring the `dems:write` claim to requiring an `approver` claim. A new `APPROVER` constant and `RequireActivationRole()` decorator are added to support this.

2. **Test coverage expansion** — 14 new test cases are added to `dems-engine.service.spec.ts`, covering previously untested branches in mapping logic, configured function execution, error handling, pacs.002 processing, related transaction lookup, and AJV validation.

The motivation is clearly stated: approvers were unable to activate/deactivate configs on TCS because the endpoint was locked behind `dems:write`, a claim approvers do not hold. The fix correctly addresses that access gap.

The PR description checklist is well-completed: locally tested, dev environment tested, Husky run, and unit tests passing — with a coverage screenshot provided.

---

## What Changed (Detailed)

[↑ Back to top](#pr-review-dems-43--feat-approver-claims-added-for-activation)

### 1. src/auth/auth.decorator.ts — New claim and decorator

[↑ Back to top](#pr-review-dems-43--feat-approver-claims-added-for-activation)

```diff
 export const EventMonitoringClaims = Object.freeze({
   DEMS_WRITE: 'dems:write',
+  APPROVER: 'approver',
 } as const);

 export const RequireDemsWriteRole = (): ReturnType<typeof SetMetadata> =>
   RequireClaims(EventMonitoringClaims.DEMS_WRITE);
+
+export const RequireActivationRole = (): ReturnType<typeof SetMetadata> =>
+  RequireClaims(EventMonitoringClaims.APPROVER);
```

The new `APPROVER: 'approver'` claim is registered on the frozen `EventMonitoringClaims` object alongside `DEMS_WRITE`. The decorator follows the exact same pattern as `RequireDemsWriteRole` — it wraps `RequireClaims` with the specific claim string. This is consistent and correct.

The claim string value `'approver'` must match exactly what the identity provider issues in the JWT. This is an external contract — if the JWT contains `'APPROVER'` or `'approver_role'`, the guard will deny access silently. The match is case-sensitive (the guard calls `validateTokenAndClaims` from `@tazama-lf/auth-lib` which checks the token's `claims` array).

---

### 2. src/config-notify/config-notify.controller.ts — Endpoint re-gating

[↑ Back to top](#pr-review-dems-43--feat-approver-claims-added-for-activation)

```diff
-import { RequireDemsWriteRole } from '../auth/auth.decorator';
+import { RequireActivationRole } from '../auth/auth.decorator';

 @Patch(':id')
-@RequireDemsWriteRole()
+@RequireActivationRole()
```

The full controller after the change:

```typescript
@Controller('config-notify')
@UseGuards(TazamaAuthGuard)
export class ConfigNotifyController {
  @Patch(':id')
  @RequireActivationRole()
  @HttpCode(HttpStatus.OK)
  async updateCache(@Param('id', ParseIntPipe) id: number, @Body() body: UpdateCacheDto): Promise<{ message: string }> {
    await this.configNotifyService.updateCache(id, body.publishing_status);
    return { message: `Cache updated successfully for config ID: ${id}` };
  }
}
```

This is a **complete replacement** of the claim — `dems:write` is no longer accepted on this endpoint. Only tokens carrying `approver` will pass the guard. The import is cleaned up correctly; `RequireDemsWriteRole` is fully removed from this file.

The guard implementation (`TazamaAuthGuard`) uses **AND logic** — `requiredClaims.every((claim) => validated[claim])` — so there is no built-in OR support. A user with both `dems:write` and `approver` would pass, but a user with only `dems:write` would now be blocked.

---

### 3. tests/dems-engine/dems-engine.service.spec.ts — Test expansion

[↑ Back to top](#pr-review-dems-43--feat-approver-claims-added-for-activation)

**Mock extensions** (setup block):

```typescript
mockDatabaseOperationsService = {
  ...
  addDataModelTable: jest.fn(),
  addAccount: jest.fn(),
  addEntity: jest.fn(),
  addAccountHolder: jest.fn(),
  getTransaction: jest.fn(),
} as any;
```

Five new mock methods added. These are required by the new test cases and would have caused `TypeError: mockDatabaseOperationsService.addAccount is not a function` failures without them.

**New test suites:**

| Suite | Cases | What is covered |
|-------|-------|-----------------|
| `processMappings additional branches` | 2 | Null mapping (no mapping configured); dynamic mapping to non-redis/transactionDetails destination |
| `executeConfiguredFunctions branches` | 10 | Invalid function name; `saveTransactionDetails` success and failure; `addDataModelTable` with payload/dataModel datasource; `addDataModelTable` invalid columns and missing table name; `saveTransactionHistory` with `trackedFields`; `addAccount` success, failure, and no-mapping path |
| `saveTransactionDataAndNotify error types` | 2 | `TransactionOperationError` non-specific; generic NATS error from `notifyEventDirector` |
| `handleMessage pacs.002 branch` | 1 | pacs.002 payload takes the `enhancedRequest` path |
| `handleMessage related transaction branch` | 1 | `related_transaction` config triggers a secondary `getTransaction` lookup |
| `handleMessage AJV validation error message` | 1 | Invalid `$ref` in schema triggers AJV exception path |

The second commit (`186c2c3b`) contains all the test additions, cleanly separated from the auth change.

---

## Authorization Architecture Analysis

[↑ Back to top](#pr-review-dems-43--feat-approver-claims-added-for-activation)

### How the guard enforces claims

[↑ Back to top](#pr-review-dems-43--feat-approver-claims-added-for-activation)

`TazamaAuthGuard.canActivate` works as follows:

1. Reads `requiredClaims` from the route metadata (set by `@RequireActivationRole()`).
2. Validates the Bearer token via `validateTokenAndClaims` from `@tazama-lf/auth-lib`.
3. Checks `requiredClaims.every((claim) => validated[claim])` — **ALL claims must be present**.
4. Throws `UnauthorizedException` if any required claim is missing.

This means:
- `RequireActivationRole()` sets `requiredClaims = ['approver']`.
- A token with `['approver']` → **passes**.
- A token with `['dems:write']` → **blocked**.
- A token with `['dems:write', 'approver']` → **passes**.
- A token with no claims → **blocked**.

### Impact of the decorator swap

[↑ Back to top](#pr-review-dems-43--feat-approver-claims-added-for-activation)

The PR's stated goal is to give approvers access to `PATCH /config-notify/:id`. The swap achieves that. However it also **revokes access from `dems:write` holders** who previously had it. Whether this is intentional depends on the role model:

- If `dems:write` users should no longer be able to trigger cache activation — the swap is correct.
- If `dems:write` users and `approver` users should both have access — the current single-claim decorator cannot express OR logic. A `RequireClaims('dems:write', 'approver')` call would require BOTH, not EITHER.

The developer clarified to CodeRabbit that the `approver`-only gating is intentional for this endpoint. That context is accepted, but it is not documented in code — see Issue 1 below.

---

## Code Quality Analysis

[↑ Back to top](#pr-review-dems-43--feat-approver-claims-added-for-activation)

### Strengths

[↑ Back to top](#pr-review-dems-43--feat-approver-claims-added-for-activation)

- **Minimal and focused** — the auth change is 4 lines. No unnecessary refactoring.
- **Pattern consistency** — `RequireActivationRole` follows the exact same factory function pattern as `RequireDemsWriteRole`. Anyone reading the decorator file can immediately understand how it works.
- **Centralized claim registry** — adding `APPROVER` to the frozen `EventMonitoringClaims` object means the claim string is defined once and referenced by name throughout the codebase, avoiding magic strings.
- **Clean import management** — the unused `RequireDemsWriteRole` import is removed from the controller; no dead imports remain.
- **Well-structured test expansion** — the new tests are grouped into clearly named `describe` blocks, each targeting a specific branch or error path. The mock extension is tidy and contained.
- **PR checklist fully completed** — locally tested, dev environment tested, Husky run, and unit tests passing are all checked. Coverage screenshot is provided.

---

### Issues and Observations

[↑ Back to top](#pr-review-dems-43--feat-approver-claims-added-for-activation)

#### Issue 1 — No test for the authorization change itself

**Severity: Major (Test Coverage)**

There are no tests for `config-notify.controller.ts` in this PR or apparently in the repo. The entire auth change — the core of this PR — is untested. There is no test that:
- Verifies a request with `approver` claim is accepted
- Verifies a request with `dems:write` claim is now rejected
- Verifies a request with no claim is rejected

The 14 new tests all cover `dems-engine.service.spec.ts`, a completely different service. The stated coverage target of 95% likely refers to that file. The controller change itself is not covered at all.

A `config-notify.controller.spec.ts` (or similar) with at minimum a happy path and an unauthorized path would be required to give this auth change any test-level protection against regression.

#### Issue 2 — `approver` claim string is not documented or cross-referenced

**Severity: Minor (Maintainability)**

The claim string `'approver'` is now in the codebase but there is no documentation explaining:
- Where this claim originates (which identity provider issues it)
- What role in the system maps to this claim
- Why `PATCH /config-notify/:id` now requires it instead of `dems:write`

The developer explained this to CodeRabbit verbally in a review comment, but that context will be lost. At minimum, a short JSDoc comment on `RequireActivationRole` would preserve the intent:

```typescript
/**
 * Requires the 'approver' claim. Used on endpoints that change publishing status,
 * since only approvers are authorised to activate/deactivate configs.
 */
export const RequireActivationRole = (): ReturnType<typeof SetMetadata> =>
  RequireClaims(EventMonitoringClaims.APPROVER);
```

#### Issue 3 — `dems:write` users are silently locked out with no migration path

**Severity: Minor (Breaking Change Risk)**

Any integration, service account, or client that was previously calling `PATCH /config-notify/:id` with a `dems:write` token will start receiving `401 Unauthorized` after this merges. There is no deprecation notice, no changelog entry, and no mention of this in the PR description.

If this is a breaking change for any consumer, it should be explicitly called out in the PR description so downstream teams can update their token claims before this deploys.

#### Issue 4 — Related-transaction test does not assert the lookup key

**Severity: Minor (Test Weakness) — Flagged by CodeRabbit**

In the `handleMessage related transaction branch` test:

```typescript
expect(mockDatabaseOperationsService.getTransaction).toHaveBeenCalled();
```

This only asserts that `getTransaction` was invoked — not that it was called with the correct argument. The test is driven by `FIToFIPmtSts.TxInfAndSts.OrgnlEndToEndId: 'e2e-123'`, so the assertion should be:

```typescript
expect(mockDatabaseOperationsService.getTransaction).toHaveBeenCalledWith('e2e-123');
```

Without this, the test would pass even if the service passed a wrong or empty identifier to the database lookup.

#### Issue 5 — pacs.002 test does not assert the downstream payload

**Severity: Minor (Test Weakness) — Flagged by CodeRabbit**

The `handleMessage pacs.002 branch` test asserts only:

```typescript
expect(result).toHaveProperty('success', true);
```

The stated purpose of this test is to verify that `enhancedRequest` is used as `transactionForNats` for pacs.002 flows. That branch-specific behaviour is not verified. The test would pass even if the code sent the original payload rather than the enhanced one. The assertion should verify what was passed to `saveTransactionDataAndNotify` or `notifyEventDirector`.

---

## Security Assessment

[↑ Back to top](#pr-review-dems-43--feat-approver-claims-added-for-activation)

| Concern | Assessment |
|---------|-----------|
| Privilege escalation | No — `approver` replaces `dems:write`; this narrows or shifts access, it does not widen it. |
| Claim string case sensitivity | The guard compares claim strings from the JWT directly. `'approver'` must match exactly what the IdP issues. If the IdP issues `'Approver'` or `'APPROVER'`, the guard will silently deny. This should be confirmed against the IdP configuration. |
| AND-only guard logic | `TazamaAuthGuard` requires ALL listed claims. `RequireActivationRole` only lists one claim (`approver`), so there is no multi-claim confusion here. The risk would arise if someone added a second claim to `RequireActivationRole` in the future expecting OR behaviour. |
| Unauthorized users locked out of activation | This is the intended outcome — the endpoint now correctly restricts publishing status changes to approvers only. |
| JWT decoding without verification | `extractTokenPayload` uses `jwt.decode` (not `jwt.verify`). Signature validation is delegated to `validateTokenAndClaims` in `auth-lib`. If that library does not verify the signature, tokens could be forged. This is pre-existing and out of scope for this PR, but worth noting. |

No new security vulnerabilities introduced by this PR.

---

## Test Coverage

[↑ Back to top](#pr-review-dems-43--feat-approver-claims-added-for-activation)

The PR description confirms unit tests passing and includes a screenshot showing ≥95% coverage for `dems-engine.service.spec.ts`. The 14 new tests are meaningful additions that exercise real branches in the service. However:

- **The auth change itself has no test coverage** — this is the most critical gap.
- The related-transaction and pacs.002 tests have weak assertions (Issues 4 and 5 above).

For a security-sensitive change like an authorization decorator swap, controller-level tests are not optional — they are the primary protection against accidental regression.

---

## CodeRabbit Activity

[↑ Back to top](#pr-review-dems-43--feat-approver-claims-added-for-activation)

### Pass 1 — Decorator change review

[↑ Back to top](#pr-review-dems-43--feat-approver-claims-added-for-activation)

**Commit reviewed:** `1e18a124`  
**Finding:** CodeRabbit raised a **Major** security concern — the decorator swap removes `dems:write` access from the endpoint, potentially breaking existing callers.

**Developer response:** MuhammadAli-Paysys clarified that the `approver`-only gating is intentional.

**CodeRabbit resolution:** Withdrew the comment after the developer confirmed intent. CodeRabbit saved a learning note recording this decision.

| Finding | Severity | Status |
|---------|----------|--------|
| `PATCH /config-notify/:id` switches from `dems:write` to `approver`, blocking existing write-role callers | Major | ✅ Intentional — confirmed by developer and withdrawn by CodeRabbit |

---

### Pass 2 — Test expansion review

[↑ Back to top](#pr-review-dems-43--feat-approver-claims-added-for-activation)

**Commit reviewed:** `186c2c3b`  
**Findings:** 2 nitpick-level comments on test assertion strength.

| Finding | Severity | Status |
|---------|----------|--------|
| Related-transaction test only checks `getTransaction` was called, not the exact key used | Trivial | ❌ Not resolved |
| pacs.002 test only checks `success: true`, does not verify `enhancedRequest` is forwarded | Trivial | ❌ Not resolved |

CodeRabbit classified both as nitpicks. The developer did not respond to the second pass. MAdeel95 approved the PR on the same day without addressing these.

---

## Summary and Verdict

[↑ Back to top](#pr-review-dems-43--feat-approver-claims-added-for-activation)

**Verdict: Changes Requested**

The authorization change is intentional, well-reasoned, and correctly implemented. The pattern is consistent, imports are clean, and the PR description explains the business reason clearly. The test expansion is a real improvement to service coverage.

However the PR has one significant gap and two weaker tests that need attention:

### Blocking

1. **No tests for the controller authorization change** — the core of this PR (swapping `RequireDemsWriteRole` for `RequireActivationRole` on `PATCH /config-notify/:id`) is completely untested. A `config-notify.controller.spec.ts` must be added with at least: (a) a request with `approver` claim passes, (b) a request with `dems:write` claim is rejected with 401.

### Non-blocking but recommended

2. **Assert the exact argument to `getTransaction`** in the related-transaction test — currently the test only verifies the call happened, not what it was called with.

3. **Assert the downstream payload** in the pacs.002 branch test — the test does not verify that `enhancedRequest` was used instead of the original payload.

4. **Document the intentional `dems:write` lockout** — add a JSDoc comment to `RequireActivationRole` and/or a note in the PR description that `dems:write` callers of this endpoint will receive 401 after merge.

---

## GitHub Review Comment

[↑ Back to top](#pr-review-dems-43--feat-approver-claims-added-for-activation)

````markdown
**Changes Requested**

The auth change is clean and the motivation is well-explained — approvers couldn't activate configs because the endpoint required `dems:write`, which they don't hold. The `RequireActivationRole` decorator follows the existing pattern correctly and the import cleanup is tidy. The test expansion is a genuine improvement.

One thing needs to be fixed before this merges.

---

### Blocking

**1. Add a controller test for the auth change**

This PR's core change — swapping `@RequireDemsWriteRole()` for `@RequireActivationRole()` on `PATCH /config-notify/:id` — is currently untested. The 14 new tests all target `DemsEngineService`, not the controller. Without a controller test, nothing will catch a regression if this decorator is accidentally changed back or overridden.

Please add a `config-notify.controller.spec.ts` (or equivalent) with at minimum these two cases:

```typescript
it('should allow a request with the approver claim', async () => {
  // mock guard to pass with 'approver' claim — assert 200
});

it('should reject a request with only dems:write claim', async () => {
  // mock guard to fail without 'approver' claim — assert 401
});
```

The guard itself is already tested elsewhere — you can mock `TazamaAuthGuard` at the controller level and just verify the decorator metadata is wired up correctly.

---

### Non-blocking (please address if possible)

**2. Assert the exact lookup key in the related-transaction test**

In `handleMessage related transaction branch`, line 1027:
```typescript
expect(mockDatabaseOperationsService.getTransaction).toHaveBeenCalled();
```
This only checks that `getTransaction` was called — not that it was called with the right identifier. The payload has `FIToFIPmtSts.TxInfAndSts.OrgnlEndToEndId: 'e2e-123'`, so please change this to:
```typescript
expect(mockDatabaseOperationsService.getTransaction).toHaveBeenCalledWith('e2e-123');
```
Without this, the test would pass even if the service passed a wrong or empty identifier to the database.

**3. Assert the downstream payload in the pacs.002 branch test**

In `handleMessage pacs.002 branch`, the test only checks `expect(result).toHaveProperty('success', true)`. The stated purpose is to verify `enhancedRequest` is forwarded as `transactionForNats` — but a test that only checks for generic success would pass even if the original payload were used instead. Please add an assertion on what was passed to `saveTransactionDataAndNotify` or `notifyEventDirector`.

**4. Add a JSDoc comment to `RequireActivationRole`**

You explained to CodeRabbit why this endpoint is `approver`-only, but that context only lives in a review comment. Please put it in the code so future maintainers don't have to dig through PR history:

```typescript
/**
 * Requires the 'approver' claim. Applied to endpoints that change publishing
 * status — only approvers are authorised to activate/deactivate configs.
 * Note: dems:write callers do NOT have access to these endpoints.
 */
export const RequireActivationRole = (): ReturnType<typeof SetMetadata> =>
  RequireClaims(EventMonitoringClaims.APPROVER);
```
````
