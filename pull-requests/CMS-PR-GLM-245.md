# PR Review: CMS #245 — fix: added the efrup flow processor in alert detail modal

**Repo:** tazama-lf/case-management-system
**Branch:** `paysys/SLA-task` → `dev`
**Author:** ibadkhan088 (Ibad Ahmed Khan)
**Date Reviewed:** 2026-07-16
**Label:** bug
**Size:** +91 / -4 lines across 2 files
**Commits:** 1 (`beeb85411b6ba1f70bc05e6a1acf278aefe9953c`)
**State:** OPEN
**Existing approvals:** ahmad-paysys — CHANGES_REQUESTED (2026-07-15)

## Table of Contents

- [Overview](#overview)
- [What Changed (Detailed)](#what-changed-detailed)
  - [1. AlertsDetailModal.tsx — EFRuP flow processor fetch and display](#1-alertsdetailmodaltsx--efrup-flow-processor-fetch-and-display)
  - [2. AlertsDetailModal.test.tsx — Tests for EFRuP rendering](#2-alertsdetailmodaltesttsx--tests-for-efrup-rendering)
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

This PR adds an EFRuP (flow processor) badge to the Alert Detail Modal in the Case Management System frontend. When a user opens an alert, the modal now fetches alert-navigator data via the existing `alertNavigatorService.getAlertNavigator()` call (scoped to the logged-in user's tenant), extracts the first typology's `flowProcessorData` field, and displays it as a red "EFRuP: {value}" label next to the "Triggered Typologies" heading. If no flow processor data is found, or the fetch fails, or the tenant/user is missing, the badge is hidden.

The PR touches two files:
- `AlertsDetailModal.tsx` — new state, `useEffect` fetch, and conditional render
- `AlertsDetailModal.test.tsx` — new mocks and two test cases (EFRuP present, EFRuP absent)

Targets `dev` — correct for this repo's flow. All 15 CI checks pass (CodeQL, DCO, dependency-review, encoding-check, hadolint, Node.js CI build/style/tests, conventional-commits, GPG verify, njsscan, CodeRabbit). No failing checks.

The PR is linked to issue #188 ("Typologies Displayed Improperly in CMS Alert Details Modal"). The `mergeStateStatus` is `BLOCKED` because reviewer `ahmad-paysys` requested changes on 2026-07-15. The author responded with an explanation (not a code change) addressing the cancellation-guard concern, and CodeRabbit withdrew its finding. Two non-blocking items from the review remain unaddressed.

| File | Nature of Change |
|------|-----------------|
| `frontend/src/features/alerts/components/AlertsDetailModal.tsx` | Added `flowProcessorData` state, `useEffect` to fetch alert-navigator data, and conditional EFRuP badge in the Triggered Typologies section |
| `frontend/src/features/alerts/components/__tests__/AlertsDetailModal.test.tsx` | Added mocks for `authService.getUser` and `alertNavigatorService.getAlertNavigator`, plus two tests for EFRuP rendering |

[↑ Back to top](#pr-review-cms-245--fix-added-the-efrup-flow-processor-in-alert-detail-modal)

---

## What Changed (Detailed)

### 1. AlertsDetailModal.tsx — EFRuP flow processor fetch and display

**Import added (line 18):**

```diff
+ import alertNavigatorService from '../../cases/components/view/visualizations/alertnavigator/services';
```

This imports the existing `AlertNavigatorService` singleton, which calls `/api/v1/lakehouse/alert-navigator/{alertId}?tenantId={tenantId}`. The service parses JSON-string rules into arrays and returns an `AlertNavigatorDto` with a `typologies` array where each `TypologyDto` has an optional `flowProcessorData?: string` field.

**New state (lines 260–262):**

```diff
+ const [flowProcessorData, setFlowProcessorData] = useState<
+   string | undefined
+ >(undefined);
```

A simple string state to hold the EFRuP value. `undefined` means "not loaded / not available" → badge hidden.

**New useEffect (lines 356–385):**

```tsx
useEffect(() => {
  const fetchFlowProcessorData = async () => {
    // The lakehouse alert-navigator lookup is scoped to the logged-in user's tenant
    // (the same convention VisualizationsTab uses for AlertNavigatorTab), not the
    // alert's own tenant_id field.
    const tenantId = authService.getUser()?.tenantId;
    if (!alert?.alert_id || !tenantId) {
      setFlowProcessorData(undefined);
      return;
    }

    try {
      const navigatorData = await alertNavigatorService.getAlertNavigator(
        alert.alert_id,
        tenantId,
      );
      setFlowProcessorData(
        navigatorData.typologies?.find(
          (typology) => typology.flowProcessorData,
        )?.flowProcessorData,
      );
    } catch (err) {
      console.error('Failed to fetch alert navigator flow processor data:', err);
      setFlowProcessorData(undefined);
    }
  };

  fetchFlowProcessorData();
}, [alert?.alert_id]);
```

This effect runs whenever `alert?.alert_id` changes. It:
1. Gets the logged-in user's `tenantId` from `authService.getUser()`.
2. Bails out (setting `flowProcessorData` to `undefined`) if there's no alert ID or no tenant ID.
3. Calls `alertNavigatorService.getAlertNavigator(alertId, tenantId)`.
4. Finds the first typology with a truthy `flowProcessorData` and stores its value.
5. On error, logs to `console.error` and clears the state.

**Render change (lines 869–882):**

```diff
- <h4 className="text-sm font-semibold text-gray-900 mb-4">
-   Triggered Typologies
- </h4>
-
+ <div className="mb-4 grid grid-cols-3 items-center">
+   <h4 className="text-sm font-semibold text-gray-900">
+     Triggered Typologies
+   </h4>
+
+   {flowProcessorData && (
+     <div className="justify-self-center text-sm font-semibold">
+       <span className="text-gray-900">EFRuP:</span>{' '}
+       <span className="text-red-600">{flowProcessorData}</span>
+     </div>
+   )}
+ </div>
```

The "Triggered Typologies" heading is now wrapped in a 3-column CSS grid. The EFRuP badge is placed in column 2 (centered via `justify-self-center`) when `flowProcessorData` is truthy. Column 3 is empty. The EFRuP value is rendered in red (`text-red-600`).

[↑ Back to top](#pr-review-cms-245--fix-added-the-efrup-flow-processor-in-alert-detail-modal)

---

### 2. AlertsDetailModal.test.tsx — Tests for EFRuP rendering

**New import (line 14):**

```diff
+ import alertNavigatorService from "../../../cases/components/view/visualizations/alertnavigator/services";
```

**New mock (lines 73–79):**

```diff
+ vi.mock(
+   "../../../cases/components/view/visualizations/alertnavigator/services",
+   () => ({
+     default: {
+       getAlertNavigator: vi.fn(),
+     },
+   }),
+ );
```

**Updated `authService` mock (line 68):**

```diff
  vi.mock("@/features/auth/services/authService", () => ({
    default: {
      hasCMSComplianceOfficerRole: vi.fn(),
+     getUser: vi.fn(),
    },
  }));
```

**New mock references (lines 158–159):**

```diff
+ const mockGetUser = authService.getUser as Mock;
+ const mockGetAlertNavigator = alertNavigatorService.getAlertNavigator as Mock;
```

**New `beforeEach` setup (lines 183–184):**

```diff
+ mockGetUser.mockReturnValue({ userId: "user-1", tenantId: "DEFAULT" });
+ mockGetAlertNavigator.mockResolvedValue({ typologies: [] });
```

**Test 1 — EFRuP displayed with data (lines 409–425):**

```tsx
it("displays the EFRuP flow processor value from the alert navigator lookup", async () => {
  mockGetAlertNavigator.mockResolvedValue({
    typologies: [
      { typologyId: "typology-1", typologyCfg: "Money Laundering", flowProcessorData: "Block" },
    ],
  });

  renderModal();

  await waitFor(() => {
    expect(mockGetAlertNavigator).toHaveBeenCalledWith(
      baseAlert.alert_id,
      "DEFAULT",
    );
    expect(screen.getByText("EFRuP:")).toBeInTheDocument();
    expect(screen.getByText("Block")).toBeInTheDocument();
  });
});
```

Verifies that when the navigator returns a typology with `flowProcessorData: "Block"`, the EFRuP label and value are rendered, and the service is called with the correct alert ID and tenant ID.

**Test 2 — EFRuP hidden when no data (lines 428–437):**

```tsx
it("does not display an EFRuP label when the alert navigator has no flow processor data", async () => {
  mockGetAlertNavigator.mockResolvedValue({ typologies: [] });

  renderModal();

  await waitFor(() => {
    expect(screen.getByText("Triggered Typologies")).toBeInTheDocument();
  });
  expect(screen.queryByText("EFRuP:")).not.toBeInTheDocument();
});
```

Verifies that when the navigator returns empty typologies, the EFRuP label is not rendered.

[↑ Back to top](#pr-review-cms-245--fix-added-the-efrup-flow-processor-in-alert-detail-modal)

---

## Code Quality Analysis

### Strengths

- **Reuses existing service.** The PR correctly reuses `alertNavigatorService` (the same singleton used by `AlertNavigatorTab`) rather than creating a new API client or duplicating the fetch logic. This is the right abstraction level.
- **Consistent tenant scoping.** The comment on lines 357–359 correctly documents that the tenant comes from the logged-in user (`authService.getUser()?.tenantId`), not from the alert's `tenant_id` field. This matches the convention used elsewhere in the codebase.
- **Clean error handling.** The `catch` block logs the error and clears `flowProcessorData` to `undefined`, ensuring the badge is hidden on failure rather than showing stale or undefined data.
- **Minimal scope.** The change is small and focused — 46 additions in the component, 45 in tests. No unrelated refactoring or drive-by changes.
- **Good test structure.** The two new tests follow the existing patterns in the file (mock setup in `beforeEach`, `waitFor` for async assertions, `renderModal()` helper). The mocks are properly registered and cleared.
- **Conditional rendering done correctly.** `{flowProcessorData && (...)}` ensures the badge only appears when there's a truthy value. Empty string, `undefined`, and `null` all correctly hide the badge.

### Issues and Observations

#### Issue 1 — EFRuP badge placement uses 3-column grid with empty third column

**Severity: Minor (Code Quality)**

The layout uses `grid grid-cols-3 items-center` with the EFRuP badge in column 2 (`justify-self-center`). Column 3 is always empty. This means the badge appears visually centered in the row, which may or may not match the design intent. If the intent was right-aligned, `flex justify-between` would be more natural:

```tsx
<div className="mb-4 flex items-center justify-between">
  <h4 className="text-sm font-semibold text-gray-900">Triggered Typologies</h4>
  {flowProcessorData && (
    <div className="text-sm font-semibold">
      <span className="text-gray-900">EFRuP:</span>{' '}
      <span className="text-red-600">{flowProcessorData}</span>
    </div>
  )}
</div>
```

If centered is deliberate, this can be ignored. Reviewer `ahmad-paysys` flagged this as non-blocking. No author response was found addressing whether the centered placement is deliberate.

#### Issue 2 — Missing test coverage for error and missing-tenant branches

**Severity: Minor (Test Coverage)**

The two new tests cover the happy path (EFRuP data present) and the empty-typologies path (EFRuP hidden). However, the following branches in the new `useEffect` are not tested:

1. **Error branch** — `mockGetAlertNavigator.mockRejectedValue(new Error(...))` → badge should be hidden, `console.error` called, no throw.
2. **Missing tenant branch** — `mockGetUser.mockReturnValue(undefined)` → badge hidden, service not called.

These are non-blocking but would close the loop on all code paths in the new `useEffect`. No author response was found addressing this recommendation.

#### Issue 3 — No cancellation guard in useEffect (declined by author)

**Severity: Informational (Code Quality)**

The `useEffect` does not use a cancellation flag. If the component unmounts while the `getAlertNavigator` promise is in flight, `setFlowProcessorData` will be called on an unmounted component. React 18 no longer warns about this.

The author responded to this concern (raised by both `ahmad-paysys` and CodeRabbit) with a valid explanation: the modal is instantiated for a single alert at a time, `alert?.alert_id` does not change while the modal is open, and other effects in the same file (`checkCompleteNewCaseStatus`, `checkCaseAccess`) also lack cancellation guards. CodeRabbit reviewed the explanation and withdrew its finding. This is consistent with existing codebase patterns and does not introduce a regression.

[↑ Back to top](#pr-review-cms-245--fix-added-the-efrup-flow-processor-in-alert-detail-modal)

---

## Security Assessment

| Concern | Assessment |
|---------|-----------|
| XSS / injection | `flowProcessorData` is a string from the API response rendered as text content in a `<span>`. React escapes text content by default — no `dangerouslySetInnerHTML` is used. No XSS risk. |
| Auth bypass | The fetch uses `authService.getUser()?.tenantId` for tenant scoping. If the user is not authenticated or has no tenant, the effect bails early and no API call is made. No auth bypass introduced. |
| Data exposure | The EFRuP value is displayed in the same modal that already shows alert details, typologies, and action history. No new data exposure surface — the user already has access to the alert. |
| Request security | The `alertNavigatorService.getAlertNavigator()` call uses the existing `apiClient` which handles auth headers. The `alertId` and `tenantId` are passed as URL path/query params. `alertId` is a number from the alert object (not user input), and `tenantId` comes from the authenticated session. No injection risk. |
| Error handling | Errors are caught and logged to `console.error`. No sensitive data is logged — only the error object. State is cleared on failure. |

No new security vulnerabilities introduced by this PR.

[↑ Back to top](#pr-review-cms-245--fix-added-the-efrup-flow-processor-in-alert-detail-modal)

---

## Test Coverage

**What is tested:**
- EFRuP badge renders with correct value when `alertNavigatorService.getAlertNavigator` returns a typology with `flowProcessorData` — verified that the service is called with the correct `alertId` and `tenantId` arguments.
- EFRuP badge is hidden when the navigator returns empty typologies.
- Mocks for `authService.getUser` and `alertNavigatorService.getAlertNavigator` are properly set up in `beforeEach` with sensible defaults.

**What is not tested:**
- **Error branch** — no test for the case where `getAlertNavigator` rejects. The `catch` block (`console.error` + state clear) is untested.
- **Missing tenant branch** — no test for the case where `authService.getUser()` returns `undefined` or an object without `tenantId`. The early-return guard is untested.
- **Multiple typologies** — no test verifying that `find()` picks the *first* typology with `flowProcessorData` when multiple typologies have the field.

**PR checklist:**
- [x] Locally
- [ ] Development Environment
- [ ] Not needed, changes very basic
- [x] Husky successfully run
- [x] Unit tests passing and Documentation done

The core functional change (fetch + display) has tests for the two main render branches. The error and edge-case branches are missing but non-blocking.

[↑ Back to top](#pr-review-cms-245--fix-added-the-efrup-flow-processor-in-alert-detail-modal)

---

## CodeRabbit Activity

### Pass 1 — Initial walkthrough

**Commit reviewed:** `f27d887b440758ed119a9afba687d84c5e866239`
**Findings:** 0 actionable comments (walkthrough only)

CodeRabbit provided a walkthrough summary and pre-merge checks (all 5 passed). No actionable findings.

### Pass 2 — Review on intermediate commit

**Commit reviewed:** `2da50771fe76ee2297c6bc5e7b6687af10c02fc5` (force-pushed, no longer exists on origin)
**Findings:** 1 nitpick comment

| Finding | Severity | Status |
|---------|----------|--------|
| Test `consoleErrorSpy` should use `try...finally` to prevent spy leakage; assert `console.error` was called in `waitFor` to sync with catch handler | Minor (nitpick) | ⚠️ Preemptive — the error test it references does not exist in the current code |

**Note:** This commit was force-pushed and no longer exists on origin. Finding reconstructed from GitHub review API.

### Pass 3 — Review on final HEAD

**Commit reviewed:** `beeb85411b6ba1f70bc05e6a1acf278aefe9953c`
**Findings:** 1 actionable comment

| Finding | Severity | Status |
|---------|----------|--------|
| Add cancellation flag (`let ignore = false`) in `useEffect` cleanup to prevent race conditions when `alert?.alert_id` changes or component unmounts | Minor (Stability) | ➖ Declined by author — modal is single-alert per instance, `alert_id` stable for component lifetime, consistent with other effects in the file. CodeRabbit reviewed explanation and withdrew finding. |

**Author response** ([issue comment #4982264035](https://github.com/tazama-lf/case-management-system/pull/245#issuecomment-4982264035)):

> Clearing flowProcessorData at the start of the fetch effect is not applicable in the current implementation. The alert detail modal is instantiated for a single alert at a time, and alert?.alert_id does not change while the modal remains open. A new alert cannot be loaded concurrently into the same component instance, so there is no practical race where an outdated request overwrites the state. Therefore, the additional cancellation flag would add complexity without addressing a real execution path.

**CodeRabbit reply** (inline, with `<review_comment_withdrawn>`):

> `@ibadkhan088`, that makes sense — since the modal is scoped to a single alert per instance and `alert.alert_id` remains stable for the component's lifetime, the concurrent-request scenario I flagged doesn't apply here. I also note this is consistent with the other effects in this file (e.g., `checkCompleteNewCaseStatus`, `checkCaseAccess`) which don't use cancellation guards either, so there's no inconsistency introduced. Thanks for the clarification.

[↑ Back to top](#pr-review-cms-245--fix-added-the-efrup-flow-processor-in-alert-detail-modal)

---

## Summary and Verdict

**Verdict: Approve with minor cleanup requested**

This is a small, well-scoped PR that correctly reuses the existing `alertNavigatorService` to display EFRuP flow processor data in the alert detail modal. The implementation is clean — proper tenant scoping, error handling, and conditional rendering. All CI checks pass. The two new tests cover the primary render branches.

Reviewer `ahmad-paysys` requested changes with three non-blocking items. The author addressed the cancellation-guard concern with a valid explanation that CodeRabbit concurred with and withdrew. The remaining two items (badge layout confirmation, additional test coverage for error/missing-tenant branches) are minor polish items that have not been addressed but do not block merge.

### Blocking

None.

### Non-blocking but recommended

1. **Confirm EFRuP badge placement** — The 3-column grid centers the badge in the row with an empty third column. If right-aligned was intended, switch to `flex justify-between`.
2. **Add error and missing-tenant test cases** — A test for `mockGetAlertNavigator.mockRejectedValue(...)` and one for `mockGetUser.mockReturnValue(undefined)` would close the loop on all code paths.

[↑ Back to top](#pr-review-cms-245--fix-added-the-efrup-flow-processor-in-alert-detail-modal)

---

## GitHub Review Comment

````markdown
**Approve with minor cleanup requested**

Small, well-scoped change that reuses the existing `alertNavigatorService` and correctly hides the EFRuP badge when there's no data. All CI green, no security concerns, both render branches covered by new tests. The cancellation-guard concern was resolved by the author's explanation — the modal is single-alert per instance, consistent with other effects in the file.

---

### Non-blocking (please address in this PR if possible)

**1. Confirm the EFRuP badge placement matches the design**

`grid grid-cols-3 items-center` + `justify-self-center` places the badge in the visual middle of the row (column 2 of 3, with column 3 empty). If the intent was right-aligned, `flex justify-between` reads more naturally:

```tsx
<div className="mb-4 flex items-center justify-between">
  <h4 className="text-sm font-semibold text-gray-900">Triggered Typologies</h4>
  {flowProcessorData && (
    <div className="text-sm font-semibold">
      <span className="text-gray-900">EFRuP:</span>{' '}
      <span className="text-red-600">{flowProcessorData}</span>
    </div>
  )}
</div>
```

If centred is deliberate, ignore this.

**2. Consider covering the error and missing-tenant branches in tests**

The new tests cover populated and empty typologies. A short case for `mockGetAlertNavigator.mockRejectedValue(...)` (badge hidden, no throw) and one for `mockGetUser.mockReturnValue(undefined)` (badge hidden, service not called) would close the loop.
````

[↑ Back to top](#pr-review-cms-245--fix-added-the-efrup-flow-processor-in-alert-detail-modal)

---
