# Solution — Issue #217: Case Ageing Report

Track A is a small backend-only PR and merges independently. Track B is a full nine-widget rework and lands as a single coordinated backend + frontend PR after Track A is merged.

---

## Track A — Investigator scope helper + abandoned exclusion (PR 1 of 2)

All changes below are in a single file: `backend/src/modules/report/report.service.ts`.

### Step A1 — Add the shared ageing investigator-scope helper

Placed alongside the existing `applyInvestigatorScope` helper (`report.service.ts` around line 149) so both scoping functions live together.

**BEFORE** — nothing between the existing `applyInvestigatorScope` and the next method.

**AFTER** — add:

```ts
/**
 * Scope for the Case Ageing Report: an investigator's ageing view must reflect
 * work they explicitly own, not the shared claimable pool. Keeps only two arms:
 *   - cases they own (`case_owner_user_id = me`),
 *   - cases with a task assigned to them.
 * Drops the "any unassigned" and "any STATUS_02_READY_FOR_ASSIGNMENT" arms that
 * the previous inline OR clause carried — those belong on a separate unassigned
 * queue view, not summed into any one investigator's backlog.
 */
private applyAgeingInvestigatorScope(baseFilters: Prisma.CaseWhereInput, requestingUserId?: string): Prisma.CaseWhereInput {
  if (!requestingUserId) return baseFilters;

  return {
    AND: [
      baseFilters,
      {
        OR: [
          { case_owner_user_id: requestingUserId },
          { tasks: { some: { assigned_user_id: requestingUserId } } },
        ],
      },
    ],
  };
}
```

Fixes quirk 6 (investigator scope conflates my-cases with claimable pool). Uses the same `AND` / `OR` shape as the code it replaces, so any Prisma nuance carries over cleanly.

---

### Step A2 — Replace the three inline OR clauses inside `getCaseAgeing`

**BEFORE** — the shared query builder ([report.service.ts:940-965](../../../repos/case-management-system/backend/src/modules/report/report.service.ts)):

```ts
let whereClause: any;

// If requestingUserId is provided (investigator), filter to show only unassigned, ready for assignment, or assigned to them
if (filters?.requestingUserId) {
  whereClause = {
    AND: [
      baseFilters,
      {
        OR: [
          { case_owner_user_id: filters.requestingUserId },
          {
            tasks: {
              some: {
                assigned_user_id: filters.requestingUserId,
              },
            },
          },
          { case_owner_user_id: null },
          { status: 'STATUS_02_READY_FOR_ASSIGNMENT' },
        ],
      },
    ],
  };
} else {
  whereClause = baseFilters;
}
```

**AFTER**:

```ts
const whereClause = this.applyAgeingInvestigatorScope(baseFilters, filters?.requestingUserId);
```

**BEFORE** — the `caseTypeResolution` builder ([report.service.ts:1060-1085](../../../repos/case-management-system/backend/src/modules/report/report.service.ts)):

```ts
let caseTypeWhereClause: any;

// Apply the same user filtering logic
if (filters?.requestingUserId) {
  caseTypeWhereClause = {
    AND: [
      caseTypeBaseFilters,
      {
        OR: [
          { case_owner_user_id: filters.requestingUserId },
          {
            tasks: {
              some: {
                assigned_user_id: filters.requestingUserId,
              },
            },
          },
          { case_owner_user_id: null },
          { status: 'STATUS_02_READY_FOR_ASSIGNMENT' },
        ],
      },
    ],
  };
} else {
  caseTypeWhereClause = caseTypeBaseFilters;
}
```

**AFTER**:

```ts
const caseTypeWhereClause = this.applyAgeingInvestigatorScope(caseTypeBaseFilters, filters?.requestingUserId);
```

**BEFORE** — the `resolutionTrend` builder ([report.service.ts:1128-1153](../../../repos/case-management-system/backend/src/modules/report/report.service.ts)):

```ts
let recentClosedWhereClause: any;

// Apply the same user filtering logic
if (filters?.requestingUserId) {
  recentClosedWhereClause = {
    AND: [
      recentClosedBaseFilters,
      {
        OR: [
          { case_owner_user_id: filters.requestingUserId },
          {
            tasks: {
              some: {
                assigned_user_id: filters.requestingUserId,
              },
            },
          },
          { case_owner_user_id: null },
          { status: 'STATUS_02_READY_FOR_ASSIGNMENT' },
        ],
      },
    ],
  };
} else {
  recentClosedWhereClause = recentClosedBaseFilters;
}
```

