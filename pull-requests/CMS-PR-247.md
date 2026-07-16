# PR Review: CMS #247 — fix: added the efrup subrule ref in alert detail tab

**Repo:** tazama-lf/case-management-system
**Branch:** `paysys/SLA-task` → `dev`
**Author:** ibadkhan088 (Ibad Ahmed Khan)
**Date Reviewed:** 2026-07-16
**Label:** bug
**Size:** +101 / -3 lines across 2 files
**Commits:** 1 (`c3261a4c`)
**State:** OPEN
**HEAD SHA verified:** `c3261a4cb772f7a6eaba8607846abe50074631fe`
**Existing approvals:** none — CodeRabbit posted the walkthrough only, no human approval yet

---

## Table of Contents

- [Overview](#overview)
- [What Changed (Detailed)](#what-changed-detailed)
  - [1. `AlertsDetailModal.tsx` — read EFRuP `subRuleRef` from `alert_data.tadpResult`](#1-alertsdetailmodaltsx--read-efrup-subruleref-from-alert_datatadpresult)
  - [2. `AlertsDetailModal.test.tsx` — assert badge from `tadpResult` when `alerted_typologies` is filtered](#2-alertsdetailmodaltesttsx--assert-badge-from-tadpresult-when-alerted_typologies-is-filtered)
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

This PR reworks how the **EFRuP** flow-processor badge is populated in the Alert Detail modal. It replaces the network-fetch approach that shipped in the now-closed PR [#245](https://github.com/tazama-lf/case-management-system/pull/245) — which called the `alert-navigator` lakehouse endpoint on modal open — with a purely client-side derivation that reads `alert_data.tadpResult.typologyResult[].ruleResults[].subRuleRef` from data already loaded by `getAlertById`. PR #245 was closed unmerged and #247 supersedes it on the same branch (`paysys/SLA-task`).

The rationale for the rewrite is captured directly in the new test's comment: **server-side filtering excludes typologies whose `result` is below `alertThreshold` from `alerted_typologies`**, but the raw `tadpResult` still carries them. Reading EFRuP from the unfiltered `tadpResult` therefore surfaces the flow-processor decision even when the typology didn't cross its own alert threshold — a functionally meaningful improvement over PR #245's approach (which relied on the alert-navigator response and would have missed EFRuP for filtered typologies) and, as a side benefit, eliminates the extra network request.

The PR targets `dev` — correct for this repo's flow. All 19 CI checks (CodeQL x2, njsscan, DCO, GPG signature verify, encoding-check, hadolint, dependency-review, node-ci build/style/tests, conventional-commits, CodeRabbit) pass. `mergeStateStatus` is `BLOCKED` only because no human approver has reviewed yet.

| File | Nature of Change |
|------|-----------------|
| `frontend/src/features/alerts/components/AlertsDetailModal.tsx` | Add `EFRUP_RULE_ID` constant + `extractFlowProcessorData()` helper that walks `alert.alert_data.tadpResult.typologyResult`; render EFRuP badge inside the Triggered Typologies header when a non-empty `subRuleRef` is found |
| `frontend/src/features/alerts/components/__tests__/AlertsDetailModal.test.tsx` | Two new specs: one asserts the badge renders from `tadpResult` when `alerted_typologies` is empty; one asserts the badge is absent when no rule has id `EFRuP@1.0.0` |

[↑ Back to top](#pr-review-cms-247--fix-added-the-efrup-subrule-ref-in-alert-detail-tab)

---

## What Changed (Detailed)

### 1. `AlertsDetailModal.tsx` — read EFRuP `subRuleRef` from `alert_data.tadpResult`

New constant and helper (added around line 166 in the file, immediately after the existing `extractTriggeredTypologies` at lines 129–164):

```typescript
const EFRUP_RULE_ID = 'EFRuP@1.0.0';
const extractFlowProcessorData = (
  alert: AlertWithAlertedTypologies,
): string | undefined => {
  const tadpResult = isRecord(alert.alert_data)
    ? alert.alert_data.tadpResult
    : undefined;
  const typologyResult = isRecord(tadpResult)
    ? tadpResult.typologyResult
    : undefined;
  if (!Array.isArray(typologyResult)) {
    return undefined;
  }

  for (const typology of typologyResult) {
    if (!isRecord(typology)) continue;
    const ruleResults = Array.isArray(typology.ruleResults)
      ? typology.ruleResults.filter(isRecord)
      : [];
    const flowProcessorRule = ruleResults.find(
      (rule) => rule.id === EFRUP_RULE_ID,
    );
    const subRuleRef = flowProcessorRule?.subRuleRef;
    if (typeof subRuleRef === 'string' && subRuleRef.trim()) {
      return subRuleRef;
    }
  }
  return undefined;
};
```

Call site (around line 534, alongside the existing `extractTriggeredTypologies` call):

```diff
   const triggeredTypologies = extractTriggeredTypologies(alert);
+  const flowProcessorData = extractFlowProcessorData(alert);
```

Render section (around line 868, replaces the plain header):

```diff
               <div className="rounded-lg border border-gray-200 bg-white p-5 mb-4">
-                <h4 className="text-sm font-semibold text-gray-900 mb-4">
-                  Triggered Typologies
-                </h4>
+                <div className="mb-4 grid grid-cols-3 items-center">
+                  <h4 className="text-sm font-semibold text-gray-900">
+                    Triggered Typologies
+                  </h4>
+
+                  {flowProcessorData && (
+                    <div className="justify-self-center text-sm font-semibold">
+                      <span className="text-gray-900">EFRuP:</span>{' '}
+                      <span className="text-red-600">{flowProcessorData}</span>
+                    </div>
+                  )}
+                </div>
```

**Behaviour.**
- Walks all typologies in `tadpResult.typologyResult`, returns the first `subRuleRef` on a rule whose `id === "EFRuP@1.0.0"`, provided the value is a non-empty string.
- All navigation steps are type-guarded with `isRecord` and `Array.isArray`; `.filter(isRecord)` keeps the loop safe against `null`/primitive elements the server might emit.
- `typeof subRuleRef === 'string' && subRuleRef.trim()` filters both non-string and whitespace-only values before rendering — matches the existing `asString` convention at line 114.
- The `flowProcessorData &&` conditional in JSX hides the badge entirely when the value is `undefined`, so the `grid-cols-3` row becomes a header-only row in that case.

**Data-source rationale (per new test comment).** `alerted_typologies` is filtered server-side to those where `result >= alertThreshold`. `tadpResult` is the raw pre-filter shape. So a typology whose score fell below its alert threshold but which nonetheless carries an EFRuP flow-processor directive from BIAR will not appear in `alerted_typologies` — but will appear in `tadpResult.typologyResult`. Reading EFRuP from the raw shape means the badge surfaces on those alerts, which was the motivation for reworking PR #245's approach (that PR sourced the value from the `alert-navigator` endpoint, whose upstream shape mirrors `alerted_typologies` and therefore had the same filtering blind spot).

**Consistency with `extractTriggeredTypologies`.** The existing helper reads *rendered* typologies from `alerted_typologies` (filtered) — that is the correct source for the visible typology cards, since it aligns with what the user sees as "triggered". The new helper reads from the unfiltered `tadpResult` — the correct source for a decision that the flow processor made regardless of the typology's own threshold. The two helpers pointing at different shapes is intentional, not a drift.

### 2. `AlertsDetailModal.test.tsx` — assert badge from `tadpResult` when `alerted_typologies` is filtered

Two new specs added at line 394. Both mirror the mocking style already used elsewhere in the file (`mockGetAlertById.mockResolvedValue`, `renderModal()`, `waitFor(...)`).

**Positive branch (lines 394–425):**

```typescript
it("displays the EFRuP subRuleRef as a badge, reading it from unfiltered alert_data even though the typology missed its own alertThreshold", async () => {
  mockGetAlertById.mockResolvedValue({
    ...baseAlert,
    // No alerted_typologies entry survives server-side filtering (result <
    // alertThreshold), so the badge must come from the raw tadpResult, not
    // from alerted_typologies.
    alerted_typologies: [],
    alert_data: {
      tadpResult: {
        typologyResult: [
          {
            id: "typology-1",
            cfg: "Money Laundering",
            result: 40,
            workflow: { alertThreshold: 50, interdictionThreshold: 80 },
            ruleResults: [
              { id: "075@1.0.0", cfg: "1.0.0", wght: 100, subRuleRef: ".02" },
              { id: "EFRuP@1.0.0", cfg: "1.0.0", wght: 0, subRuleRef: "Block" },
            ],
          },
        ],
      },
    },
  });
  ...
  expect(screen.getByText("EFRuP:")).toBeInTheDocument();
  expect(screen.getByText("Block")).toBeInTheDocument();
});
```

This is the regression-lock: even with `alerted_typologies: []`, the badge must appear from `tadpResult`. Removing the new helper would break this spec — exactly what a regression test should do.

**Negative branch (lines 427–450):**

```typescript
it("does not display an EFRuP badge when no rule has id EFRuP@1.0.0", async () => {
  mockGetAlertById.mockResolvedValue({
    ...baseAlert,
    alerted_typologies: [
      { id: "typology-1", cfg: "Money Laundering", result: 95, ...,
        ruleResults: [
          { id: "075@1.0.0", cfg: "1.0.0", wght: 100, subRuleRef: ".02" },
        ],
      },
    ],
  });
  ...
  expect(screen.getByText("Triggered Typologies")).toBeInTheDocument();
  expect(screen.queryByText("EFRuP:")).not.toBeInTheDocument();
});
```

Covers the "no EFRuP rule anywhere in the payload" path.

**Small note on the negative spec.** It only exercises the case where `alert_data` is entirely absent from the mock (i.e. `extractFlowProcessorData` short-circuits at the first `isRecord(alert.alert_data)` check). It does *not* exercise the case where `alert_data.tadpResult.typologyResult` is present but no `ruleResults` entry has `id === "EFRuP@1.0.0"`, nor the case where an EFRuP rule exists but its `subRuleRef` is empty/whitespace/non-string. These would exercise the deeper branches inside the loop and the `trim()` guard. See Issue 3.

[↑ Back to top](#pr-review-cms-247--fix-added-the-efrup-subrule-ref-in-alert-detail-tab)

---

## Code Quality Analysis

### Strengths

- **Correct root cause identified.** PR #245 read EFRuP from a source (`alert-navigator` response) that inherited the server-side alertThreshold filter and would have silently omitted the badge on sub-threshold alerts. This PR reads from the raw `tadpResult`, which is the pre-filter shape. That's a real correctness win, not just refactoring.
- **Eliminates a network round-trip.** PR #245 fired a `useEffect`-driven call to the lakehouse `alert-navigator` endpoint whenever the modal opened. This PR uses data already in the loaded alert, removing a dependency on the lakehouse being reachable for a UI badge to render.
- **Type-safe traversal.** Every step uses `isRecord` / `Array.isArray` / `filter(isRecord)`; the `subRuleRef` is validated as a non-empty string before use. No `as any` casts, no chained optional chaining that could silently produce `undefined` on a mistyped path.
- **Named constant** for the rule id (`EFRUP_RULE_ID`) rather than a stringly-typed literal inline — matches the style used elsewhere in the modal file.
- **New test lock for the regression case** (`alerted_typologies: []` + populated `tadpResult`) — this is the case that motivated the rewrite, and the spec would fail if the helper were reverted to reading from `alerted_typologies`.
- **Diff is confined to one component and its spec file.** No collateral edits, no import churn beyond what's needed.
- **CI is fully green** (19 checks, including CodeQL, njsscan, encoding-check).

### Issues and Observations

#### Issue 1 — First-match iteration order across multiple typologies with EFRuP

**Severity: Minor (Code Quality)**

The helper returns the `subRuleRef` from the first typology whose `ruleResults` contains `id === "EFRuP@1.0.0"` with a non-empty string. If a real alert can carry `typologyResult` with more than one typology, and different typologies could have different EFRuP `subRuleRef` values (e.g. one says `"Block"`, another says `"Override"`), the badge renders only the first-seen value with no indication that others exist.

Whether this is a defect depends on the semantics of EFRuP:
- If EFRuP is an alert-level decision (one flow-processor outcome per alert, replicated across each typology entry as convenience), the first-match approach is fine and matches how PR #245 read it.
- If EFRuP is a per-typology decision, the badge is showing a truncated view.

Given PR #245 shipped with the same first-match semantics and no reviewer flagged it there, I lean toward "alert-level decision, first-match is fine" — but the behaviour deserves a one-line comment above the loop explaining the intent, so a future reader doesn't need to reason about it from scratch.

**Suggested comment:**

```typescript
// EFRuP is an alert-level flow-processor decision; the same value is expected on
// every typology's rule list, so returning the first non-empty subRuleRef is
// sufficient. If BIAR ever emits per-typology values, this needs to become an aggregation.
for (const typology of typologyResult) { ... }
```

#### Issue 2 — Three-column grid leaves the third column empty

**Severity: Minor (UI / Maintainability)**

The header row uses:

```tsx
<div className="mb-4 grid grid-cols-3 items-center">
  <h4 ...>Triggered Typologies</h4>
  {flowProcessorData && (
    <div className="justify-self-center ...">
      <span>EFRuP:</span> <span>{flowProcessorData}</span>
    </div>
  )}
</div>
```

With `grid-cols-3` and only two cells occupied, the badge sits in the middle column while column 3 is empty — this visually places the EFRuP indicator at the geometric centre of the section header, not right-aligned. That may be intentional (screenshots not provided). If the design calls for a right-aligned badge, `flex justify-between` (or `grid-cols-2 justify-items-end` on the badge cell) would read more naturally and future-proof against layout tweaks.

This is the same layout concern flagged on PR #245's review (Issue 4 there). Confirming placement against the design is worth doing before merge.

#### Issue 3 — Incomplete branch coverage in the negative test

**Severity: Minor (Test Coverage)**

The "no EFRuP badge" spec passes a mock that lacks `alert_data` entirely, so `extractFlowProcessorData` short-circuits at the first `isRecord(alert.alert_data)` check. The helper has three additional paths that no spec exercises:

1. `alert_data` present, `tadpResult` missing / not a record — short-circuit at the second guard.
2. `alert_data.tadpResult.typologyResult` present but no rule with `id === "EFRuP@1.0.0"` — loop completes without hitting the return.
3. Rule with `id === "EFRuP@1.0.0"` present but `subRuleRef` is empty string, whitespace, `null`, or a non-string — the `typeof === 'string' && .trim()` guard rejects it.

Any of these would be a one-mock spec addition. Not blocking — the helper's structure is simple enough that inspection covers it — but future-proofing the trim/type guard would prevent an easy regression like `subRuleRef: number` being accidentally accepted.

**Suggested minimal addition (one spec covering path 3):**

```typescript
it("does not display an EFRuP badge when the EFRuP rule's subRuleRef is empty", async () => {
  mockGetAlertById.mockResolvedValue({
    ...baseAlert,
    alerted_typologies: [],
    alert_data: {
      tadpResult: {
        typologyResult: [
          {
            id: "typology-1",
            ruleResults: [
              { id: "EFRuP@1.0.0", cfg: "1.0.0", wght: 0, subRuleRef: "   " },
            ],
          },
        ],
      },
    },
  });
  renderModal();
  await waitFor(() => {
    expect(screen.getByText("Triggered Typologies")).toBeInTheDocument();
  });
  expect(screen.queryByText("EFRuP:")).not.toBeInTheDocument();
});
```

#### Issue 4 — Positive test's `workflow` field is unused by the helper

**Severity: Informational (Test Fidelity)**

The positive spec mocks a typology entry with `workflow: { alertThreshold: 50, interdictionThreshold: 80 }`. The new helper never reads `workflow`, so this field is fixture noise — it doesn't influence pass/fail. It's harmless, but if `alerted_typologies` mocks elsewhere in the file expose these values at the top level (as they do — e.g. lines 435–436 in the sibling test place `alertThreshold` and `interdictionThreshold` as siblings of `id`), a future reader may assume the two shapes are interchangeable when they aren't. Consider dropping the unused `workflow` from the positive spec's fixture, or adding a comment noting it's there to match the raw BIAR shape but not read by the code under test.

[↑ Back to top](#pr-review-cms-247--fix-added-the-efrup-subrule-ref-in-alert-detail-tab)

---

## Security Assessment

| Concern | Assessment |
|---------|-----------|
| XSS from rendering `subRuleRef` | Value flows into a React text node: `<span className="text-red-600">{flowProcessorData}</span>`. React escapes text nodes by default. `subRuleRef` is validated as a non-empty string; even if a hostile server returned `<script>`, React would render it as literal text. No new XSS surface. |
| Prototype pollution / injection via `alert.alert_data` traversal | All property accesses go through `isRecord` (which is `!!value && typeof === 'object' && !Array.isArray`) and `Array.isArray`. `filter(isRecord)` drops any non-object elements. The helper never uses computed indexing or dynamic property names on user data. Safe. |
| Untrusted input reaching a data or command layer | N/A — pure client-side derivation, no outbound calls. |
| Auth-guard decorators / tenant scoping | N/A — no new endpoint; the PR *removes* the network call that PR #245 added, so if anything this reduces the auth-scoped surface. |
| URL construction / open-redirect / SSRF | N/A — no URL construction. |
| Secrets / PII in logs, errors, and API responses | The helper does not log. No new console/error output. |
| Removal of a network fetch | Reduces attack surface (one fewer authenticated request per modal open). Positive. |
| CI security-scanner status | CodeQL (actions + javascript-typescript), njsscan, and dependency-review all pass. |

No new security vulnerabilities introduced by this PR. Removing the network fetch is a small net reduction in surface.

[↑ Back to top](#pr-review-cms-247--fix-added-the-efrup-subrule-ref-in-alert-detail-tab)

---

## Test Coverage

**Covered by this PR:**
- The **primary behavioural change** — badge sourced from raw `tadpResult` even when `alerted_typologies` is empty — is locked in by the new positive spec. This is the case that motivated the rewrite, and the test would fail if the helper reverted to reading from `alerted_typologies`.
- The **basic "no EFRuP" hiding** — badge is absent when `alert_data` is missing altogether — is covered by the new negative spec.
- CI: full `node-ci / check tests` passes at HEAD `c3261a4c`.

**Not covered by this PR:**
- **Deeper negative branches** in `extractFlowProcessorData`: (a) `alert_data` present but `tadpResult` missing, (b) `typologyResult` present but no rule with `id === "EFRuP@1.0.0"`, (c) EFRuP rule present but `subRuleRef` empty / whitespace / non-string. See Issue 3. All three would be one-mock spec additions.
- **First-match semantics across multiple typologies** — no spec asserts that when `typologyResult` contains two typologies each carrying a different EFRuP `subRuleRef`, the helper returns the first-seen. See Issue 1. If EFRuP is genuinely alert-level (same value on every typology), this is uninteresting to test; if it's per-typology, a spec would be actively wrong to lock in first-match. Worth resolving the semantic question first, then deciding.
- **The `workflow` fixture field** in the positive spec is unused (Issue 4) — cosmetic.

**Regression tests for the reason the rewrite happened.** The new positive spec's comment explicitly documents the "score < alertThreshold" filtering that motivated the rewrite, and the mock uses `result: 40, alertThreshold: 50` with `alerted_typologies: []` to encode that assumption. This is exactly the kind of regression lock that Section 3.1's mindset calls for — it ties the fix's rationale to a repeatable assertion.

No blocker on coverage. Every issue in this review is `Minor` or `Informational`; there is no `Major` to bind a regression test to.

[↑ Back to top](#pr-review-cms-247--fix-added-the-efrup-subrule-ref-in-alert-detail-tab)

---

## CodeRabbit Activity

CodeRabbit posted only its automated walkthrough — no actionable findings, no inline comments, no review at severity above informational. `gh api pulls/247/reviews` returns `[]` and `gh api pulls/247/comments` returns `[]`.

### Pass 1 — Auto-walkthrough

**Commit reviewed:** `c3261a4c` (via post-push walkthrough)
**Findings:** 0 actionable comments

| Finding | Severity | Status |
|---------|----------|--------|
| — | — | — |

CodeRabbit's walkthrough itself confirms the two behavioural branches (populated `tadpResult` renders the badge; missing EFRuP rule hides it) match the two new tests, and correctly flags PR #245 as related. Pre-merge checks: all 5 passed. Nothing to reconcile against my own findings.

[↑ Back to top](#pr-review-cms-247--fix-added-the-efrup-subrule-ref-in-alert-detail-tab)

---

## Summary and Verdict

**Verdict: Approved**

This is a well-motivated, tightly-scoped rewrite of PR #245's EFRuP display. Sourcing the value from the raw `alert_data.tadpResult` rather than from `alerted_typologies` / the `alert-navigator` endpoint fixes a real correctness gap (sub-threshold typologies would have silently dropped the badge under the old approach) and eliminates a network round-trip per modal open. The traversal is type-safe end-to-end, the diff is confined to two files, the primary behavioural change is locked in by a regression spec that captures its rationale in a comment, and every CI check is green. No blockers.

### Blocking

None.

### Non-blocking but recommended

1. **Document first-match semantics** (Issue 1) — a one-line comment above the `for` loop stating that EFRuP is treated as an alert-level decision (so the first non-empty `subRuleRef` is sufficient) would save a future reader from re-deriving the assumption.
2. **Confirm the three-column grid placement matches the design** (Issue 2) — same layout concern as PR #245's review; `grid-cols-3 + justify-self-center` centres the badge in the row rather than right-aligning it. If right-align was intended, `flex justify-between` reads more naturally.
3. **Add one small negative-branch spec** (Issue 3) — the current negative spec only exercises the "no `alert_data`" short-circuit; a whitespace-`subRuleRef` case would lock in the `typeof === 'string' && .trim()` guard.

[↑ Back to top](#pr-review-cms-247--fix-added-the-efrup-subrule-ref-in-alert-detail-tab)

---

## GitHub Review Comment

````markdown
**Approved**

This PR rewrites PR #245's EFRuP badge to read from `alert.alert_data.tadpResult.typologyResult[].ruleResults[].subRuleRef` instead of the network-fetched `alert-navigator` response. Correct choice: `alerted_typologies` is filtered server-side to `result >= alertThreshold`, so PR #245's source had a blind spot for sub-threshold typologies that still carry a flow-processor directive. The new helper reads the pre-filter shape, is type-safe end-to-end (`isRecord` / `Array.isArray` / `filter(isRecord)` at every step, plus a `typeof === 'string' && .trim()` guard on the value), and the new positive spec locks in the "score < alertThreshold, badge still renders" regression case explicitly. CI green (19/19). No blockers.

---

### Non-blocking (please address in this PR if possible)

**1. Document first-match semantics on the loop**

`AlertsDetailModal.tsx:180-192`. The loop returns on the first non-empty `subRuleRef`. If EFRuP is an alert-level decision (same value replicated on every typology) that's fine; if it's per-typology, the badge silently truncates. A one-line comment above the loop noting the assumption would spare a future reader from re-deriving it:

```typescript
// EFRuP is an alert-level flow-processor decision; the same value is expected on
// every typology's rule list, so returning the first non-empty subRuleRef is
// sufficient. If BIAR ever emits per-typology values, this needs to aggregate.
for (const typology of typologyResult) { ... }
```

**2. Confirm the badge placement matches the design**

`AlertsDetailModal.tsx:869`. The header uses `grid grid-cols-3 items-center` with the badge in the middle column (`justify-self-center`), leaving column 3 empty. That visually centres the EFRuP badge in the row rather than right-aligning it. If right-aligned was the intent, `flex justify-between` reads more naturally:

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

(This is the same layout note that came up on PR #245.)

**3. Add one small negative-branch spec**

`AlertsDetailModal.test.tsx:427-450`. The current "no badge" spec only exercises the case where `alert_data` is absent entirely, which short-circuits at the first `isRecord` guard. A spec covering the "EFRuP rule present, `subRuleRef` whitespace-only" path would lock in the `typeof === 'string' && subRuleRef.trim()` guard against a future change accidentally accepting non-string or empty values:

```typescript
it("does not display an EFRuP badge when the EFRuP rule's subRuleRef is empty", async () => {
  mockGetAlertById.mockResolvedValue({
    ...baseAlert,
    alerted_typologies: [],
    alert_data: {
      tadpResult: {
        typologyResult: [{
          ruleResults: [{ id: "EFRuP@1.0.0", subRuleRef: "   " }],
        }],
      },
    },
  });
  renderModal();
  await waitFor(() => expect(screen.getByText("Triggered Typologies")).toBeInTheDocument());
  expect(screen.queryByText("EFRuP:")).not.toBeInTheDocument();
});
```
```
````

[↑ Back to top](#pr-review-cms-247--fix-added-the-efrup-subrule-ref-in-alert-detail-tab)
