# PR Review: CMS #245 — fix: added the efrup flow processor in alert detail modal

**Repo:** tazama-lf/case-management-system
**Branch:** `paysys/SLA-task` → `dev`
**Author:** ibadkhan088 (Ibad Ahmed Khan)
**Date Reviewed:** 2026-07-15
**Label:** —
**Size:** +91 / -4 lines across 2 files
**Commits:** 1 (`f27d887b`)
**State:** OPEN (mergeStateStatus: `BLOCKED` — awaiting required approvals; all CI green)
**Existing approvals:** none

## Table of Contents

- [Overview](#overview)
- [What Changed (Detailed)](#what-changed-detailed)
  - [1. AlertsDetailModal.tsx — fetch and render EFRuP flow-processor value](#1-alertsdetailmodaltsx--fetch-and-render-efrup-flow-processor-value)
  - [2. AlertsDetailModal.test.tsx — tests for EFRuP rendering / hiding](#2-alertsdetailmodaltesttsx--tests-for-efrup-rendering--hiding)
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

This PR adds an **EFRuP** (flow-processor) indicator to the "Triggered Typologies" panel of the Alerts Detail Modal. On modal open, it calls the existing lakehouse alert-navigator endpoint (same one that powers the `AlertNavigatorTab`), scans the returned typologies for the first one carrying `flowProcessorData`, and renders that value next to the "Triggered Typologies" header in red. When no navigator data or no populated `flowProcessorData` is present, the label is hidden entirely.

Targets `dev` — correct for this repo's flow. All CI checks pass (CodeQL, DCO, dependency-review, encoding-check, Hadolint, Node.js CI build/style/tests, njsscan, GPG verify, conventional-commit title, CodeRabbit). `mergeStateStatus` is `BLOCKED` only because required reviewer approvals have not been recorded yet — no failing checks. The PR branch is behind `dev` by ~60 files / 5.8k additions since its merge base, but a dry-run three-way merge shows no textual conflicts, and the types/services this PR imports from (`alertnavigator/services`, `alertnavigator/types`) already expose the `flowProcessorData?: string` field on `TypologyDto` on current `dev`.

| File | Nature of Change |
|------|-----------------|
| `frontend/src/features/alerts/components/AlertsDetailModal.tsx` | Adds `flowProcessorData` state, a `useEffect` that fetches alert-navigator data using the logged-in user's `tenantId`, and conditional EFRuP rendering in the Triggered Typologies header. |
| `frontend/src/features/alerts/components/__tests__/AlertsDetailModal.test.tsx` | Mocks `authService.getUser` and `alertNavigatorService.getAlertNavigator`; adds two tests covering the populated-data and empty-typologies branches. |

[↑ Back to top](#pr-review-cms-245--fix-added-the-efrup-flow-processor-in-alert-detail-modal)

---

## What Changed (Detailed)

### 1. AlertsDetailModal.tsx — fetch and render EFRuP flow-processor value

**New import and state** ([AlertsDetailModal.tsx:18](repos/case-management-system/frontend/src/features/alerts/components/AlertsDetailModal.tsx#L18), [:260](repos/case-management-system/frontend/src/features/alerts/components/AlertsDetailModal.tsx#L260)):

```tsx
import alertNavigatorService from '../../cases/components/view/visualizations/alertnavigator/services';
// ...
const [flowProcessorData, setFlowProcessorData] = useState<
  string | undefined
>(undefined);
```

**Fetch effect** ([AlertsDetailModal.tsx:356-384](repos/case-management-system/frontend/src/features/alerts/components/AlertsDetailModal.tsx#L356-L384)):

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

**Header rendering change** ([AlertsDetailModal.tsx:869-883](repos/case-management-system/frontend/src/features/alerts/components/AlertsDetailModal.tsx#L869-L883)):

```diff
- <h4 className="text-sm font-semibold text-gray-900 mb-4">
-   Triggered Typologies
- </h4>
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

**What it does.** When the modal opens for an alert, look up its lakehouse alert-navigator payload for the current user's tenant, extract the first typology that carries a `flowProcessorData` string, and show it inline with the section header.

**Interactions / risks.**

- The re-use of the existing `alertNavigatorService.getAlertNavigator` is appropriate — the same endpoint already backs the Alert Navigator visualization tab and returns `flowProcessorData?: string` on each `TypologyDto` (see [types/index.ts:16](repos/case-management-system/frontend/src/features/cases/components/view/visualizations/alertnavigator/types/index.ts#L16)). No new backend surface is required.
- The tenant lookup uses `authService.getUser()?.tenantId`, matching `VisualizationsTab` convention (called out in the inline comment). That is a legitimate choice, but it does mean cross-tenant investigators (if such a role exists) may not see EFRuP for alerts belonging to another tenant even though the alert itself is visible. Not a defect of this PR — worth being aware of.
- The `find(typology => typology.flowProcessorData)?.flowProcessorData` pattern silently drops flow-processor values from any typology beyond the first. If a real alert can carry more than one distinct flow-processor value across typologies, the UI collapses that ambiguity. Given the header is a single-value badge, this is a product decision, not a bug.

### 2. AlertsDetailModal.test.tsx — tests for EFRuP rendering / hiding

New service mocks ([:67-79](repos/case-management-system/frontend/src/features/alerts/components/__tests__/AlertsDetailModal.test.tsx#L67-L79)):

```tsx
vi.mock("@/features/auth/services/authService", () => ({
  default: {
    hasCMSComplianceOfficerRole: vi.fn(),
    getUser: vi.fn(),
  },
}));

vi.mock(
  "../../../cases/components/view/visualizations/alertnavigator/services",
  () => ({
    default: {
      getAlertNavigator: vi.fn(),
    },
  }),
);
```

Default happy-path setup in `beforeEach` ([:183-184](repos/case-management-system/frontend/src/features/alerts/components/__tests__/AlertsDetailModal.test.tsx#L183-L184)):

```tsx
mockGetUser.mockReturnValue({ userId: "user-1", tenantId: "DEFAULT" });
mockGetAlertNavigator.mockResolvedValue({ typologies: [] });
```

Two new tests: one asserts EFRuP renders when a typology carries `flowProcessorData: "Block"`, one asserts the label is absent when `typologies` is empty. Both also verify the header itself still renders.

[↑ Back to top](#pr-review-cms-245--fix-added-the-efrup-flow-processor-in-alert-detail-modal)

---

## Code Quality Analysis

### Strengths

- **Reuses existing service and types.** No duplicated fetch logic or ad-hoc types — the PR imports the same `alertNavigatorService` and `TypologyDto` shape already used by the Alert Navigator tab.
- **Fails safe.** On error or missing preconditions (`alert_id`, `tenantId`, empty typologies), the component resets `flowProcessorData` to `undefined` and the badge disappears — no stale value, no rendered error.
- **Minimal blast radius.** Two files, one net UI addition, no changes to services or types. Easy to revert if needed.
- **Rationale captured in code.** The inline comment on the tenant source (logged-in user's `tenantId`, not `alert.tenant_id`) documents a non-obvious choice so future readers won't flip it.
- **Tests cover both branches** of the conditional render (populated + empty) and assert the service was called with the exact `(alertId, tenantId)` pair.

### Issues and Observations

#### Issue 1 — Stale-alert flash between alerts due to `find` without reset before fetch

**Severity: Minor (Code Quality)**

The effect only resets `flowProcessorData` in the early-return / catch paths — not immediately when `alert.alert_id` changes. Concretely, if the modal is reused to switch from alert A (which has `flowProcessorData: "Block"`) to alert B (whose navigator response arrives seconds later with no flow processor), the badge from alert A remains visible until B's response resolves. The current tests cover initial-mount cases only, so the regression risk is small but real.

Suggested fix — clear at the start of every fetch:

```tsx
useEffect(() => {
  const fetchFlowProcessorData = async () => {
    setFlowProcessorData(undefined); // clear stale value before refetch
    const tenantId = authService.getUser()?.tenantId;
    if (!alert?.alert_id || !tenantId) return;
    // ...
  };
  fetchFlowProcessorData();
}, [alert?.alert_id]);
```

This is a small polish, not a merge blocker.

#### Issue 2 — Missing cleanup / cancellation for in-flight request

**Severity: Minor (Code Quality)**

Related to Issue 1: if the user closes the modal (or the parent unmounts) between `getAlertNavigator` being dispatched and its promise resolving, `setFlowProcessorData` will run on an unmounted component. React 18 tolerates this silently in production, but it can produce noisy dev-mode warnings and, together with Issue 1, is the mechanism behind the stale-value flash.

Suggested fix — either an `AbortController` or a simple mounted flag:

```tsx
useEffect(() => {
  let cancelled = false;
  const fetchFlowProcessorData = async () => {
    if (cancelled) return;
    // ... existing logic, but wrap every setFlowProcessorData with `if (!cancelled)`
  };
  fetchFlowProcessorData();
  return () => { cancelled = true; };
}, [alert?.alert_id]);
```

Non-blocking.

#### Issue 3 — Effect dependency omits `authService`

**Severity: Informational (Code Quality)**

`authService.getUser()?.tenantId` is read inside the effect but not listed in the dependency array. In practice the tenant of the logged-in user does not change while a modal is open, so this is fine and matches how `AlertNavigatorTab` treats the same lookup. Calling it out only because a stricter `react-hooks/exhaustive-deps` lint could pick at it.

#### Issue 4 — Layout: three-column grid with only two occupied cells

**Severity: Informational (Code Quality)**

`grid grid-cols-3 items-center` with the header in column 1 and the badge in the middle cell (`justify-self-center`) leaves column 3 empty and shifts the EFRuP badge to the visual centre of the section, not the right. That may be exactly what was intended (screenshots not provided in the PR). If the intent was right-aligned, `flex justify-between` reads more naturally. Confirm the placement matches the design before merge.

[↑ Back to top](#pr-review-cms-245--fix-added-the-efrup-flow-processor-in-alert-detail-modal)

---

## Security Assessment

| Concern | Assessment |
|---------|-----------|
| Auth / tenant boundary | Uses the logged-in user's `tenantId` (via `authService.getUser()`) as the query parameter, not any client-supplied value. This matches the `AlertNavigatorTab` pattern. No tenant escalation risk introduced. |
| Injection / XSS | `flowProcessorData` is rendered as a plain React child inside a `<span>`; React escapes it. No `dangerouslySetInnerHTML`, no URL construction from the payload. |
| Data exposure | Values shown come from the same lakehouse endpoint already reachable from the Alert Navigator tab for the same user. No new data path exposed. |
| Error handling | Errors are logged to the console and the badge is hidden. Nothing sensitive is surfaced in the UI. |

No new security vulnerabilities introduced by this PR.

[↑ Back to top](#pr-review-cms-245--fix-added-the-efrup-flow-processor-in-alert-detail-modal)

---

## Test Coverage

- **Populated typologies path** — covered: mocks `getAlertNavigator` to return one typology with `flowProcessorData: "Block"` and asserts both `EFRuP:` and `Block` render.
- **Empty typologies path** — covered: mocks `{ typologies: [] }` and asserts `EFRuP:` is absent while the "Triggered Typologies" header still renders.
- **Service arguments** — covered: asserts `getAlertNavigator` is called with `(baseAlert.alert_id, "DEFAULT")`.
- **Not tested:** error branch (rejected promise → badge hidden), missing-tenant branch (`getUser` returns `undefined`), and the alert-switch stale-value scenario described in Issue 1. Missing these is a minor gap, not a blocker.

PR checklist:
- [x] Locally tested
- [ ] Development Environment (unchecked — noted for the author)
- [ ] Not needed (unchecked)
- [x] Husky successfully run
- [x] Unit tests passing and documentation done

CI evidence: Node.js CI `check tests` job green.

[↑ Back to top](#pr-review-cms-245--fix-added-the-efrup-flow-processor-in-alert-detail-modal)

---

## CodeRabbit Activity

### Pass 1 — walkthrough only

**Commit reviewed:** `f27d887b`
**Findings:** 0 actionable comments (pre-merge checks all passed: docstring coverage, linked issues, out-of-scope, description, title). Only a walkthrough summary and pre-merge check widget were posted; no line-level suggestions were raised.

| Finding | Severity | Status |
|---------|----------|--------|
| _None_ | — | — |

[↑ Back to top](#pr-review-cms-245--fix-added-the-efrup-flow-processor-in-alert-detail-modal)

---

## Summary and Verdict

**Verdict: Approve with minor cleanup requested**

The change is small, well-scoped, reuses existing infrastructure (`alertNavigatorService`, `TypologyDto`), and behaves correctly on both the happy path and the empty-data path. Both branches of the conditional render are covered by new tests, CI is fully green, and no security concerns are introduced. The only items worth touching are the two related polish points — clearing `flowProcessorData` at the start of the effect to prevent a stale-value flash when the modal switches alerts, and adding a cancellation flag for in-flight requests. Neither blocks merge.

### Blocking

_None._

### Non-blocking but recommended

1. **Clear `flowProcessorData` at the start of the effect** — prevents stale value from a prior alert leaking into the modal for the next alert while its navigator request is in flight (Issue 1).
2. **Add a cancellation flag / `AbortController`** — avoids `setState` on unmount and complements the reset above (Issue 2).
3. **Confirm the EFRuP badge placement matches the design** — `grid-cols-3` + `justify-self-center` puts the badge in the visual middle of the row; if right-aligned was intended, use `flex justify-between` (Issue 4).

**On the "pull from dev" question:** not required for correctness — the PR branch is ~60 files behind `dev` but a three-way merge dry run shows no textual conflicts, and every symbol this PR imports (`alertNavigatorService`, `TypologyDto.flowProcessorData`) is present and stable on current `dev`. That said, dev has moved substantially since the branch was cut, so asking the author to `git pull origin dev` (or rebase) before merge is good hygiene — it eliminates the risk of a semantic (non-textual) regression from any of the intervening 60 files and gives CI a chance to run against the exact code that will land.

[↑ Back to top](#pr-review-cms-245--fix-added-the-efrup-flow-processor-in-alert-detail-modal)

---

## GitHub Review Comment

````markdown
**Approve with minor cleanup requested**

Small, well-scoped change that reuses the existing `alertNavigatorService` and correctly hides the EFRuP badge when there's no data. All CI green, no security concerns, both render branches covered by new tests. Two small polish items and one layout confirmation — none block merge, but worth addressing in this PR if convenient.

---

### Non-blocking (please address in this PR if possible)

**1. Clear `flowProcessorData` at the start of the fetch effect**

If the modal is reused to switch from an alert with `flowProcessorData` to one without, the previous alert's badge stays visible until the new response resolves. Reset up front so the badge is always in sync with the alert being viewed:

```tsx
useEffect(() => {
  let cancelled = false;
  const fetchFlowProcessorData = async () => {
    setFlowProcessorData(undefined); // clear stale value before refetch
    const tenantId = authService.getUser()?.tenantId;
    if (!alert?.alert_id || !tenantId) return;
    try {
      const navigatorData = await alertNavigatorService.getAlertNavigator(
        alert.alert_id,
        tenantId,
      );
      if (cancelled) return;
      setFlowProcessorData(
        navigatorData.typologies?.find((t) => t.flowProcessorData)?.flowProcessorData,
      );
    } catch (err) {
      if (cancelled) return;
      console.error('Failed to fetch alert navigator flow processor data:', err);
      setFlowProcessorData(undefined);
    }
  };
  fetchFlowProcessorData();
  return () => { cancelled = true; };
}, [alert?.alert_id]);
```

This rolls the reset and a cancellation flag into one change — the flag avoids `setState` on unmount if the modal closes mid-request.

**2. Confirm the EFRuP badge placement matches the design**

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

**3. Consider covering the error and missing-tenant branches in tests**

The new tests cover populated and empty typologies. A short case for `mockGetAlertNavigator.mockRejectedValue(...)` (badge hidden, no throw) and one for `mockGetUser.mockReturnValue(undefined)` (badge hidden, service not called) would close the loop.

---

### Merge hygiene (optional)

The branch is ~60 files behind `dev`. There are no textual conflicts, and every symbol this PR imports (`alertNavigatorService`, `TypologyDto.flowProcessorData`) is present on current `dev`, so this isn't required — but a `git pull --rebase origin dev` before merge would let CI validate against the exact code that lands.
````

[↑ Back to top](#pr-review-cms-245--fix-added-the-efrup-flow-processor-in-alert-detail-modal)