**AFTER**:

```ts
const recentClosedWhereClause = this.applyAgeingInvestigatorScope(recentClosedBaseFilters, filters?.requestingUserId);
```

The three sites are now a single-line call to the shared helper. This retires the copy-paste and delivers quirk 6's fix in one place.

---

### Step A3 — Exclude `STATUS_99_ABANDONED` at the query level

The recent-closed trend query already filters to `CLOSED_STATUSES` (which does not include `99`), so no change is required there. The change is on the shared and per-type query base filters.

**BEFORE** — `baseFilters` construction for the shared query ([report.service.ts:933-938](../../../repos/case-management-system/backend/src/modules/report/report.service.ts)):

```ts
let baseFilters: any = {};

if (filters?.tenantId) {
  baseFilters.tenant_id = filters.tenantId;
}
baseFilters = ReportsService.withNonContainerCaseFilter(baseFilters);
```

**AFTER**:

```ts
let baseFilters: any = {
  status: { not: CaseStatus.STATUS_99_ABANDONED },
};

if (filters?.tenantId) {
  baseFilters.tenant_id = filters.tenantId;
}
baseFilters = ReportsService.withNonContainerCaseFilter(baseFilters);
```

**BEFORE** — `caseTypeBaseFilters` construction for the per-type query ([report.service.ts:1048-1058](../../../repos/case-management-system/backend/src/modules/report/report.service.ts)):

```ts
let caseTypeBaseFilters: any = {
  status: {
    in: ReportsService.CLOSED_STATUSES,
  },
  case_type: type,
};

if (filters?.tenantId) {
  caseTypeBaseFilters.tenant_id = filters.tenantId;
}
caseTypeBaseFilters = ReportsService.withNonContainerCaseFilter(caseTypeBaseFilters);
```

**AFTER** — no change required. `status: { in: CLOSED_STATUSES }` already excludes `STATUS_99_ABANDONED` because `99` is not in `CLOSED_STATUSES`. Left as-is.

Fixes quirk 9 (abandoned cases pollute the ageing backlog). The shared query now returns only non-abandoned in-scope cases; every widget fed by the shared query (Avg Case Age, over-15/30, by-status bar, distribution pie, details table) is corrected in one place.

---

### Impact of Track A alone

- Investigator's personal ageing metrics stop double-counting the tenant-wide claimable pool (unassigned + ready-for-assignment).
- Abandoned draft-only cases stop appearing in every ageing widget for both supervisor and investigator callers.
- Three copy-pasted OR clauses become one shared helper — future scope refinements land in a single place.
- Supervisor / admin output is otherwise unchanged.

---

## Track B — Full nine-widget rework (PR 2 of 2)

Track B is a large coordinated backend + frontend PR. It lands **after** Track A is merged.

### Step B1 — Split `getCaseAgeing` into two lifecycle-scoped datasets

**File:** `backend/src/modules/report/report.service.ts`

Introduce two private builders:

- `private buildOpenSnapshotWhere(filters): Prisma.CaseWhereInput` — the open-population dataset. `tenant_id` [+ `withNonContainerCaseFilter`] [+ investigator scope via the Track A helper] [+ `status: { notIn: [...CLOSED_STATUSES, STATUS_99_ABANDONED] }`]. No `created_at` window.
- `private buildClosedInWindowWhere(filters, dateRange): Prisma.CaseWhereInput` — the closed-in-window dataset. Same base plus `status: { in: CLOSED_STATUSES }` and `updated_at: { gte: windowStart, lte: windowEnd }` derived from `dateRange`.

`getCaseAgeing` then orchestrates:

- Avg Case Age (3.1), over-15/30 cards (3.3/3.4), by-status bar (3.5), distribution pie (3.6), details table (3.9) → run against the open-snapshot dataset.
- Avg Resolution Time (3.2), resolution trend (3.7), case-type resolution (3.8) → run against the closed-in-window dataset.

