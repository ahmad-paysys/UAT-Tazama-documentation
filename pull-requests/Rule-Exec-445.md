# PR Review: Rule-Exec #445 — fix: collapse unexpected rule config fetch error into concise reason (#444)

**Repo:** tazama-lf/rule-executer
**Branch:** `fix/444-reason-field` → `dev`
**Author:** Justus-at-Tazama (Justus Ortlepp)
**Date Reviewed:** 2026-07-03
**Label:** `bug`
**Size:** +34 / -1 lines across 2 files
**Commits:** 1 (`4e1ad5d27efdcd8bdeed2b344b0f91465dc27e04`)
**State:** OPEN
**Existing approvals:** None

---

## Table of Contents

- [Overview](#overview)
- [What Changed (Detailed)](#what-changed-detailed)
  - [1. src/controllers/execute.ts — Collapse unexpected config-fetch errors to a generic reason](#1-srccontrollersexecutets--collapse-unexpected-config-fetch-errors-to-a-generic-reason)
  - [2. __tests__/unit/logic.service.test.ts — Regression test for ZodError leakage](#2-__tests__unitlogicservicetestts--regression-test-for-zoderror-leakage)
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

This PR fixes issue #444, where a failure in `getRuleConfig` — particularly a Zod schema validation error — caused the full, multi-line `ZodError` JSON blob to be written into `ruleResult.reason` and sent on the wire to downstream consumers. The error message could span dozens of lines and expose internal schema structure, making it noisy and potentially revealing implementation details.

The fix is applied in the `catch` block of `execute.ts` inside the `execute` function. Two pre-existing internal guard reasons (`Rule not found in network map` and `Rule processor configuration not retrievable`) are surfaced as-is because they are already concise and meaningful. Any other error is collapsed to the generic label `Error while getting rule configuration`. Full error detail is still captured in `loggerService.error` so diagnostics are not lost.

A single regression test is added confirming that a ZodError-shaped message is collapsed and does not appear on the wire.

| File | Nature of Change |
|------|-----------------|
| `src/controllers/execute.ts` | Replace raw `error.message` with an allowlist-based guard in the config-fetch catch block |
| `__tests__/unit/logic.service.test.ts` | Add regression test asserting ZodError is not leaked in `ruleResult.reason` |

[↑ Back to top](#pr-review-rule-exec-445--fix-collapse-unexpected-rule-config-fetch-error-into-concise-reason-444)

---

## What Changed (Detailed)

### 1. `src/controllers/execute.ts` — Collapse unexpected config-fetch errors to a generic reason

**Before:**
```typescript
ruleRes = {
  ...ruleRes,
  subRuleRef: '.err',
  reason: (error as Error).message,
};
```

**After:**
```typescript
// The full error detail is retained in the log above. On the wire we surface the concise,
// known guard reasons as-is, but collapse any other failure (e.g. a config validation/DB
// error such as a ZodError) to a generic reason rather than leaking the full message.
const knownGuardReasons = ['Rule not found in network map', 'Rule processor configuration not retrievable'];
const rawReason = (error as Error).message;
ruleRes = {
  ...ruleRes,
  subRuleRef: '.err',
  reason: knownGuardReasons.includes(rawReason) ? rawReason : 'Error while getting rule configuration',
};
```

The change introduces an inline allowlist (`knownGuardReasons`) and does a string-equality check against it. Only the two known internal guard strings pass through; everything else is replaced with a fixed label. The `loggerService.error` call that precedes this block already captures `util.inspect(error)`, so the full error payload remains available in logs.

**Risk note:** The allowlist is a plain `string[]` literal defined inline inside the catch block. If a third concise guard reason is added in the future (e.g. a new early-return path higher up that throws a specific string), the developer must remember to update this list — there is no type-system enforcement. This is a minor maintainability concern but not a blocker for a two-element list at this scope.

---

### 2. `__tests__/unit/logic.service.test.ts` — Regression test for ZodError leakage

A new `it` block is added after the existing tests for `getRuleConfig` failure paths:

```typescript
it('should collapse an unexpected getRuleConfig error (e.g. ZodError) to a concise reason ', async () => {
  const zodLikeError = new Error(
    'ZodError: [\n  {\n    "expected": "number",\n    "code": "invalid_type",\n    "path": [ "config", "parameters", "maxQueryRange" ],\n    "message": "Invalid input: expected number, received undefined"\n  }\n]',
  );

  jest.spyOn(databaseManager, 'getRuleConfig').mockImplementationOnce(async (): Promise<RuleConfig> => {
    throw zodLikeError;
  });

  const expectedReq = getMockRequest();

  responseSpy = jest.spyOn(server, 'handleResponse').mockImplementationOnce(jest.fn());

  await execute(expectedReq);

  expect(responseSpy).toHaveBeenCalledWith(
    expect.objectContaining({
      ruleResult: expect.objectContaining({
        subRuleRef: '.err',
        reason: 'Error while getting rule configuration',
      }),
    }),
  );
  // The verbose underlying message must not leak onto the wire.
  const call = responseSpy.mock.calls[0][0] as { ruleResult: RuleResult };
  expect(call.ruleResult.reason).not.toContain('ZodError');
});
```

The test mocks `getRuleConfig` to throw an `Error` whose `.message` is a realistic ZodError payload, then asserts both that the collapsed reason is set and that the raw `ZodError` string does not appear. The double assertion (positive + negative) is well-structured: the positive check guards the happy path of the fix; the negative check guards against the original regression.

**Observation:** The test description has a trailing space before the closing `'` (`'...concise reason '`). This does not affect behaviour but is a minor cosmetic issue.

[↑ Back to top](#pr-review-rule-exec-445--fix-collapse-unexpected-rule-config-fetch-error-into-concise-reason-444)

---

## Code Quality Analysis

### Strengths

- **Minimal scope.** The PR touches exactly two files and one catch block. It does not refactor surrounding logic or change unrelated behaviour.
- **Diagnostic preservation.** The decision to keep `util.inspect(error)` in `loggerService.error` before applying the generic label means operators lose no debugging information.
- **Self-documenting comment.** The inline comment in `execute.ts` explains the intent (keep logs verbose, keep wire output concise) clearly and concisely without over-explaining.
- **Double-sided test assertion.** The regression test checks both that the correct collapsed value is present and that the raw ZodError string is absent — a thorough guard against the specific regression.
- **Commit message quality.** The commit message is detailed, explains the before/after, references the issue, and is signed off.

### Issues and Observations

#### Issue 1 — Inline allowlist is not co-located with the strings it guards against

**Severity: Minor (Maintainability)**

The two entries in `knownGuardReasons` are string literals that must match error messages thrown earlier in the same function or in `databaseManager`. If either guard message changes, the allowlist silently stops working — the reason field will collapse to the generic label for what should be a known, surfaced reason.

```typescript
// Current
const knownGuardReasons = ['Rule not found in network map', 'Rule processor configuration not retrievable'];
```

A safer pattern is to define the guard strings as named constants or an enum where they are thrown:

```typescript
// In a shared constants file (or top of execute.ts)
export const RULE_NOT_FOUND_REASON = 'Rule not found in network map';
export const CONFIG_NOT_RETRIEVABLE_REASON = 'Rule processor configuration not retrievable';

// In the catch block
const knownGuardReasons = [RULE_NOT_FOUND_REASON, CONFIG_NOT_RETRIEVABLE_REASON];
```

This refactor is beyond the strict scope of this PR, so treat this as an observation rather than a blocking request.

#### Issue 2 — Test description trailing whitespace

**Severity: Informational (Code Quality)**

```typescript
it('should collapse an unexpected getRuleConfig error (e.g. ZodError) to a concise reason ', async () => {
```

There is a trailing space before the closing quote. Cosmetic only; does not affect test execution or CI.

[↑ Back to top](#pr-review-rule-exec-445--fix-collapse-unexpected-rule-config-fetch-error-into-concise-reason-444)

---

## Security Assessment

| Concern | Assessment |
|---------|-----------|
| Error detail leakage on the wire | **Fixed by this PR.** Previously, a ZodError's full multi-line JSON payload (including internal field paths and type expectations) was written into `ruleResult.reason`. This PR collapses all non-guard errors to a generic label, eliminating information leakage to downstream consumers. |
| Log retention of full error | The full error is still written to `loggerService.error` via `util.inspect(error)`. This is correct — logs are internal; the wire payload is external. No concern here. |
| Allowlist bypass | The allowlist is checked with strict string equality (`Array.includes`). There is no injection risk; the values are constants compared against an already-thrown `Error.message`. |
| Other attack surfaces | No new network inputs, database queries, HTML rendering, or auth changes introduced. |

No new security vulnerabilities introduced by this PR. The PR actively reduces an information-disclosure risk.

[↑ Back to top](#pr-review-rule-exec-445--fix-collapse-unexpected-rule-config-fetch-error-into-concise-reason-444)

---

## Test Coverage

- **What is tested:** A ZodError-like `Error` thrown from `getRuleConfig` is correctly collapsed to `'Error while getting rule configuration'` and the verbose ZodError string does not appear in `ruleResult.reason`.
- **What is not tested:**
  - The two allowlisted guard reasons (`Rule not found in network map`, `Rule processor configuration not retrievable`) are reportedly covered by pre-existing tests (the PR description states "The two existing tests that assert the concise guard reasons remain green"), but no new explicit tests are added for the pass-through path — this is acceptable given pre-existing coverage.
  - Non-`Error` throwables (e.g., a plain string or object thrown from `getRuleConfig`) are not tested. In those cases `(error as Error).message` would be `undefined`, which would not be in the allowlist and would correctly collapse to the generic reason — but this path is untested.
- **PR checklist:** The description states: build clean, eslint clean, 12 unit tests passed. No coverage screenshot is attached, but the test suite result is stated explicitly.
- **CI evidence:** Not visible in the PR at time of review (no CI status checks shown in the comments).

[↑ Back to top](#pr-review-rule-exec-445--fix-collapse-unexpected-rule-config-fetch-error-into-concise-reason-444)

---

## CodeRabbit Activity

### Pass 1 — Rate-limited, no review produced

**Commit reviewed:** `4e1ad5d27efdcd8bdeed2b344b0f91465dc27e04`
**Findings:** 0 actionable comments (review was rate-limited and did not complete)

CodeRabbit was triggered on this PR but hit a per-developer rate limit. It identified the two files for processing (`__tests__/unit/logic.service.test.ts`, `src/controllers/execute.ts`) but produced no findings. No CodeRabbit review content is available for this PR.

[↑ Back to top](#pr-review-rule-exec-445--fix-collapse-unexpected-rule-config-fetch-error-into-concise-reason-444)

---

## Summary and Verdict

**Verdict: Approved**

This is a clean, tightly scoped bug fix. The root cause (raw `error.message` being placed on the wire) is correctly identified and fixed with a simple allowlist guard. Diagnostic information is preserved in logs. The regression test is well-structured with both positive and negative assertions. The PR description and commit message are thorough.

The only observations are a minor maintainability note about the inline allowlist not being co-located with the strings it mirrors, and a cosmetic trailing space in a test description. Neither is blocking.

### Blocking

None.

### Non-blocking but recommended

1. **Inline allowlist string literals** — Consider extracting the two guard reason strings as named constants in a shared location so that a future change to either guard message is caught by reference rather than discovered at runtime. This is an observation for a follow-on cleanup, not a requirement for this PR.

[↑ Back to top](#pr-review-rule-exec-445--fix-collapse-unexpected-rule-config-fetch-error-into-concise-reason-444)

---

## GitHub Review Comment

````markdown
```markdown
**Approved**

Clean, tightly scoped fix. The root cause — raw `error.message` (which for a Zod validation failure is a multi-line JSON blob) being written into `ruleResult.reason` and sent on the wire — is correctly identified and resolved. Diagnostic detail is preserved in the log. The regression test is well-structured with both positive and negative assertions.

---

### Blocking

None.

---

### Non-blocking (please address in this PR if possible)

**1. Inline allowlist strings are not co-located with the messages they guard**

The two entries in `knownGuardReasons` are string literals that must exactly match error messages thrown earlier in the same function or in `databaseManager`. If either guard message ever changes, the allowlist silently stops working — that reason will fall through to the generic label instead of being surfaced as intended.

`src/controllers/execute.ts`:
```typescript
const knownGuardReasons = ['Rule not found in network map', 'Rule processor configuration not retrievable'];
```

Consider extracting these as named constants (exported from a shared file or defined at the top of `execute.ts`) so that a future change to a guard message is caught by reference:

```typescript
// e.g. at the top of execute.ts or in a constants file
const RULE_NOT_FOUND_REASON = 'Rule not found in network map';
const CONFIG_NOT_RETRIEVABLE_REASON = 'Rule processor configuration not retrievable';

// In the catch block
const knownGuardReasons = [RULE_NOT_FOUND_REASON, CONFIG_NOT_RETRIEVABLE_REASON];
```

This is a follow-on cleanup suggestion, not a blocker.
```
````

[↑ Back to top](#pr-review-rule-exec-445--fix-collapse-unexpected-rule-config-fetch-error-into-concise-reason-444)
