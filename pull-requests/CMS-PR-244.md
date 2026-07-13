# PR Review: CMS #244 — fix: add SLA-state filter and expand case search normalization

**Repo:** tazama-lf/case-management-system
**Branch:** `paysys/SLA-task` → `dev`
**Author:** ibadkhan088 (Ibad Ahmed Khan)
**Label:** bug
**State:** OPEN (mergeStateStatus: BLOCKED, mergeable: MERGEABLE)
**Existing approvals:** None (CodeRabbit posted 1 actionable comment on the initial commit, declined by author as an accepted trade-off)

---

## Table of Contents

- [Initial Review (2026-07-13)](#initial-review-2026-07-13)
  - [Overview](#overview)
  - [What Changed (Detailed)](#what-changed-detailed)
    - [1. `backend/src/modules/case/dto/get-all-cases.dto.ts` — new optional `slaState` query param](#1-backendsrcmodulescasedtoget-all-casesdtots--new-optional-slastate-query-param)
    - [2. `backend/src/modules/case/services/case-query.service.ts` — search refactor + SLA state search + SLA state filter](#2-backendsrcmodulescaseservicescase-queryservicets--search-refactor--sla-state-search--sla-state-filter)
    - [3. `backend/test/case-query.service.spec.ts` — new tests for SLA filter/search, case-id prefix, priority partial match](#3-backendtestcase-queryservicespects--new-tests)
    - [4. Frontend hook `useCaseDashboard.ts` — `slaStateFilter` state and pagination reset](#4-frontendsrcfeaturescaseshooksusecasedashboardts--slastatefilter-state-and-pagination-reset)
    - [5. Frontend `caseService.ts` — forward `slaState` in query string](#5-frontendsrcfeaturescasesservicescaseservicets--forward-slastate-in-query-string)
    - [6. Frontend `CaseDashboardContainer.tsx` / `CaseDashboardContent.tsx` — wire new callback](#6-frontendsrcfeaturescasescomponentscasedashboardcontainertsx--casedashboardcontenttsx--wire-new-callback)
    - [7. Frontend `CaseFilters.tsx` — SLA state dropdown, saved-filter payload, clear handler refactor, placeholder change](#7-frontendsrcfeaturescasescomponentscasefilterstsx--sla-state-dropdown-saved-filter-payload-clear-handler-refactor-placeholder-change)
    - [8. Frontend test files — added `slaStateFilter` / callback to fixtures, updated placeholder assertions](#8-frontend-test-files)
  - [Code Quality Analysis](#code-quality-analysis)
    - [Strengths](#strengths)
    - [Issues and Observations](#issues-and-observations)
  - [Security Assessment](#security-assessment)
  - [Test Coverage](#test-coverage)
  - [CodeRabbit Activity](#coderabbit-activity)
  - [Summary and Verdict](#summary-and-verdict)
- [Follow-up Review (2026-07-13)](#follow-up-review-2026-07-13)
  - [Resolution Status](#resolution-status)
    - [Item 1 — Frontend behavioural tests for SLA dropdown, clear-filters, saved-filter round-trip](#item-1--frontend-behavioural-tests-for-sla-dropdown-clear-filters-saved-filter-round-trip)
    - [Item 2 — Style consistency in `caseService.ts` (braced `if`)](#item-2--style-consistency-in-caseservicets-braced-if)
    - [Item 3 — Tighten PR title before merge](#item-3--tighten-pr-title-before-merge)
  - [New Issues Found in Updated Commits](#new-issues-found-in-updated-commits)
  - [Final Verdict](#final-verdict)
  - [GitHub Review Comment](#github-review-comment)

---

## Initial Review (2026-07-13)

**Reviewed commit:** `b4f66cec42f415aaab0a1490b7d7cce34c850e08` — *"fix: fixed the case search to incorporate new columns and added filte…rs to search SLA states"* (2026-07-13)
**Size at time of review:** +362 / -84 lines across 11 files
**CI at time of review:** One transient `validate-pr-title` failure on a superseded run; all other checks green.

### Overview

This PR extends the case search and filtering surface to include the derived `SlaState` value (`ON_TRACK`, `AT_RISK`, `DUE_SOON`, `BREACHED`), and refines the existing free-text search to accept an explicit `"Case"`/`"CASE-"` ID prefix, partial priority matching, and separator-normalised enum matching (so `"at risk"` / `"at-risk"` / `"AT_RISK"` all match).

Because `sla_state` is not a persisted column (it is computed at read time by [`computeCaseSlaState`](repos/case-management-system/backend/src/modules/case/services/case-query.service.ts) using `sla_due_at`, `sla_started_at`, and tenant-specific SLA escalation ratios), the backend implements both the explicit filter and the search-term match via an in-memory candidate scan: fetch all rows matching every *other* filter, compute `sla_state` per row, then narrow the paginated query to the matching `case_id` set. The trade-off is discussed in a CodeRabbit thread and explicitly accepted by the author (see [CodeRabbit Activity](#coderabbit-activity)).

Frontend changes plumb `slaStateFilter` through the dashboard hook, container, content component, and filter panel, add the new field to the saved-filter payload/parser, and refactor the clear-filters logic into a named handler. The search placeholder is updated to reflect the expanded search surface.

**Base branch:** `dev` — correct for this repo's flow.

**CI:** One `conventional-commits / validate-pr-title` check is reported as FAILURE ([run 29253624633](https://github.com/tazama-lf/case-management-system/actions/runs/29253624633/job/86827909500)), but its only step is `Set up job` and the same workflow ran successfully four other times on this PR (runs 29253618800, 29253619160, 29253624741, 29253722492, 29253869979). This looks like a transient CI infra failure on a superseded run, not a real title-format violation. The PR title itself is valid Conventional Commits (`fix: …`), though it is visibly truncated (`… added filte…`) in GitHub metadata because it exceeds the 100-char header limit; the tail (`…rs to search SLA states`) survives in the commit body. Worth tightening in the merge commit but not blocking.

| File | Nature of Change |
|------|-----------------|
| `backend/src/modules/case/dto/get-all-cases.dto.ts` | Adds optional `slaState?: SlaState` query field |
| `backend/src/modules/case/services/case-query.service.ts` | Reworks the free-text search block (adds "Case" prefix, priority partial match, SLA-state partial match, separator normalisation) and adds an SLA-state filter step before the paginated query |
| `backend/test/case-query.service.spec.ts` | New tests: SLA-state filter, "Case" prefix search, priority partial match, SLA-state partial search (4 term variants + multi-word "due soon") |
| `frontend/src/features/cases/hooks/useCaseDashboard.ts` | Adds `slaStateFilter` state, forwards it to the request, includes it in the pagination-reset dependency list |
| `frontend/src/features/cases/services/caseService.ts` | Adds `slaState` to the DTO and appends it to the query string when present |
| `frontend/src/features/cases/components/CaseDashboardContainer.tsx` | Passes `onSlaStateFilterChange` through |
| `frontend/src/features/cases/components/CaseDashboardContent.tsx` | Same wiring at the content layer |
| `frontend/src/features/cases/components/CaseFilters.tsx` | Adds the SLA state dropdown, updates saved-filter payload/label parsing, refactors clear-filters into `handleClearFilters`, updates placeholder |
| `frontend/src/features/cases/components/__tests__/CaseDashboardContainer.test.tsx` | Adds fixtures for `slaStateFilter` / `setSlaStateFilter` |
| `frontend/src/features/cases/components/__tests__/CaseDashboardContent.test.tsx` | Same fixture updates |
| `frontend/src/features/cases/components/__tests__/CaseFilters.test.tsx` | Updates placeholder assertions and adds `slaStateFilter` / `onSlaStateFilterChange` props |

[↑ Back to top](#pr-review-cms-244--fix-add-sla-state-filter-and-expand-case-search-normalization)

---

### What Changed (Detailed)

#### 1. `backend/src/modules/case/dto/get-all-cases.dto.ts` — new optional `slaState` query param

```diff
- import { CaseStatus, Priority, CaseType } from '@prisma/client-cms';
+ import { CaseStatus, Priority, CaseType, SlaState } from '@prisma/client-cms';
```

```typescript
@ApiProperty({
  description: 'Filter by SLA state',
  enum: SlaState,
  required: false,
  example: 'AT_RISK',
})
@IsOptional()
@IsEnum(SlaState)
slaState?: SlaState;
```

Class-validator + Prisma-generated enum wiring is consistent with the surrounding `status`/`priority`/`caseType` fields. No breaking change: field is optional and absent behaviour is unchanged.

---

#### 2. `backend/src/modules/case/services/case-query.service.ts` — search refactor + SLA state search + SLA state filter

**a) Slate-state destructuring and separator normalisation** ([case-query.service.ts:319-321, 378-380](repos/case-management-system/backend/src/modules/case/services/case-query.service.ts#L319))

```typescript
// Before
const {
  ...,
  closedOnly = false,
} = query;

// After
const {
  ...,
  closedOnly = false,
  slaState,
} = query;
```

```typescript
// New (after searchUpper / normalizedSearch)
// Enum values are SCREAMING_SNAKE_CASE (e.g. "DUE_SOON", "AT_RISK") but users type them
// with spaces or dashes (e.g. "due soon"), so collapse those separators to underscores
// before matching against enum literals below.
const searchEnumMatch = searchUpper.replace(/[\s\-]+/gv, '_');
```

The comment justifies why a new normalisation constant is needed alongside the existing `searchUpper`/`normalizedSearch`. It is then reused for `caseTypeEnums`, `statusEnums`, `taskStatusEnums`, `priorityEnums`, and `slaStateEnums` filtering, replacing the previous `searchUpper.includes(...)` scheme. This is a correctness improvement — e.g. `"fraud and aml"` and `"pending case"` now match `FRAUD_AND_AML` and `STATUS_01_PENDING_CASE_CREATION_APPROVAL` respectively, which they did not before.

**b) `"Case"`/`"CASE-"` prefix on case_id search** ([case-query.service.ts:414-424](repos/case-management-system/backend/src/modules/case/services/case-query.service.ts#L414))

```typescript
// 1. SEARCH by case_id — supports an explicit "Case"/"CASE-" prefix (e.g. "Case 123",
// "CASE-123"), mirroring how Alert ID search strips a leading "Alert" word before
// matching on alert_id. A successful prefixed match is unambiguous, so short-circuit
// past the other search branches below instead of also treating "123" as e.g. a
// confidence score.
const caseIdSearch = searchTerm.replace(/^case(?:-|_|\s)*/iv, '');
const numericCaseIdSearch = Number(caseIdSearch);
const isPrefixedCaseIdSearch = caseIdSearch !== searchTerm && caseIdSearch !== '' && !Number.isNaN(numericCaseIdSearch);

const numericSearch = parseInt(searchTerm, 10);

if (isPrefixedCaseIdSearch) {
  orConditions.push({ case_id: numericCaseIdSearch });
} else {
  // ... all pre-existing search branches, now wrapped in this else
}
```

The `iv` regex flags (`i` = case-insensitive, `v` = Unicode-mode with strict escaping) are supported on modern V8; consistent with `gv` used elsewhere in this diff. The short-circuit means `"CASE-123abc"` will *not* be treated as a plain-text/N/A search — it will simply produce no case-id match and no other branch fires. That is intentional per the comment ("A successful prefixed match is unambiguous"), and matches the Alert ID pattern the author cites.

**Edge case (informational):** `Number("")` is `0`, not `NaN`, so the explicit `caseIdSearch !== ''` guard is required — without it, a bare `"Case"` or `"CASE-"` would search for `case_id: 0`. The guard is present, so this is fine.

**c) Priority partial search** ([case-query.service.ts:475-482](repos/case-management-system/backend/src/modules/case/services/case-query.service.ts#L475))

```typescript
// 7. PARTIAL SEARCH by priority (e.g., "hi" matches "HIGH")
const priorityEnums = ['LOW', 'MEDIUM', 'HIGH'];
const matchingPriorities = priorityEnums.filter((p) => p.includes(searchEnumMatch));
if (matchingPriorities.length > 0) {
  orConditions.push({
    priority: { in: matchingPriorities },
  });
}
```

Standard substring-match pattern, mirroring the existing status/case-type approach. One subtle behavioural change: because `''.includes('')` is `true`, an empty `searchEnumMatch` would match *every* priority — but this branch only executes inside the `search.trim()` block, so `searchEnumMatch` is never empty here. Safe.

**d) SLA-state partial search inside free-text search** ([case-query.service.ts:517-537](repos/case-management-system/backend/src/modules/case/services/case-query.service.ts#L517))

```typescript
// 8. PARTIAL SEARCH by SLA state. sla_state isn't a DB column (it's derived at read
// time from sla_due_at/sla_started_at), so it can't be matched in the where clause
// directly. Only pay for the extra lookup when the term could plausibly match one of
// the enum values, then resolve which case_ids currently compute to a matching state.
const slaStateEnums = ['ON_TRACK', 'AT_RISK', 'DUE_SOON', 'BREACHED'];
const matchingSlaStates = slaStateEnums.filter((state) => state.includes(searchEnumMatch));
if (matchingSlaStates.length > 0) {
  const slaSearchCandidates = await this.prismaService.case.findMany({
    where: baseFilters,
    select: { case_id: true, tenant_id: true, sla_due_at: true, sla_started_at: true },
  });
  const slaSearchRatiosByTenant = await this.resolveRatiosByTenant(slaSearchCandidates.map((caseItem) => caseItem.tenant_id));
  const matchingSlaCaseIds = slaSearchCandidates
    .filter((caseItem) => {
      const candidateState = computeCaseSlaState(caseItem, slaSearchRatiosByTenant.get(caseItem.tenant_id)!);
      return candidateState !== null && matchingSlaStates.includes(candidateState);
    })
    .map((caseItem) => caseItem.case_id);
  orConditions.push({
    case_id: { in: matchingSlaCaseIds },
  });
}
```

- Uses `baseFilters` (tenant-scoped baseline) rather than the in-progress `whereClause`, which is what you want for a search-OR branch (this branch's role is to produce a set of case_ids to OR into the search predicate, not to be intersected with in-progress AND branches).
- `slaSearchRatiosByTenant.get(caseItem.tenant_id)!` — non-null assertion is safe because `resolveRatiosByTenant` populates the map from the same tenant list.
- `computeCaseSlaState` can return `null` — the `candidateState !== null` guard handles it.
- Empty `matchingSlaCaseIds` produces `{ case_id: { in: [] } }`, which Prisma resolves to zero matches — this branch then contributes nothing new via OR, which is the correct outcome.

**e) SLA-state explicit filter, applied after the where-clause is built** ([case-query.service.ts:666-680](repos/case-management-system/backend/src/modules/case/services/case-query.service.ts#L666))

```typescript
// sla_state is derived at read time (not a DB column), so it can't be filtered
// in the Prisma where clause directly. Resolve the case_ids matching every other
// filter first, compute sla_state for those candidates, then narrow whereClause
// to that id set before running the paginated count/findMany below.
if (slaState) {
  const candidateCases = await this.prismaService.case.findMany({
    where: whereClause,
    select: { case_id: true, tenant_id: true, sla_due_at: true, sla_started_at: true },
  });
  const candidateRatiosByTenant = await this.resolveRatiosByTenant(candidateCases.map((caseItem) => caseItem.tenant_id));
  const matchingCaseIds = candidateCases
    .filter((caseItem) => computeCaseSlaState(caseItem, candidateRatiosByTenant.get(caseItem.tenant_id)!) === slaState)
    .map((caseItem) => caseItem.case_id);
  whereClause.case_id = { in: matchingCaseIds };
}
```

- Runs against the fully-composed `whereClause` (post-AND-composition), so it intersects with every other filter — correct for the AND semantics of an explicit filter (as opposed to the OR-search branch above).
- Overwrites `whereClause.case_id` — safe because there is no other path in this method that sets `case_id` on `whereClause` before this point.
- Same trade-off as the search branch: this loads every candidate row in memory. See [CodeRabbit Activity](#coderabbit-activity) for the author's rationale for accepting this.

---

#### 3. `backend/test/case-query.service.spec.ts` — new tests

Six new test cases exercise the new logic:

1. `should filter by SLA state via a candidate lookup before pagination` — asserts the paginated `findMany` receives `where.case_id: { in: [atRiskCandidate.case_id] }` after the candidate scan filters out the on-track case.
2. `should search by case id with a "Case" prefix, mirroring Alert ID search` — asserts the OR includes `{ case_id: 1 }` for search `"Case 1"`.
3. `should partially search by priority` — asserts `{ priority: { in: ['HIGH'] } }` in the OR for search `"hi"`.
4. `it.each([['risk'], ['at risk'], ['at-risk'], ['AT_RISK']])` — asserts each variant maps to `AT_RISK` via the candidate lookup.
5. `should partially search by SLA state using "due soon" (multi-word, matches DUE_SOON)` — asserts the multi-word input reaches the candidate scan and produces `DUE_SOON` matches.

Fixtures use realistic `sla_due_at` / `sla_started_at` offsets from `now` so `computeCaseSlaState` actually returns the expected state. Good.

---

#### 4. `frontend/src/features/cases/hooks/useCaseDashboard.ts` — `slaStateFilter` state and pagination reset

```diff
  const [sarStrStatusFilter, setSarStrStatusFilter] = useState<string>('');
+ const [slaStateFilter, setSlaStateFilter] = useState<string>('');
```

The new filter is:
- Passed to the request DTO: `slaState: slaStateFilter || undefined`
- Added to the `fetchCases` dependency array (line 202)
- Added to the pagination-reset `useEffect` dependency array (line 261) — so changing the SLA filter resets the page to 1, consistent with other filters
- Exposed via `filterActions.setSlaStateFilter` and `filters.slaStateFilter`

Consistent with the surrounding filter pattern.

---

#### 5. `frontend/src/features/cases/services/caseService.ts` — forward `slaState` in query string

```diff
  if (query?.sarStrStatus) {
    params.append('sarStrStatus', query.sarStrStatus);
  }
+ if (query?.slaState) params.append('slaState', query.slaState);
```

`GetUserCasesQueryDto` gets the field, and the request builder appends it. Trivial.

**Style nit:** the existing `sarStrStatus` block uses a braced `if`, the new `slaState` block uses a single-line `if` — minor inconsistency, not blocking. *(Resolved in follow-up commit `8f88adc8` — see [Item 2](#item-2--style-consistency-in-caseservicets-braced-if).)*

---

#### 6. `frontend/src/features/cases/components/CaseDashboardContainer.tsx` / `CaseDashboardContent.tsx` — wire new callback

Straight prop-drilling: `filterActions.setSlaStateFilter` → `onSlaStateFilterChange` → `CaseFilters`. No logic added at these layers.

---

#### 7. `frontend/src/features/cases/components/CaseFilters.tsx` — SLA state dropdown, saved-filter payload, clear handler refactor, placeholder change

Key additions:

**Saved-filter type expanded:**

```diff
  export interface UserSavedFilter {
    ...
    sarStrStatus: string;
+   slaState: string;
  }
```

**Saved-filter parsing + label composition:**

```typescript
label: [
  parsed.status ? parsed.status.toUpperCase() : null,
  parsed.priority ? parsed.priority.toUpperCase() : null,
  parsed.sarStrStatus ? parsed.sarStrStatus.toUpperCase() : null,
  parsed.slaState ? parsed.slaState.toUpperCase() : null,
]
  .filter(Boolean)
  .join(' - '),
...
slaState: parsed.slaState ?? '',
```

Backwards-compatible parse: `parsed.slaState ?? ''` means older stored payloads without the field still parse cleanly.

**Applying a saved filter now applies the SLA state:**

```typescript
onSlaStateFilterChange(filter.slaState);
```

**Refactored clear-filters into a named handler:**

```typescript
const handleClearFilters = (): void => {
  setSelectedSavedFilterId('');
  onStatusFilterChange('');
  onPriorityFilterChange('');
  onSarStrStatusFilterChange('');
  onSlaStateFilterChange('');
  onSortChange('recent');
  onCaseTypeFilterChange('all');
};
```

This replaces the previous inline arrow that also called `handleSavedFilterSelect('Select a filter')` — the new version directly sets `selectedSavedFilterId` to `''` via `setSelectedSavedFilterId('')`, which is cleaner. `handleSavedFilterSelect('Select a filter')` in the old code was actually a no-op that also called `setSelectedSavedFilterId('Select a filter')` (an ID that doesn't match any saved filter), so the previous "clear filters" left a stale label in the dropdown — this refactor is a genuine bug fix.

**Saved-filter dropdown default option is now `disabled hidden`:**

```diff
- <option value="">Select a saved filter</option>
+ <option value="" disabled hidden>
+   Select a saved filter
+ </option>
```

Prevents users from re-selecting the placeholder as an active saved filter. Small UX polish.

**`useEffect` now clears the selection on fetch:**

```diff
  React.useEffect(() => {
    fetchSavedFilters();
+   setSelectedSavedFilterId('');
  }, [fetchSavedFilters]);
```

**Placeholder change:**

```diff
- placeholder="Search cases..."
+ placeholder="Search by Case ID, priority, SLA state, or keywords..."
```

**Nit:** the SLA dropdown JSX is inserted between the SAR/STR dropdown and the "Saved Filters & Save Button" row, but the enclosing `.sm:col-span-3` styling on the saved-filters row is untouched — visual layout should be unchanged for existing dropdowns. Not verified in a running UI, only by reading the diff.

---

#### 8. Frontend test files

- `CaseDashboardContainer.test.tsx`: adds `slaStateFilter: ''` to the filters fixture and `setSlaStateFilter: vi.fn()` to filterActions.
- `CaseDashboardContent.test.tsx`: same fixture updates + `onSlaStateFilterChange: vi.fn()`.
- `CaseFilters.test.tsx`: adds the prop pair to `defaultProps` and updates three `getByPlaceholderText('Search cases...')` assertions to the new placeholder.

No new frontend behavioural tests were added for the SLA dropdown itself (rendering the four options, calling `onSlaStateFilterChange`, wiring `handleClearFilters` clearing SLA, or the saved-filter round-trip through `slaState`). See [Issues and Observations](#issues-and-observations) Issue 2. *(Resolved in follow-up commit `8f88adc8` — see [Item 1](#item-1--frontend-behavioural-tests-for-sla-dropdown-clear-filters-saved-filter-round-trip).)*

[↑ Back to top](#pr-review-cms-244--fix-add-sla-state-filter-and-expand-case-search-normalization)

---

### Code Quality Analysis

#### Strengths

- **Clear inline commentary at each new branch.** The comments explaining *why* `slaState` can't go into the where clause, *why* the "Case" prefix short-circuits the other search branches, and *why* the `searchEnumMatch` normalisation is needed are non-obvious and non-redundant. They read like the kind of comment the CLAUDE.md guidance actually asks for.
- **Refactor discipline.** The search reorganisation preserves all existing branches (case_id, case_type, status, alert message, confidence_per, SAR/STR task status) intact and adds priority + SLA-state alongside them. The switch from `searchUpper.includes` to `searchEnumMatch.includes` is applied consistently.
- **`handleClearFilters` extraction fixes a latent UX bug.** The old inline `onClick` called `handleSavedFilterSelect('Select a filter')` which left the dropdown showing a stale non-empty selection; the new handler sets `selectedSavedFilterId` to `''` directly.
- **Saved-filter payload extension is backwards-compatible.** `parsed.slaState ?? ''` handles pre-existing payloads.
- **Tests exercise the new logic against the real `computeCaseSlaState` (not a mocked SLA function).** The parameterised `it.each` over `'risk' | 'at risk' | 'at-risk' | 'AT_RISK'` locks in the separator-normalisation contract precisely where the mistake is easiest to make.
- **DTO validation is in place** — `@IsEnum(SlaState)` protects the API from garbage values without any extra server-side guard.

#### Issues and Observations

##### Issue 1 — SLA-state candidate scan is unbounded

**Severity: Informational (Performance / Data Integrity — acknowledged trade-off)**

Both the SLA-state explicit filter ([case-query.service.ts:670-680](repos/case-management-system/backend/src/modules/case/services/case-query.service.ts#L670)) and the SLA-state search branch ([case-query.service.ts:517-537](repos/case-management-system/backend/src/modules/case/services/case-query.service.ts#L517)) do a full `findMany` of every case matching every other filter, then compute `sla_state` per row, then feed the matching `case_id` set back into the paginated query. Under a tenant with many cases, this pays a full scan even though the endpoint is paginated.

CodeRabbit flagged this ([review 4685149152](https://github.com/tazama-lf/case-management-system/pull/244#pullrequestreview-4685149152)); the author responded that this is an accepted trade-off because `sla_state` is time-dependent (`NOW()`-relative) and neither a persisted column with a refresh job nor a raw-SQL rewrite eliminates the fundamental need to evaluate the expression per row against the current time. CodeRabbit acknowledged the argument in a follow-up.

I concur that a persisted `sla_state` column would just replace this scan with a staleness problem, and raw SQL would relocate rather than solve it. Flagging as **Informational**, not blocking. Worth revisiting only if production data shows individual tenants growing large enough for the scan to matter.

**Not requesting a change.**

---

##### Issue 2 — Frontend has no behavioural test for the new SLA dropdown or the saved-filter round-trip

**Severity: Minor (Test Coverage)**

The backend tests are thorough. The frontend tests only extend fixtures — none actually assert that:

- The SLA dropdown renders the five options (`''`, `ON_TRACK`, `AT_RISK`, `DUE_SOON`, `BREACHED`).
- Selecting a value fires `onSlaStateFilterChange` with the expected enum literal.
- `handleClearFilters` clears the SLA state (there is no explicit assertion in `CaseFilters.test.tsx` that clicking the clear button calls `onSlaStateFilterChange('')`).
- Applying a saved filter whose payload contains `slaState: 'AT_RISK'` calls `onSlaStateFilterChange('AT_RISK')`.

None of these are difficult to add — the surrounding tests already do the equivalent for priority and status filters.

Non-blocking; the backend covers the correctness-critical logic.

---

##### Issue 3 — Style inconsistency in `caseService.ts`

**Severity: Informational (Code Quality)**

```typescript
if (query?.sarStrStatus) {
  params.append('sarStrStatus', query.sarStrStatus);
}
if (query?.slaState) params.append('slaState', query.slaState);
```

The surrounding block consistently uses braced `if` statements; the new line uses a single-line `if` without braces. Not a functional issue; would just recommend matching the surrounding style.

---

##### Issue 4 — PR title is truncated; commit body carries the tail

**Severity: Informational (Metadata)**

The title `fix: fixed the case search to incorporate new columns and added filte…` exceeds the 100-character Conventional Commits header limit and is cut off. The commit message body preserves the tail (`…rs to search SLA states`). Consider a tighter title on merge, e.g. `fix: add SLA-state filter and expand case search normalization`. Not blocking — the failing `validate-pr-title` run is an unrelated "Set up job" infra failure on a superseded workflow run; four sibling runs on the same PR passed.

[↑ Back to top](#pr-review-cms-244--fix-add-sla-state-filter-and-expand-case-search-normalization)

---

### Security Assessment

| Concern | Assessment |
|---------|-----------|
| Input validation on `slaState` | `@IsEnum(SlaState)` on the DTO — invalid values are rejected at the boundary before reaching the service. Safe. |
| Injection via search term | The search term is passed to Prisma's `contains` / equality operators (parameterised) and to safe regex `.replace()` on the JS side. No string-concatenated SQL. No new injection surface. |
| Tenant isolation | The SLA candidate scan uses `where: baseFilters` (search branch) and `where: whereClause` (filter branch) — both include the tenant filter established earlier in the method. Cross-tenant leakage risk is unchanged. |
| PII exposure | The `select` on the candidate scan is limited to `case_id`, `tenant_id`, `sla_due_at`, `sla_started_at` — no additional PII pulled into memory beyond what a normal listing already reads. |
| Auth / authorization | Not touched by this PR. |
| Regex ReDoS | `^case(?:-|_|\s)*` and `[\s\-]+` are both linear-time patterns over short search strings — no catastrophic-backtracking risk. |

No new security vulnerabilities introduced by this PR.

[↑ Back to top](#pr-review-cms-244--fix-add-sla-state-filter-and-expand-case-search-normalization)

---

### Test Coverage

**Backend:** Strong. Six new tests cover:
- SLA state explicit filter (asserts candidate lookup runs first and constrains the paginated query).
- `"Case N"` prefix search.
- Priority partial match.
- SLA state search across four separator variants (`risk`, `at risk`, `at-risk`, `AT_RISK`).
- Multi-word SLA state search (`due soon` → `DUE_SOON`).

Fixtures use realistic date offsets so `computeCaseSlaState` returns actual states rather than being mocked out — this is the right choice because the derivation is the thing under test.

**Frontend (initial commit):** Weak-to-adequate. Only fixture updates and placeholder-assertion updates. No behavioural test of:
- SLA dropdown rendering / interaction.
- Clear-filters clearing the SLA state.
- Saved-filter payload round-trip carrying `slaState`.

*(Improved in follow-up commit `8f88adc8` — see [Item 1](#item-1--frontend-behavioural-tests-for-sla-dropdown-clear-filters-saved-filter-round-trip).)*

**CI evidence:** `node-ci / check tests` passed on the PR head; test suites are green.

**PR checklist:** The PR description on GitHub is bare (no template checklist visible). Cannot assess checkbox state.

[↑ Back to top](#pr-review-cms-244--fix-add-sla-state-filter-and-expand-case-search-normalization)

---

### CodeRabbit Activity

CodeRabbit posted one actionable comment; it maps to my own [Issue 1](#issue-1--sla-state-candidate-scan-is-unbounded). Corroborated.

#### Pass 1 — SLA candidate scan performance

**Commit reviewed:** `b4f66cec42f415aaab0a1490b7d7cce34c850e08`
**Findings:** 1 actionable comment

| Finding | Severity | Status |
|---------|----------|--------|
| Unbounded in-memory SLA filtering in both branches (lines 516-537 and 666-680) — full candidate scan before pagination | 🟠 Major (per CodeRabbit) / Informational (per this review, given trade-off) | ➖ Declined by author as intentional; CodeRabbit acknowledged |

Author's response ([comment 3571202088](https://github.com/tazama-lf/case-management-system/pull/244#discussion_r3571202088), [comment 3571237460](https://github.com/tazama-lf/case-management-system/pull/244#discussion_r3571237460)):

> The scalability concern is valid, but we are intentionally accepting it as a known limitation in this PR. `sla_state` is calculated at read time using the current timestamp and tenant-specific SLA ratios, so efficiently filtering it would require either moving the calculation to raw SQL or introducing persisted derived fields with a migration and scheduled refresh process. Both options are outside the scope of this change. […] A persisted/indexed `sla_state` is not technically reliable in the current model because the value changes with time even when the case row itself is not updated. […] Computing it in raw SQL would only move the calculation from Node to the database; it would not make the predicate indexable because the result still depends on `NOW()` and multiple row fields.

CodeRabbit acknowledged and recorded a learning ([comment 3571241467](https://github.com/tazama-lf/case-management-system/pull/244#discussion_r3571241467)):

> Agreed there's no isolated fix available within this PR's scope. I won't open a follow-up issue as requested and will treat this as accepted, intended behavior.

I agree with the author's reasoning — a persisted `sla_state` column would trade this problem for a staleness/refresh-orchestration problem, and raw SQL relocates rather than solves the per-row-per-`NOW()` evaluation. This is not a merge blocker.

[↑ Back to top](#pr-review-cms-244--fix-add-sla-state-filter-and-expand-case-search-normalization)

---

### Summary and Verdict

**Verdict: Approve with minor cleanup requested**

Functionally correct, well-commented, and adequately tested on the backend. The SLA-state filter and search additions do what the PR title says, the search refactor is a proper improvement (separator normalisation, `"Case"` prefix short-circuit, priority partial match), and the frontend wiring is consistent with existing filter patterns. The one performance concern raised by CodeRabbit is a real trade-off the author has articulated a coherent justification for; I would not block on it in this PR's scope.

The main items are minor cleanups: frontend behavioural tests for the SLA dropdown / clear-filters / saved-filter round-trip, a style-consistency nit in `caseService.ts`, and a tighter PR/merge title. None of these should hold up the merge.

#### Blocking

None.

#### Non-blocking but recommended

1. **Frontend behavioural tests for the SLA dropdown, clear-filters behavior, and saved-filter round-trip** — see [Issue 2](#issue-2--frontend-has-no-behavioural-test-for-the-new-sla-dropdown-or-the-saved-filter-round-trip). The backend logic is thoroughly covered; the frontend surface is not.
2. **Match the surrounding braced-`if` style in `caseService.ts`** — one-line inconsistency, see [Issue 3](#issue-3--style-inconsistency-in-caseservicets).
3. **Tighten the PR title on merge** — currently truncated to `… added filte…`. See [Issue 4](#issue-4--pr-title-is-truncated-commit-body-carries-the-tail).

[↑ Back to top](#pr-review-cms-244--fix-add-sla-state-filter-and-expand-case-search-normalization)

---
---
---

## Follow-up Review (2026-07-13)

**Reviewed commit:** `8f88adc8b8da6c64da3bba345b6ceedb3045fad1` — *"fix: fixed the style issues and added test cases for SLA search"* (2026-07-13 15:54 UTC)
**Reviewed against:** Initial review from 2026-07-13 (3 non-blocking items on commit `b4f66cec`)
**Size of follow-up:** +72 / -1 lines across 2 files
**CI on new HEAD:** All checks green (including all `validate-pr-title` runs); PR title has been updated to `fix: add SLA-state filter and expand case search normalization`, matching the exact suggestion from Item 3.

The author addressed every non-blocking item from the initial review with a single follow-up commit. No new logic was introduced — this is a pure cleanup/test-hardening push.

### Resolution Status

#### Item 1 — Frontend behavioural tests for SLA dropdown, clear-filters, saved-filter round-trip

**Status: RESOLVED**

The follow-up adds three new tests in `frontend/src/features/cases/components/__tests__/CaseFilters.test.tsx`, one per gap flagged in the initial review:

**1a) SLA dropdown renders all five options and fires `onSlaStateFilterChange`** — [CaseFilters.test.tsx:173-193](repos/case-management-system/frontend/src/features/cases/components/__tests__/CaseFilters.test.tsx#L173):

```typescript
it('should render SLA state filter options and call onSlaStateFilterChange when changed', async () => {
  const user = userEvent.setup();
  const onSlaStateFilterChange = vi.fn();
  render(
    <CaseFilters
      {...defaultProps}
      onSlaStateFilterChange={onSlaStateFilterChange}
    />,
  );
  await user.click(screen.getByText('Filters'));
  expect(screen.getByRole('option', { name: 'All SLA States' })).toBeInTheDocument();
  expect(screen.getByRole('option', { name: 'On Track' })).toBeInTheDocument();
  expect(screen.getByRole('option', { name: 'At Risk' })).toBeInTheDocument();
  expect(screen.getByRole('option', { name: 'Due Soon' })).toBeInTheDocument();
  expect(screen.getByRole('option', { name: 'Breached' })).toBeInTheDocument();

  const slaStateSelect = screen.getByDisplayValue('All SLA States');
  await user.selectOptions(slaStateSelect, 'AT_RISK');
  expect(onSlaStateFilterChange).toHaveBeenCalledWith('AT_RISK');
});
```

Covers both the rendering surface (all five options present with the expected human-readable labels) and the change-callback wiring. This is stronger than the sketch I provided in the review comment — it asserts each option label rather than just one.

**1b) Clear button clears SLA state** — [CaseFilters.test.tsx:196-208](repos/case-management-system/frontend/src/features/cases/components/__tests__/CaseFilters.test.tsx#L196):

```typescript
it('should clear slaStateFilter when Clear button is clicked', async () => {
  const user = userEvent.setup();
  const onSlaStateFilterChange = vi.fn();
  render(
    <CaseFilters
      {...defaultProps}
      slaStateFilter="AT_RISK"
      onSlaStateFilterChange={onSlaStateFilterChange}
    />,
  );
  await user.click(screen.getByText('Clear'));
  expect(onSlaStateFilterChange).toHaveBeenCalledWith('');
});
```

Directly asserts the `handleClearFilters` path clears SLA — the behaviour I was concerned about after the clear-handler refactor.

**1c) Saved-filter round-trip carries `slaState`** — [CaseFilters.test.tsx:458-488](repos/case-management-system/frontend/src/features/cases/components/__tests__/CaseFilters.test.tsx#L458):

```typescript
it('should apply slaState from a saved filter when selected', async () => {
  mockGetFilters.mockResolvedValue([
    {
      filter_Id: 1,
      user_filters: JSON.stringify({
        sortBy: 'oldest',
        status: 'STATUS_20_IN_PROGRESS',
        priority: 'MEDIUM',
        sarStrStatus: '',
        slaState: 'AT_RISK',
      }),
    },
  ]);
  const user = userEvent.setup();
  const onSlaStateFilterChange = vi.fn();
  render(
    <CaseFilters
      {...defaultProps}
      onSlaStateFilterChange={onSlaStateFilterChange}
    />,
  );
  await user.click(screen.getByText('Filters'));
  await waitFor(() => {
    expect(screen.getByText('OLDEST - STATUS_20_IN_PROGRESS - MEDIUM - AT_RISK'))
      .toBeInTheDocument();
  });
  const savedFilterSelect = screen.getByDisplayValue('Select a saved filter');
  await user.selectOptions(savedFilterSelect, '1');
  expect(onSlaStateFilterChange).toHaveBeenCalledWith('AT_RISK');
});
```

This test asserts *both* the label composition path (`OLDEST - STATUS_20_IN_PROGRESS - MEDIUM - AT_RISK`, i.e. the new `.filter(Boolean).join(' - ')` picks up `slaState`) *and* the `parsed.slaState ?? ''` parse path that was previously entirely untested. Solid coverage of the round-trip.

All three tests use the same idiomatic patterns as the surrounding suite (`userEvent.setup()`, `screen.getByRole`, `mockGetFilters.mockResolvedValue`), so they should be maintainable alongside the rest of the file.

---

#### Item 2 — Style consistency in `caseService.ts` (braced `if`)

**Status: RESOLVED**

[caseService.ts:597-602](repos/case-management-system/frontend/src/features/cases/services/caseService.ts#L597):

```diff
- if (query?.slaState) params.append('slaState', query.slaState);
+ if (query?.slaState) {
+   params.append('slaState', query.slaState);
+ }
```

Matches the surrounding `sarStrStatus` block. Note that the pre-existing single-line `if (query?.search)` / `if (query?.page)` / `if (query?.limit)` blocks *below* this line remain unbraced — the author addressed the specific new line I flagged without expanding scope to the pre-existing style variance, which is the correct scoping call.

---

#### Item 3 — Tighten PR title before merge

**Status: RESOLVED**

The PR title was updated from `fix: fixed the case search to incorporate new columns and added filte…` (truncated at 100 chars) to `fix: add SLA-state filter and expand case search normalization` — exactly the suggestion from the review comment. All `validate-pr-title` workflow runs on the new HEAD (`8f88adc8`) pass, including the three that ran after the title edit (runs 29264274685, 29264337609, 29264354920).

The commit message headline itself (on commit `8f88adc8`) is a separate `fix: fixed the style issues and added test cases for SLA search`, which is fine — the PR title is what governs the merge commit and the conventional-commits check.

---

### New Issues Found in Updated Commits

None. The follow-up commit is entirely additive (three new tests) plus a two-line style fix. No new logic, no behavioural regressions surface risk.

I re-read the three added tests end-to-end and confirmed:
- Test 1a asserts that all four state options plus the empty option render; the option labels match the JSX in [CaseFilters.tsx](repos/case-management-system/frontend/src/features/cases/components/CaseFilters.tsx). No brittle text.
- Test 1b relies on the same `Clear` button that clears every other filter — no risk of drift with the new `handleClearFilters` refactor.
- Test 1c's mock payload includes an empty `sarStrStatus`, which correctly exercises the `.filter(Boolean)` branch (the label omits it), while the non-empty `slaState: 'AT_RISK'` is preserved. This is exactly the shape of a payload written by the current UI's save-filter flow.

### Resolution Summary

| # | Item | Status |
|---|------|--------|
| 1 | Frontend behavioural tests for the SLA dropdown, clear-filters, and saved-filter round-trip | ✅ Resolved (3 tests added in `8f88adc8`) |
| 2 | Style consistency in `caseService.ts` (braced `if`) | ✅ Resolved (`8f88adc8`) |
| 3 | Tighten PR title | ✅ Resolved (title updated to the suggested text) |

Prior CodeRabbit finding on the SLA candidate scan performance remains `➖ Declined by author as intentional` — no change on that front, no change expected.

---

### Final Verdict

**Verdict: Approved**

Every non-blocking item from the initial review has been addressed in `8f88adc8`. The frontend test coverage gap that was the most substantive of the three is now closed with three focused tests that cover the dropdown-change, clear-button, and saved-filter parse paths — including the previously untested `parsed.slaState ?? ''` branch. The style nit and PR title are both fixed. All CI checks are green on the new HEAD. The only remaining concern from the initial review (the SLA-candidate scan cost) is an accepted trade-off with a coherent rationale from the author, corroborated by CodeRabbit in the same thread.

Ready to merge.

#### Blocking

None.

#### Non-blocking

None outstanding.

[↑ Back to top](#pr-review-cms-244--fix-add-sla-state-filter-and-expand-case-search-normalization)

---

### GitHub Review Comment

`````markdown
**Approved**

Thanks for turning this around so quickly. All three non-blocking items from the earlier review are addressed in `8f88adc8`:

1. **Frontend tests for the SLA dropdown, clear-filters, and saved-filter round-trip** — the three new tests in `CaseFilters.test.tsx` cover exactly the gaps I flagged, including the previously untested `parsed.slaState ?? ''` parse path (the saved-filter test asserts both the composed label `OLDEST - STATUS_20_IN_PROGRESS - MEDIUM - AT_RISK` and the `onSlaStateFilterChange('AT_RISK')` callback). Nicely scoped.
2. **Braced `if` in `caseService.ts`** — fixed, and I appreciate that you didn't expand scope to the pre-existing single-line `if`s further down the block. That was the right call.
3. **PR title** — updated to `fix: add SLA-state filter and expand case search normalization`. `validate-pr-title` is green across the board.

No new issues in the follow-up commit — it's purely additive tests plus a two-line style fix. The SLA-candidate-scan performance thread with CodeRabbit remains the one accepted trade-off, and I still concur that it isn't a merge blocker given the `NOW()`-relative computation.

Ready to merge from my side.
`````

[↑ Back to top](#pr-review-cms-244--fix-add-sla-state-filter-and-expand-case-search-normalization)