### Step B2 — Shared age-band helper with Hamilton rounding

**File:** `backend/src/modules/report/report.service.ts`

```ts
private static readonly AGE_BANDS = [
  { key: 'age0to7',   label: '0-7 days',   min: 0,  max: 7,        color: '#10b981' },
  { key: 'age8to15',  label: '8-15 days',  min: 8,  max: 15,       color: '#f59e0b' },
  { key: 'age16to30', label: '16-30 days', min: 16, max: 30,       color: '#ef4444' },
  { key: 'age30Plus', label: '30+ days',   min: 31, max: Infinity, color: '#7c2d12' },
] as const;

private bucketAge(ageDays: number): typeof ReportsService.AGE_BANDS[number]['key'] {
  const band = ReportsService.AGE_BANDS.find((b) => ageDays >= b.min && ageDays <= b.max);
  return (band ?? ReportsService.AGE_BANDS[0]).key;
}

private hamiltonPercentages(counts: number[]): number[] {
  const total = counts.reduce((s, n) => s + n, 0);
  if (total === 0) return counts.map(() => 0);
  const exact = counts.map((n) => (n / total) * 100);
  const floor = exact.map((x) => Math.floor(x));
  let remainder = 100 - floor.reduce((s, n) => s + n, 0);
  const order = exact.map((x, i) => ({ i, frac: x - Math.floor(x) })).sort((a, b) => b.frac - a.frac);
  const out = floor.slice();
  for (let k = 0; k < order.length && remainder > 0; k++, remainder--) out[order[k].i]++;
  return out;
}
```

Both the by-status bar and the distribution pie consume `AGE_BANDS`; the pie consumes `hamiltonPercentages`. Boundaries `>= min && <= max` settle the `>15` vs `>=30` conflict from Track A's baseline.

### Step B3 — Monthly `groupBy` for `resolutionTrend`

Replace the current per-close-day emission with a backend aggregation. Compute median + IQR from raw `resolutionDays` per bucket (no average-of-averages). Payload shape:

```ts
resolutionTrend: Array<{
  bucketStart: string;   // ISO date, first day of month
  median: number;
  p25: number;
  p75: number;
  n: number;
}>;
```

Emit one row per month across the window (empty months included, `n = 0`) so the frontend continuous temporal axis reads a real rate of change.

### Step B4 — Single grouped case-type aggregation

Replace the per-type `Promise.all` with a single `case.groupBy({ by: ['case_type'], where: closedInWindowWhere })` and aggregate `resolutionDays` per group. Retire the `FRAUD_AND_AML` filter — the container is already dropped by `withNonContainerCaseFilter`.

### Step B5 — `null` empty averages

**File:** `backend/src/modules/report/report.service.ts`

```ts
const avgCaseAge = casesWithAge.length > 0
  ? Math.round(casesWithAge.reduce((s, c) => s + c.ageDays, 0) / casesWithAge.length)
  : null;

const avgResolutionTime = closedCasesWithTimes.length > 0
  ? Math.round(closedCasesWithTimes.reduce((s, c) => s + resolutionDays(c), 0) / closedCasesWithTimes.length)
  : null;
```

**File:** `frontend/src/features/reports/types/reports.types.ts`

```ts
export interface CaseAgeingStats {
  avgCaseAge: number | null;
  avgResolutionTime: number | null;
  casesOver15Days: number;
  casesOver30Days: number;
}
```

**File:** `frontend/src/features/reports/components/CaseAgeingStatsCards.tsx`

```tsx
<StatCard days={stats.avgCaseAge ?? 'N/A'} label="Live backlog" ... />
<StatCard days={stats.avgResolutionTime ?? 'N/A'} label="Closed throughput" ... />
```

### Step B6 — Stop pre-formatting `status` / `createdDate`; drop duplicated `investigator`

Backend:

```ts
const caseDetails = casesWithAge.map((c) => ({
  caseId: c.case_id,
  type: c.case_type ?? 'NONE',
  status: c.status,                              // raw enum, no formatStatusName
  createdDate: c.created_at.toISOString(),       // ISO 8601, no toLocaleDateString
  ageDays: c.ageDays,
  priority: c.priority,
  ownerUserId: c.case_owner_user_id ?? null,     // one field, one nullable id
}));
```

