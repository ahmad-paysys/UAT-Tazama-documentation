# PR Review: CMS #241 — feat: Paysys/Add Rule properties inside Typology in Visualization

## Table of Contents

- [Initial Review (2026-07-13)](#initial-review-2026-07-13)
  - [Overview](#overview)
  - [What Changed (Detailed)](#what-changed-detailed)
  - [Code Quality Analysis](#code-quality-analysis)
  - [Security Assessment](#security-assessment)
  - [Test Coverage](#test-coverage)
  - [CodeRabbit Activity — Pass 1](#coderabbit-activity--pass-1)
  - [Summary and Verdict (initial)](#summary-and-verdict-initial)
- [Follow-up Review (2026-07-13)](#follow-up-review-2026-07-13)
  - [Resolution Status — Prior Items](#resolution-status--prior-items)
  - [New Issues Found in Updated Commits](#new-issues-found-in-updated-commits)
  - [CodeRabbit Activity — Passes 2 & 3](#coderabbit-activity--passes-2--3)
  - [Test Coverage (updated)](#test-coverage-updated)
  - [Updated Verdict](#updated-verdict)
- [Final Review (2026-07-14)](#final-review-2026-07-14)
  - [Resolution Status — All Outstanding Items](#resolution-status--all-outstanding-items)
  - [New Issues Found in Final Commit](#new-issues-found-in-final-commit)
  - [Final Verdict](#final-verdict)
  - [GitHub Review Comment](#github-review-comment)

---
---
---

## Initial Review (2026-07-13)

**Repo:** tazama-lf/case-management-system
**Branch:** `paysys/addRulesPropsInTypology` → `dev`
**Author:** MAdeel95 (Muhammad Adeel)
**Reviewed commit:** `05ab1c3282a5f8a4158ada40121a69d70dc0faab`
**Label:** _none at the time — `enhancement` added later_
**Size (initial):** +37 / -281 lines across 6 files
**State:** OPEN
**Existing approvals:** none

### Overview

The PR enriches the Alert Navigator visualization so each triggered rule inside a typology can display its description and matched-band reason. This is a straight-through change: the lakehouse SQL aggregation gathers new columns (`rule_desc`, `matched_band_reason`), the backend mapping forwards them, the frontend `RuleDetailDto` accepts them, and `AlertNavigatorTab.tsx` renders them inside the per-rule details block. It also removes a verbose `loggerService.log` line in `AlertService.getTransactionHistory` that stringified full transaction payloads.

Targets `dev` — correct for this repo's flow.

CI status at time of initial review: three failing checks —
- `conventional-commits / validate-pr-title` (PR title `Paysys/Add Rule properties inside Typology in Visualization` didn't match Conventional Commits format)
- `node-ci / check style` (lint failures from trailing whitespace and stray blank lines)
- `node-ci / check tests`

| File | Nature of Change |
|------|------------------|
| `backend/src/modules/gold-lakehouse/alerts-lakehouse.service.ts` | SQL `json_build_object` gains 3 fields; `mappedRules` forwards `ruleDesc`/`matched_band_reason`/`matched_rule_reason`. |
| `backend/src/modules/gold-lakehouse/types/raw-rule-row.types.ts` | Adds `rule_desc`, `matched_band_reason`, `matched_rule_reason`, and `band_reasons_with_sub_rule_refs_json` (unused by the mapper). |
| `backend/src/modules/alert/alert.service.ts` | Removes a debug `loggerService.log` line that serialized full transaction data. |
| `frontend/.../alertnavigator/types/index.ts` | Extends `RuleDetailDto` with 8 optional fields; only 3 are actually used. |
| `frontend/.../alertnavigator/AlertNavigatorTab.tsx` | Renders `ruleDesc` and `matched_band_reason` under each rule; removes the `if (loading) …` render branch. |
| `backend/package-lock.json` | 133 `resolved` URLs and matching `integrity` SHAs deleted across dev sub-trees. |

[↑ Back to top](#pr-review-cms-241--feat-paysysadd-rule-properties-inside-typology-in-visualization)

---

### What Changed (Detailed)

Full initial-round diff analysis was captured in the previous version of this review. The salient points, kept for follow-up traceability:

1. **`alerts-lakehouse.service.ts`** — SQL added `rule_desc`, `matched_band_reason`, `matched_rule_reason`; mapping extracted into `const mappedRules`; introduced trailing whitespace and camelCase / snake_case mix.
2. **`raw-rule-row.types.ts`** — added three used fields and one unused (`band_reasons_with_sub_rule_refs_json`); trailing whitespace on the new line.
3. **`alert.service.ts`** — dropped `loggerService.log(... JSON.stringify(transactionData) ...)`. Legitimate PII-scrubbing.
4. **`types/index.ts`** — `RuleDetailDto` gained 8 optional fields (only 3 emitted, only 2 rendered). `ruleDesc?: string` did not include `null`.
5. **`AlertNavigatorTab.tsx`** — added guarded render blocks for `ruleDesc` and `matched_band_reason`; **removed the `if (loading)` early-return** while leaving the `loading` state wired up. Stray blank lines introduced.
6. **`backend/package-lock.json`** — 133 `resolved`/`integrity` pairs removed (dev sub-trees).

[↑ Back to top](#pr-review-cms-241--feat-paysysadd-rule-properties-inside-typology-in-visualization)

---

### Code Quality Analysis

#### Strengths

- Backend/frontend contract updated together in one PR.
- `loggerService.log(... transactionData ...)` removal is a genuine privacy improvement.
- Render-side guards use `!= null` (both `null` and `undefined`).

#### Issues and Observations

Nine issues were identified in the initial round. Their statuses are tracked in the [Follow-up section](#resolution-status--prior-items):

1. **Major** — "No data available" flashes during fetch (loading branch removed).
2. **Minor** — `RuleDetailDto.ruleDesc` type doesn't include `null`.
3. **Minor** — Wire-format naming mixes camelCase and snake_case.
4. **Minor** — `band_reasons_with_sub_rule_refs_json` added to `RawRuleRow` but never selected.
5. **Minor** — 5 unused optional fields on `RuleDetailDto`.
6. **Minor** — Trailing whitespace / stray blank lines fail `check style`.
7. **Major** — 133 `resolved`/`integrity` entries stripped from `backend/package-lock.json`.
8. **Minor** — PR title fails Conventional Commits.
9. **Major** — `node-ci / check tests` failing.

[↑ Back to top](#pr-review-cms-241--feat-paysysadd-rule-properties-inside-typology-in-visualization)

---

### Security Assessment

| Concern | Assessment |
|---------|-----------|
| Sensitive data in logs | **Improved.** Transaction payload no longer serialized to logs. |
| SQL injection | No change — new fields are static column names on `alert_navigator_rules`. |
| XSS via new fields | Rendered as JSX text children — React auto-escapes. Safe. |
| AuthZ / tenant scoping | Unchanged. |
| Supply-chain integrity | **Worsened** — see Issue 7. |

[↑ Back to top](#pr-review-cms-241--feat-paysysadd-rule-properties-inside-typology-in-visualization)

---

### Test Coverage

Initial round shipped no new tests. `node-ci / check tests` was red.

[↑ Back to top](#pr-review-cms-241--feat-paysysadd-rule-properties-inside-typology-in-visualization)

---

### CodeRabbit Activity — Pass 1

**Commit reviewed:** `05ab1c32`
**Findings:** 1 actionable Major, 1 inline Minor, 2 nitpicks

| Finding | Severity | Status at initial review |
|---------|----------|--------------------------|
| Loading branch removed → "No data available" flashes | Major | ❌ Not resolved (corroborated as Issue 1) |
| `ruleDesc?: string` should be `string \| null` | Minor | ❌ Not resolved (Issue 2) |
| camelCase / snake_case wire-format inconsistency | Trivial | ❌ Not resolved (Issue 3, raised to Minor) |
| `loading` state is dead if branch stays removed | Trivial | ❌ Not resolved (Issue 1) |

[↑ Back to top](#pr-review-cms-241--feat-paysysadd-rule-properties-inside-typology-in-visualization)

---

### Summary and Verdict (initial)

**Verdict: Changes Requested** — Blocking items 1, 6, 7, 8, 9. Non-blocking: 2, 3, 4, 5.

[↑ Back to top](#pr-review-cms-241--feat-paysysadd-rule-properties-inside-typology-in-visualization)

---
---
---

## Follow-up Review (2026-07-13)

**Reviewed commit:** `6af19f186d6aeaea2a33487afb271643cdc733ca` (verified against `gh pr view --json headRefOid` at the time)
**Reviewed against:** Changes Requested on `05ab1c32` by `ahmad-paysys` (2026-07-13 11:03 UTC)
**Developer response:** No written response on the PR — resolution is via new commits only. Two new heads landed between rounds: `ac86c1c1` (test suites + edits, 2026-07-13 ~13:57 UTC) and `6af19f18` (formatting touch-up, 2026-07-13 ~14:58 UTC).

**Size after follow-up commits:** +685 / -305 lines across 9 files (up from +37/-281 across 6; three new files are large test suites).

**New files touched since prior round:**

| File | Nature of Change |
|------|------------------|
| `backend/test/alerts-lakehouse.service.spec.ts` | Adds a `rule_desc and matched_band_reason columns` describe block with 3 tests: present values pass through, null values pass through, and flow-processor rule doesn't leak into `mappedRules`. |
| `frontend/.../__tests__/AlertNavigatorTab.test.tsx` | Rewritten: adds shared `baseAlertMetadata` / `buildTypology` / `buildResponse` fixtures, expand/collapse behaviour tests, rule-detail rendering tests (present/null/undefined/one-of-two/multi-rule cases), flow-processor banner tests, `alertMetadata` fallback tests, score-color threshold tests, and a statistics-summary test. 481 net-added lines. |
| `frontend/.../__tests__/alertNavigatorService.test.ts` | Adds 7 tests: empty typologies, per-typology independent rule parsing, preservation of other response fields, URL construction, error propagation on network failure, error propagation on malformed rules JSON. |

**CI status on `6af19f18`:**
- ✅ `node-ci / check tests` — green
- ✅ `conventional-commits / validate-pr-title` — green
- ✅ `node-ci / run build`, `dco-check`, `dependency-review`, `gpg-verify`, `njsscan`, `nodejsscan`, `CodeQL`, `Analyze (actions)`, `Analyze (javascript-typescript)`, `encoding-check`, `dockerfile-linter`
- ❌ `node-ci / check style` — still **FAILING**

### Resolution Status — Prior Items

Nine items from the initial round were tracked. Detailed evidence was captured in the previous version of this review file; the final Item 6 (whitespace) and Item 7 (lockfile) remained open going into the third commit. Follow-up summary at end of round:

| # | Item | Status at end of follow-up |
|---|------|--------|
| 1 | "No data available" flashes during fetch | ✅ Resolved (loading branch restored in `6af19f18`) |
| 2 | `RuleDetailDto.ruleDesc` type widening | ✅ Resolved (renamed to `rule_desc: string \| null`) |
| 3 | Wire-format naming inconsistency | ➖ Preserved by author (no explicit reply; downgraded to observation) |
| 4 | Phantom `band_reasons_with_sub_rule_refs_json` | ⚠️ Partial — removed from `RuleDetailDto`, still on `RawRuleRow` |
| 5 | Five unused `RuleDetailDto` fields | ✅ Resolved |
| 6 | `check style` failing (trailing whitespace) | ❌ Not resolved — still red on `6af19f18` |
| 7 | `backend/package-lock.json` — 133 integrity entries stripped | ❌ Not resolved — count 81 vs. 214 on `origin/dev` |
| 8 | PR title fails Conventional Commits | ✅ Resolved |
| 9 | `check tests` failing | ✅ Resolved (green + substantial new coverage added) |

[↑ Back to top](#pr-review-cms-241--feat-paysysadd-rule-properties-inside-typology-in-visualization)

---

### New Issues Found in Updated Commits

Three new observations were raised in follow-up:

- **F1** — `check style` still failing (Major, CI blocker; same root cause as Item 6).
- **F2** — Restored `if (loading)` block indented with three leading spaces (Minor).
- **F3** — `AlertNavigatorTab.test.tsx:487-488` uses a dead-guard + `parentElement` DOM walk (Informational).

[↑ Back to top](#pr-review-cms-241--feat-paysysadd-rule-properties-inside-typology-in-visualization)

---

### CodeRabbit Activity — Passes 2 & 3

**Pass 2** (`ac86c1c1`): 3 actionable + 1 nitpick — trailing whitespace in `alerts-lakehouse.service.ts:182-184` (Critical), trailing whitespace on `raw-rule-row.types.ts:10` (Minor), placeholder test file needing restoration (Major, resolved in `6af19f18`), naming nitpick (Trivial).

**Pass 3** (`6af19f18`): 0 actionable, 1 nitpick — simplify the statistics-summary DOM walk in `AlertNavigatorTab.test.tsx:487-488` (Trivial). Pass 3 also reposted the `raw-rule-row.types.ts:10` trailing-whitespace concern under the new commit, confirming it was still unresolved.

[↑ Back to top](#pr-review-cms-241--feat-paysysadd-rule-properties-inside-typology-in-visualization)

---

### Test Coverage (updated)

Substantial new coverage added: backend service (rule_desc/matched_band_reason pass-through + null + flow-processor exclusion), frontend service (empty/multi-typology/URL/error paths), and frontend component (expand/collapse, rule-detail rendering across value states, flow-processor banner, alertMetadata fallbacks, score-color thresholds, statistics summary). `node-ci / check tests` green.

[↑ Back to top](#pr-review-cms-241--feat-paysysadd-rule-properties-inside-typology-in-visualization)

---

### Updated Verdict

**Verdict at end of follow-up: Changes Requested** — Item 6 (whitespace / `check style` red) and Item 7 (lockfile integrity strip) remained blocking. Items 3, 4, F3, F2 were non-blocking recommendations.

[↑ Back to top](#pr-review-cms-241--feat-paysysadd-rule-properties-inside-typology-in-visualization)

---
---
---

## Final Review (2026-07-14)

**Reviewed commit:** `021a9b3f4409d3c5cca3e180d4c47cca54a55323` — *"fix white-spaces"* (2026-07-13 19:58 UTC, committed 2026-07-14 in local TZ)
**Reviewed against:** Changes Requested on `6af19f18` by `ahmad-paysys` (2026-07-13 15:50 UTC, [issue comment](https://github.com/tazama-lf/case-management-system/pull/241#issuecomment-))
**Developer response:** No written comment. Resolution is via a single new commit — `021a9b3f` "fix white-spaces". Head SHA verified against `gh pr view 241 --json headRefOid` — matches `021a9b3f`.

**Size on final head:** +684 / -413 lines across 9 files. **All CI checks green** on `021a9b3f`, including `node-ci / check style` (previously red).

**What the final commit changed** (`git show 021a9b3f --stat`):

| File | Δ |
|------|---|
| `backend/package-lock.json` | −106 lines (further `resolved`/`integrity` strips) |
| `backend/src/modules/gold-lakehouse/alerts-lakehouse.service.ts` | ±18 lines (whitespace, added rationale comment, indent fixes) |
| `backend/src/modules/gold-lakehouse/types/raw-rule-row.types.ts` | ±5 lines (dropped `band_reasons_with_sub_rule_refs_json`, whitespace fixed) |
| `frontend/.../__tests__/AlertNavigatorTab.test.tsx` | ±6 lines (F3 fix — statistics DOM walk rewritten per CodeRabbit) |
| `frontend/.../alertnavigator/AlertNavigatorTab.tsx` | ±13 lines (F2 fix — `if (loading)` reindented; stray blank lines removed; band-reason JSX cleaned) |
| `frontend/.../alertnavigator/types/index.ts` | ±5 lines (trailing whitespace + stray blank lines removed; rationale comment added) |

### Resolution Status — All Outstanding Items

#### Item 3 — Wire-format naming inconsistency (was ➖ Preserved by author)

**Status: RESOLVED (via documenting comment)**

The mixed shape is preserved, but the author now documents the intent inline. In `alerts-lakehouse.service.ts:176`:

```ts
// rule_desc and matched_band_reason intentionally keep snake_case: they mirror the underlying SQL column names
const mappedRules = triggeredRulesData.map((r) => ({
  ruleId: r.rule_id,
  ruleWeight: r.rule_weight,
  subRef: r.rule_sub_ref,
  independentVariable: r.rule_independent_variable,
  rule_desc: r.rule_desc,
  matched_band_reason: r.matched_band_reason,
}));
```

Mirroring comment in `frontend/.../alertnavigator/types/index.ts:7`:

```ts
// rule_desc and matched_band_reason intentionally keep snake_case: they mirror the underlying SQL column names
rule_desc?: string | null;
matched_band_reason?: string | null;
```

Exactly the resolution suggested in the previous follow-up ("if deliberate, worth a one-line comment on the DTO stating the mix is intentional"). Closed.

#### Item 4 — Phantom `band_reasons_with_sub_rule_refs_json` on `RawRuleRow` (was ⚠️ Partial)

**Status: RESOLVED**

Removed from the type. Current `RawRuleRow` (`backend/src/modules/gold-lakehouse/types/raw-rule-row.types.ts`):

```ts
export interface RawRuleRow {
  rule_id: string | null;
  rule_weight: number | null;
  rule_independent_variable: unknown;
  rule_sub_ref: string | null;
  rule_processing_time_ms: number | null;
  rule_tenant_id: string | null;
  rule_desc: string | null;
  matched_band_reason: string | null;
}
```

No more unused field.

#### Item 6 — `check style` failing (trailing whitespace) (was ❌ Not resolved / Major)

**Status: RESOLVED**

`node-ci / check style` is **green** on `021a9b3f` (CI: [job 86920793139 — SUCCESS](https://github.com/tazama-lf/case-management-system/actions/runs/29280672257/job/86920793139)). Direct inspection of the diff confirms the trailing spaces flagged in every prior round are gone:

- `alerts-lakehouse.service.ts:53` — the whitespace-only line inside `json_build_object` was removed and `'matched_band_reason', anr.matched_band_reason` is now aligned with the sibling columns.
- `alerts-lakehouse.service.ts:176-181` — trailing whitespace after `rule_desc: r.rule_desc,` and `matched_band_reason: r.matched_band_reason` gone; indentation normalized.
- `raw-rule-row.types.ts:7-8` — trailing spaces after both new fields removed.
- `AlertNavigatorTab.tsx` — the stray blank line after `setData(result);`, the stray blank line inside the rule-map iteration, and the trailing whitespace on the Band Reason `<div>` are all gone. Two JSX blocks (`Rule Description` and `Band Reason`) also collapsed from three lines to one.
- `types/index.ts:7-10` — trailing spaces after `rule_desc?: string | null;` gone, two stray blank lines before the closing `}` gone.

#### Item 7 — `backend/package-lock.json` — 133 `resolved`/`integrity` entries stripped (was ❌ Not resolved / Major)

**Status: NOT RESOLVED — worse than the follow-up round**

The `fix white-spaces` commit also touched `backend/package-lock.json`, stripping an *additional* 53 packages' `resolved`/`integrity` pairs. Current count:

```
grep -c '"resolved":' backend/package-lock.json                → 28
git show origin/dev:backend/package-lock.json | grep -c '"resolved":' → 214
```

That's 186 packages now missing tarball URLs and SHA-512 hashes (up from 133 in the follow-up round). The regression happened because the commit only fixed whitespace in the source files but continued to reflect a lockfile that was not regenerated with `npm install` / `npm ci`. No dependency changes are made in this PR, so the count should match `origin/dev` exactly.

**Why it matters:** `npm ci` verifies packages against `integrity` when present. With 186 entries stripped, integrity verification is skipped for those packages during install. `Dependency Review` remains green but it only reads top-level `package.json`, not transitive dev entries.

**Fix (unchanged from prior round):**

```bash
cd backend
rm package-lock.json
npm i --package-lock-only
```

Then confirm `grep -c '"resolved":' backend/package-lock.json` matches the `origin/dev` count of 214, then commit the regenerated file. If this strip is a deliberate convention I'm missing, a one-line note on the PR would close this item.

#### Item F2 — Three-space indent on restored `if (loading)` (was Minor)

**Status: RESOLVED**

`AlertNavigatorTab.tsx:85-86`:

```tsx
  };

  if (loading) {
```

Proper two-space indent, blank line above.

#### Item F3 — `AlertNavigatorTab.test.tsx:487-488` statistics DOM walk (was Informational)

**Status: RESOLVED — applied CodeRabbit's suggested form**

```ts
const typologiesCard = screen
  .getByText('Typologies Triggered')
  .closest('div')?.parentElement as HTMLElement;
expect(within(typologiesCard).getByText('1')).toBeInTheDocument();
```

The dead `typologiesCard && …` guard and inline `parentElement` walk are gone. Matches the rewrite suggested in the follow-up round.

### Summary Table — Final Resolution Status

| # | Item | Final Status |
|---|------|--------|
| 1 | "No data available" flashes during fetch | ✅ Resolved (`6af19f18`) |
| 2 | `RuleDetailDto.ruleDesc` type widening | ✅ Resolved (`ac86c1c1`, via rename) |
| 3 | Wire-format naming inconsistency | ✅ Resolved (`021a9b3f`, via inline rationale comment) |
| 4 | Phantom `band_reasons_with_sub_rule_refs_json` | ✅ Resolved (`021a9b3f`) |
| 5 | Five unused `RuleDetailDto` fields | ✅ Resolved (`ac86c1c1`) |
| 6 | `check style` failing (trailing whitespace) | ✅ Resolved (`021a9b3f` — CI green) |
| 7 | `backend/package-lock.json` — integrity entries stripped | ❌ **Worse — 186 entries stripped (was 133)** |
| 8 | PR title fails Conventional Commits | ✅ Resolved (`ac86c1c1`) |
| 9 | `check tests` failing | ✅ Resolved (`ac86c1c1`) |
| F1 | `check style` (dup of Item 6) | ✅ Resolved (`021a9b3f`) |
| F2 | Three-space indent on `if (loading)` | ✅ Resolved (`021a9b3f`) |
| F3 | Statistics test DOM walk | ✅ Resolved (`021a9b3f`) |

[↑ Back to top](#pr-review-cms-241--feat-paysysadd-rule-properties-inside-typology-in-visualization)

---

### New Issues Found in Final Commit

#### New Issue G1 — Lockfile integrity strip expanded

**Severity: Major (Supply-chain integrity — regression)**

The `fix white-spaces` commit (`021a9b3f`) stripped 53 additional `resolved`/`integrity` pairs from `backend/package-lock.json`, on top of the 133 already flagged in the initial round. The scope of the commit was supposed to be whitespace cleanup only — the lockfile changes are unrelated to that scope and should not have been included.

Grouped under Item 7 in the summary table but called out separately here because it is a *new* regression introduced in this round, not simply the persistence of an old issue.

**Suggested action:** regenerate the lockfile cleanly (`rm backend/package-lock.json && npm i --package-lock-only`) and commit the result. If the strip is intentional (e.g. an offline-mirror workflow), please confirm on the PR and I will withdraw both Item 7 and G1.

No other new issues introduced by `021a9b3f`. The commit is otherwise a clean whitespace / cosmetic cleanup that also closes several open recommendation items.

[↑ Back to top](#pr-review-cms-241--feat-paysysadd-rule-properties-inside-typology-in-visualization)

---

### Final Verdict

**Verdict: Changes Requested**

The author has done a thorough job across three rounds: the loading regression is fixed, the DTO is trimmed and correctly nullable, the naming decision is now explicitly documented in comments on both backend and frontend, the phantom `band_reasons_with_sub_rule_refs_json` field is removed, the test suite is substantial and well-scoped, the PR title passes Conventional Commits, `check tests` is green, and — new this round — `check style` is finally green as well. Every prior review recommendation except the lockfile issue has been actioned. On code quality this is now a merge-ready PR.

The one remaining blocker is the same one from the initial round, and it has actually **regressed**: the whitespace-fix commit stripped an additional 53 `resolved`/`integrity` entries from `backend/package-lock.json`, bringing the total to 186 packages (28 vs. 214 on `origin/dev`) that will not be integrity-verified during `npm ci`. This is unrelated to the whitespace fix that was the commit's stated purpose and needs a clean lockfile regeneration before merge.

### Blocking

1. **Lockfile integrity strip (Item 7 + G1)** — `backend/package-lock.json` is now missing 186 `resolved`/`integrity` entries (regressed further in `021a9b3f`). Regenerate cleanly with `npm i --package-lock-only` in `backend/` and commit; or, if the strip is deliberate, confirm on the PR.

### Non-blocking

None — every other item has been resolved.

[↑ Back to top](#pr-review-cms-241--feat-paysysadd-rule-properties-inside-typology-in-visualization)

---

### GitHub Review Comment

````markdown
**Changes Requested (final round)**

Great progress this round — every code-quality item from the prior two rounds is now closed. `check style` is green, the `if (loading)` branch is indented cleanly, the naming decision is documented in inline comments on both backend and frontend, the phantom `RawRuleRow.band_reasons_with_sub_rule_refs_json` field is gone, and the CodeRabbit pass-3 nitpick on `AlertNavigatorTab.test.tsx:487-488` was applied verbatim. On the feature itself this PR is ready.

The one blocker is `backend/package-lock.json`, and it has actually got worse in `021a9b3f`.

---

### Blocking

**1. `backend/package-lock.json` — 186 packages now missing `resolved`/`integrity`**

The `fix white-spaces` commit (`021a9b3f`) stripped 53 more `resolved`/`integrity` pairs from the lockfile, on top of the 133 already flagged in the earlier rounds. Direct count on the current head:

```
grep -c '"resolved":' backend/package-lock.json                → 28
git show origin/dev:backend/package-lock.json | grep -c '"resolved":' → 214
```

That's 186 packages that `npm ci` can no longer integrity-verify. This PR makes no dependency changes, so the count should match `origin/dev` exactly. The lockfile changes also seem outside the intent of a "fix white-spaces" commit.

Please regenerate cleanly and commit:

```bash
cd backend
rm package-lock.json
npm i --package-lock-only
```

After the regeneration, `grep -c '"resolved":' backend/package-lock.json` should be 214. If this strip is deliberate for your workflow (offline mirror, private registry, etc.), please say so on the PR and I'll withdraw the item.

---

Everything else from previous rounds is resolved. Once the lockfile is regenerated, this is good to merge.
````

[↑ Back to top](#pr-review-cms-241--feat-paysysadd-rule-properties-inside-typology-in-visualization)