Frontend `CaseAgeingTable.tsx`:
- Format `status` with the shared code-prefixed title-case helper (`02 Ready For Assignment`).
- Format `createdDate` with the client's date library.
- Drop the raw "User ID" column; keep the resolved "Investigator" column.
- Resolve `'Unassigned'` on the client via `getAssigneeFullName`.

### Step B7 — Wire `dateRange` through the controller

**File:** `backend/src/modules/report/report.controller.ts`

`dateRange` already reaches `getCaseAgeing`; the service now uses it to build the closed-in-window `updated_at` bound. Swagger schema updates for the nullable averages.

### Step B8-B13 — Frontend widget rework

See `impact-217.md` Frontend Code Impact table for the per-file breakdown. Each widget consumes the new payload shape and applies the presentation contract (horizontal bar, seeded status set, code-prefixed title-case labels, deterministic band order, continuous temporal axis, Hamilton rounding).

### Step B14 — Test cases

See the "Test cases" section below.

---

## Data migration

None. Track A and Track B are both query-shape and presentation-only. No schema change, no backfill.

---

## Test cases

### Unit tests (backend, add these)

`__tests__/report.service.spec.ts` — new tests under a `getCaseAgeing` describe block:

```ts
describe('getCaseAgeing — Track A', () => {
  it('investigator scope keeps only owned + task-assigned arms', async () => {
    // Seed 4 cases:
    //   c1: owned by U1
    //   c2: task assigned to U1
    //   c3: unassigned (case_owner_user_id null), owned by nobody
    //   c4: STATUS_02_READY_FOR_ASSIGNMENT owned by U2
    // Call getCaseAgeing with requestingUserId=U1.
    // Expect caseDetails ids == [c1, c2]. c3 and c4 excluded.
  });

  it('supervisor scope (no requestingUserId) returns tenant-wide', async () => {
    // Same seed. requestingUserId=undefined => c1..c4 all present (bar abandoned).
  });

  it('excludes STATUS_99_ABANDONED from the shared dataset', async () => {
    // Seed one STATUS_99_ABANDONED and one STATUS_20_IN_PROGRESS in the tenant.
    // Expect avgCaseAge / caseDetails to reflect only STATUS_20_IN_PROGRESS.
  });

  it('excludes STATUS_99_ABANDONED from per-type resolution', async () => {
    // STATUS_99 is not in CLOSED_STATUSES so this is regression coverage —
    // ensure no future change accidentally lets it in.
  });

  it('resolutionTrend still restricts to CLOSED_STATUSES (regression)', async () => {
    // Seed a STATUS_20 case with updated_at in-window. Expect resolutionTrend
    // does not contain it.
  });
});
```

### Integration / E2E scenarios (Track A)

1. Log in as an investigator with owned + task-assigned cases plus tenant-wide unassigned cases. Open the Case Ageing Report. **Expect** the details table to list only owned + task-assigned cases; unassigned rows are absent.
2. Log in as a supervisor. Seed one `STATUS_99_ABANDONED` case in the tenant. Open the Case Ageing Report. **Expect** no `99 Abandoned` bar in the by-status chart and no abandoned row in the details table.
3. Compare Avg Case Age before and after seeding a `STATUS_99_ABANDONED` case in the tenant on Track A. **Expect** the two numbers to be identical (abandoned excluded).
4. Compare an investigator's Avg Case Age on old dev vs Track A given a tenant that has any unassigned cases. **Expect** the Track A number to be **lower** (claimable arms no longer count into the investigator's personal mean).

### Data migration validation SQL

None.

### Manual / UAT checks

| # | Scenario | Steps | Expected Result |
| --- | --- | --- | --- |
| 1 | Investigator scope | Log in as investigator U1. Open the Case Ageing Report. | Only U1's owned + task-assigned cases in the details table; no unassigned / no ready-for-assignment cases. |
| 2 | Supervisor scope | Log in as supervisor. Open the Case Ageing Report. | All tenant cases in the details table, except abandoned. |
| 3 | Abandoned exclusion | Ensure a `STATUS_99_ABANDONED` case exists in the tenant. Open the report as either role. | No `99 Abandoned` bar in the by-status chart; no abandoned row in the details table. |
| 4 | Trend regression | Same seed as above. Look at the Average Resolution Time Trend. | Behaviour unchanged (this widget already filtered by `CLOSED_STATUSES`). |
| 5 | Per-type regression | Same seed. Look at the Case Type Resolution bar. | Behaviour unchanged (this widget already filters by `CLOSED_STATUSES`). |

---

## Overall Impact of the Fix

| Area | Before (dev today) | After Track A | After Track B |
| --- | --- | --- | --- |
| Investigator scope inside `getCaseAgeing` | Inline 4-arm OR clause copy-pasted three times; conflates ownership with claimable pool | Single `applyAgeingInvestigatorScope` helper; two ownership arms only | Same as Track A; refinements per-widget flow through the helper |
| `STATUS_99_ABANDONED` in ageing scope | Included in every widget fed by the shared query | Excluded at the query level | Excluded at the query level (retained) |
| Container `FRAUD_AND_AML` | Already excluded via `withNonContainerCaseFilter` (issue #214) | Unchanged | Unchanged |
| `dateRange` | Ignored by every widget on the page | Ignored (deferred) | Bound to the closed-in-window dataset (Avg Resolution Time, trend, type bar) |
| Bucket boundaries | `>` at 15 days, `>=` at 30 days; recomputed independently in three places | Unchanged (deferred) | Single shared band definition, one boundary convention |
| Stat cards `>15` / `>30` | Overlap (30+ ⊂ 15+) | Unchanged (deferred) | Non-overlapping tiers or explicitly cumulative |
| Empty averages | `0` for empty populations | `0` (deferred) | `null` on wire; `N/A` in card |
| `resolutionTrend` | Per-close-day scatter over fixed 6 months | Unchanged (deferred) | Monthly median + IQR band on a continuous axis |
| `caseTypeResolution` | N per-type queries | Unchanged (deferred) | Single grouped aggregation |
| Status label | ALL-CAPS via `formatStatusName` | Unchanged (deferred) | Code-prefixed title-case (`02 Ready For Assignment`) |
| Status axis | Data-driven — statuses with zero rows silently disappear | Unchanged (deferred) | Seeded from the canonical open-status set, deterministic order |
| Duplicated owner projection in the table | `userId` + `investigator` from the same field | Unchanged (deferred) | Single nullable `ownerUserId`; display resolved on client |
| Pre-formatted `createdDate` | `en-US` locale string on the wire | Unchanged (deferred) | ISO 8601 on the wire; client formats |

---

## Fix Summary

The Case Ageing Report today mixes two different questions — a live snapshot of open cases and a throughput view of closed cases — into a single unbounded dataset, then applies presentation contracts (ALL-CAPS labels, per-day averages, `0`-for-empty means) that make each widget answer for a different implicit period and a different implicit population. On top of that dataset, an investigator caller sees not only their own work but the entire tenant's claimable pool (unassigned + ready-for-assignment cases), and dead abandoned drafts age forever without ever counting as resolved. The issue catalogues ten cross-cutting quirks that all fall out of that shared foundation.

Track A is deliberately surgical. It removes the two dataset-level polluters that flow into every widget on the page: the copy-pasted four-arm investigator OR clause (replaced with a two-arm helper — cases you own or have a task on) and abandoned cases (excluded at the query level). Both fixes land in a single backend file, need no schema change, and need no frontend change. The investigator's personal ageing view stops averaging the shared claimable queue into their own numbers, and every ageing widget stops presenting dead drafts as live work.

Track B is the rest of the story — splitting the report into an open-snapshot dataset and a closed-in-window dataset, wiring `dateRange` to the closed dataset, introducing a single shared age-band definition, replacing the per-close-day "trend" with a monthly median and IQR, using a grouped case-type aggregation, returning `null` for empty averages and rendering `N/A` on the cards, stopping backend pre-formatting of status and date strings, and seeding the by-status axis from the canonical open-status set so it stops changing shape from one request to the next. Track B is large enough to ship as a single coordinated backend + frontend PR.

Container / `STATUS_84_COMPLETED` retirement is out of scope for this issue — `dev` already applies `withNonContainerCaseFilter` inside `getCaseAgeing`, and the container model itself is being handled under issue #214.
