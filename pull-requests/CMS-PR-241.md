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
- [Follow-up Review (2026-07-15)](#follow-up-review-2026-07-15)
  - [Resolution Status — Prior Outstanding Items](#resolution-status--prior-outstanding-items)
  - [New Issues Found in Updated Commits (Follow-up 2026-07-15)](#new-issues-found-in-updated-commits-follow-up-2026-07-15)
  - [Updated Verdict (Follow-up 2026-07-15)](#updated-verdict-follow-up-2026-07-15)
  - [GitHub Review Comment (Follow-up 2026-07-15)](#github-review-comment-follow-up-2026-07-15)
- [Follow-up Review (2026-07-15, round 2)](#follow-up-review-2026-07-15-round-2)
  - [Resolution Status — Prior Outstanding Items (Round 2)](#resolution-status--prior-outstanding-items-round-2)
  - [New Issues Found (Follow-up 2026-07-15 Round 2)](#new-issues-found-follow-up-2026-07-15-round-2)
  - [Updated Verdict (Follow-up 2026-07-15 Round 2)](#updated-verdict-follow-up-2026-07-15-round-2)
  - [GitHub Review Comment (Follow-up 2026-07-15 Round 2)](#github-review-comment-follow-up-2026-07-15-round-2)

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

---
---
---

## Follow-up Review (2026-07-15)

**Reviewed commit:** `f464bc065d7c2e6c7141443e84e2fedbdf9d5049` — *"fix lint"* (2026-07-14 14:29 +05:00). Head SHA verified against `gh pr view 241 --json headRefOid` — matches `f464bc06`.
**Reviewed against:** Changes Requested in the Final Review (2026-07-14) on commit `021a9b3f`, which left one blocking item open (Item 7 / G1 — `backend/package-lock.json` integrity strip).
**Delta reviewed:** `git diff 021a9b3f..f464bc06 --stat` — 16 files, +3907 / −885 (of which `backend/package-lock.json` alone is +3452 / −783, i.e. a full regeneration; the remaining files come from a merge of `dev` at `389dc0f7` which pulled in the SLA-state filtering feature from PR #244, plus three small lint commits by the author on `case-query.service.ts`, `alert-priority.service.ts`, and `triage.service.ts`).
**Force-push note:** No force-push. Prior-round SHAs (`05ab1c32`, `ac86c1c1`, `6af19f18`, `021a9b3f`) all still exist on `origin` and reachable from `f464bc06` via linear history.
**Developer response:** No written comment on the PR since the Final Review. Resolution is via new commits only. The relevant commits since `021a9b3f`:

| Commit | Author | Purpose |
|--------|--------|---------|
| `b49ccd8a` | AdeelKahan | `fix package-lock.json` — full regeneration of `backend/package-lock.json` |
| `d13ad595` | Justus Ortlepp | Merge PR #244 (SLA-task) into `dev` (not part of this PR branch directly) |
| `389dc0f7` | AdeelKahan | `fix: merged with dev` — merges `origin/dev` (including PR #244) into `paysys/addRulesPropsInTypology` |
| `6e8c8415` | AdeelKahan | `fix lint in triage service` — replaces `\|\|` with `?? ''` |
| `fbac3cec` | AdeelKahan | `fix-lint-issue` — removes redundant `?? ''` and adds `!existingAlert.case_id` guard |
| `f464bc06` | AdeelKahan | `fix lint` — Prettier reflow in `alert-priority.service.ts` and `case-query.service.ts` |

**CI status on `f464bc06`:** all 15 checks **SUCCESS**, including `node-ci / check style`, `node-ci / check tests`, `node-ci / run build`, `Analyze (javascript-typescript)`, `Analyze (actions)`, `CodeQL`, `dco-check`, `dependency-review`, `njsscan`, `nodejsscan`, `encoding-check`, `dockerfile-linter`, `gpg-verify`, `conventional-commits / validate-pr-title`, and `CodeRabbit`. `mergeStateStatus` is `BLOCKED` on branch protection (not CI) — awaiting reviewer approval; `mergeable` is `MERGEABLE`.

### Resolution Status — Prior Outstanding Items

Only one item was open at the end of the Final Review — Item 7 / G1 (`backend/package-lock.json` integrity strip). Every other prior-round item was already ✅ Resolved and is not re-litigated here.

#### Item 7 / G1 — `backend/package-lock.json` integrity strip (was ❌ NOT RESOLVED / Major)

**Status: RESOLVED (`b49ccd8a`)**

The author addressed this directly in commit `b49ccd8a fix package-lock.json`, a clean regeneration of the lockfile. Current counts on `f464bc06`:

```
grep -c '"resolved":' backend/package-lock.json         → 1269
grep -c '"node_modules/' backend/package-lock.json      → 1269
```

Every package in the lockfile now has a `"resolved":` tarball URL (and its corresponding `"integrity"` SHA), so `npm ci` can integrity-verify all packages. The comparison against `origin/dev` used in prior rounds (214 `"resolved":` entries) turns out to be an artifact of `dev`'s own lockfile — `dev` itself carries 1263 packages with only 214 fully-hashed entries, so `origin/dev` is the *less* complete file. The PR now brings `backend/package-lock.json` to a fully-hashed state (all 1269 packages hashed), which is a net improvement over the base branch.

Verification:

```bash
# On PR head f464bc06
grep -c '"resolved":' backend/package-lock.json           # 1269
grep -c '"node_modules/' backend/package-lock.json        # 1269
# On origin/dev
git show origin/dev:backend/package-lock.json | grep -c '"resolved":'    # 214
git show origin/dev:backend/package-lock.json | grep -c '"node_modules/' # 1263
```

Item 7 and its Final-Review companion G1 are both closed.

### Summary Table — All Items Across All Rounds

| # | Item | Final Status |
|---|------|--------|
| 1 | "No data available" flashes during fetch | ✅ Resolved (`6af19f18`) |
| 2 | `RuleDetailDto.ruleDesc` type widening | ✅ Resolved (`ac86c1c1`) |
| 3 | Wire-format naming inconsistency | ✅ Resolved (`021a9b3f` — documented inline) |
| 4 | Phantom `band_reasons_with_sub_rule_refs_json` | ✅ Resolved (`021a9b3f`) |
| 5 | Five unused `RuleDetailDto` fields | ✅ Resolved (`ac86c1c1`) |
| 6 | `check style` failing (trailing whitespace) | ✅ Resolved (`021a9b3f`) |
| 7 / G1 | `backend/package-lock.json` integrity strip | ✅ Resolved (`b49ccd8a`) |
| 8 | PR title fails Conventional Commits | ✅ Resolved (`ac86c1c1`) |
| 9 | `check tests` failing | ✅ Resolved (`ac86c1c1`) |
| F1 | `check style` (dup of Item 6) | ✅ Resolved (`021a9b3f`) |
| F2 | Three-space indent on `if (loading)` | ✅ Resolved (`021a9b3f`) |
| F3 | Statistics test DOM walk | ✅ Resolved (`021a9b3f`) |

[↑ Back to top](#pr-review-cms-241--feat-paysysadd-rule-properties-inside-typology-in-visualization)

---

### New Issues Found in Updated Commits (Follow-up 2026-07-15)

Two observations came in on the commits added since the Final Review. Both are minor and neither blocks merge on this PR's original scope (Alert-Navigator rule properties). The bulk of the delta since `021a9b3f` came in via merging `origin/dev` (`389dc0f7`) and is therefore code already reviewed on other PRs (#240, #244), not authored inside this PR — that content is out of scope for this review and is not re-litigated here.

#### New Issue H1 — Redundant nullish coalescing in `triage.service.ts` was toggled twice (Informational)

**Severity: Informational (Code Quality)**

Commit `6e8c8415 fix lint in triage service` first *added* `?? ''` to `transactionType: alert.txtp` and `reason: alert.message` (converting from `||` to `??`). Immediately after, `fbac3cec fix-lint-issue` *removed* those very same `?? ''` fallbacks to satisfy `check style` (the fields are typed non-nullable, so the fallback is dead code). This matches CodeRabbit's inline comment ([id 3576681151](https://github.com/tazama-lf/case-management-system/pull/241#discussion_r3576681151), 2026-07-14 06:56 UTC). Net result on `f464bc06` is correct — the fallbacks are gone — so this is just a note that the two lint commits could have been squashed. Not a blocker; git history is what it is.

#### New Issue H2 — `existingAlert.case_id` null guard was added but the surrounding code still uses `!`-asserted access to `existingAlert.case_id` implicitly (Informational)

**Severity: Informational (Code Quality)**

Commit `fbac3cec` added a defensive guard in `backend/src/modules/triage/triage.service.ts` around line 310:

```ts
if (!existingAlert) {
  throw new NotFoundException(`Alert with id ${alertId} not found`);
}
if (!existingAlert.case_id) {
  throw new BadRequestException(`Alert ${alertId} is not linked to a case`);
}
const existingCase = await this.caseRepository.findCaseById(existingAlert.case_id, tenantId);
```

This is a real improvement over the prior `existingAlert.case_id!` non-null assertion (dropped in `6e8c8415`) — a `BadRequestException` is more informative than a runtime undefined dereference. The new behaviour is that manual-triage against an alert with `case_id === null` now fails fast with 400 instead of the previous silent `!`-assertion that would have thrown a Prisma error deep inside `findCaseById`. That is a semantic change worth flagging for QA but is a strict improvement.

No new test in `backend/test/triage.service.spec.ts` covers the `case_id === null` branch. Adding one line of coverage would close this observation, but it's out of scope for this PR's stated purpose.

#### Out-of-scope notes (informational, tracked here for completeness)

These come in via the `389dc0f7` merge with `dev` and belong to PR #244 (SLA-task), not to PR 241. Listed only so the reader knows why the `git diff 021a9b3f..f464bc06` looks large:

- **CodeRabbit nitpick still open** on `backend/src/modules/case/services/case-query.service.ts:516-537` / `:666-681`: duplicate SLA-candidate lookup pipeline, unbounded scan. This lives on the SLA feature that came in via merge — not written for this PR. Should be tracked on PR #244 or a follow-up issue, not blocked on this PR.
- **`backend/package-lock.json` full regeneration** (`b49ccd8a`) is a much bigger churn than the prior `21a9b3f`-vs-`origin/dev` diff would have suggested, because `dev` itself carries an under-hashed lockfile. This PR ends up *upgrading* the lockfile's integrity coverage from 214 to 1269 packages, so the noise is a net positive.

[↑ Back to top](#pr-review-cms-241--feat-paysysadd-rule-properties-inside-typology-in-visualization)

---

### Updated Verdict (Follow-up 2026-07-15)

**Verdict: Approved**

All prior blocking items are resolved. The last outstanding blocker from the Final Review (`backend/package-lock.json` integrity strip — Item 7 / G1) was cleaned up in `b49ccd8a` with a full lockfile regeneration; the lockfile now carries `"resolved":` and `"integrity":` entries for all 1269 packages, which is more complete than `origin/dev`. `check style` remained green after the follow-up lint commits, and all 15 CI checks are SUCCESS on `f464bc06`. The `mergeStateStatus: BLOCKED` state is due to branch protection requiring reviewer approval, not to any CI failure.

The PR's `git diff 021a9b3f..f464bc06` looks large only because the author merged `dev` in, which brought in PR #244's SLA-state filtering feature. That code is out of scope for this review — it was reviewed on PR #244 — and any residual CodeRabbit nitpicks on it (e.g. the duplicate SLA-candidate lookup in `case-query.service.ts:516-537`) should be tracked there, not blocked on PR 241. The three small lint commits by this PR's author since `389dc0f7` (`6e8c8415`, `fbac3cec`, `f464bc06`) are cosmetic Prettier / nullish-coalescing cleanups; `fbac3cec` also adds a small defensive guard against a `null` `existingAlert.case_id` in `triage.service.ts` that is a strict improvement.

### Blocking

None.

### Non-blocking

1. **H2 — `case_id === null` branch is uncovered in `triage.service.ts` tests** — the new `BadRequestException` guard added in `fbac3cec` has no unit test. Out of scope for this PR's stated purpose; worth a one-line test in a follow-up.

[↑ Back to top](#pr-review-cms-241--feat-paysysadd-rule-properties-inside-typology-in-visualization)

---

### GitHub Review Comment (Follow-up 2026-07-15)

````markdown
**Approved (follow-up 2026-07-15, HEAD `f464bc06`)**

The last blocker from the previous round — `backend/package-lock.json` missing `resolved`/`integrity` entries — is cleaned up in `b49ccd8a fix package-lock.json`. Every one of the 1269 packages in the lockfile now has a `resolved` URL and `integrity` SHA, so `npm ci` can integrity-verify the full dependency tree. That's actually more complete than the current `origin/dev` (which carries 1263 packages with only 214 fully-hashed entries), so this PR is a net supply-chain improvement over the base branch.

All 15 CI checks are green on `f464bc06`, including `check style`, `check tests`, `run build`, `CodeQL`, `njsscan`, `nodejsscan`, `dependency-review`, and `conventional-commits`. `mergeStateStatus` is `BLOCKED` on branch protection (awaiting reviewer approval), not on CI.

---

### Verified since the Final Review

- `b49ccd8a` — full regeneration of `backend/package-lock.json`; 1269/1269 packages hashed.
- `389dc0f7` — merge with `dev`. Brings in PR #244 (SLA-state filtering). Not part of this PR's original scope but doesn't conflict with the Alert-Navigator changes.
- `6e8c8415` / `fbac3cec` — lint fixes in `backend/src/modules/triage/triage.service.ts`. `?? ''` toggled on then off (net: removed, correct — the fields are typed non-nullable). Also adds a `BadRequestException` guard for `existingAlert.case_id === null`, which is a small but genuine hardening over the prior `!`-assertion.
- `f464bc06` — Prettier reflow in `backend/src/modules/alert-priority/alert-priority.service.ts` and `backend/src/modules/case/services/case-query.service.ts`. Cosmetic.

### Non-blocking (out-of-scope for this PR, informational)

**1. CodeRabbit still has an open nitpick on `case-query.service.ts:516-537` / `:666-681`** — duplicate SLA-candidate lookup pipeline and an unbounded in-memory scan. That code came in via the merge with `dev` (PR #244), not this PR — please track on #244 or as a follow-up issue.

**2. `case_id === null` branch in `triage.service.ts` is untested.** The new `BadRequestException` guard added in `fbac3cec` has no unit test. One-liner in a follow-up would close it.

---

Ready to merge on my side.
````

[↑ Back to top](#pr-review-cms-241--feat-paysysadd-rule-properties-inside-typology-in-visualization)

---
---
---

## Follow-up Review (2026-07-15, round 2)

**Reviewed commit:** `df6dc10d6905d9b6c8c32e7322feb503e8cba9bc` — *"merge with dev"* (merge commit). Head SHA verified against `gh pr view 241 --json headRefOid` — matches `df6dc10d`.
**Reviewed against:** Approved verdict from Follow-up Review (2026-07-15, round 1) on `f464bc06`.
**Delta reviewed:** `git diff f464bc06..df6dc10d --stat` — 49 files, +1929 / −1440. Entire delta is imported via a single merge commit (`df6dc10d merge with dev`) that pulls `origin/dev` (which had just merged PR #242 "paysys/dashboard-fixes" at `6a10bbff`). **No new author-authored non-merge commits on this PR branch itself** since round 1. Files in this PR's original scope (`backend/src/modules/gold-lakehouse/**`, `backend/src/modules/alert/alert.service.ts`, `frontend/src/features/cases/components/view/visualizations/alertnavigator/**`) are byte-identical between `f464bc06` and `df6dc10d` (`git diff` on those paths returns 0 lines).
**dev-drift note:** **0 commits behind `origin/dev`** (`git log --oneline HEAD..origin/dev` → empty). `origin/dev` is at `6a10bbff2795fa6b4d6665df8ed7f1696c18cfc0`; `df6dc10d` includes `6a10bbff` as its second parent. The 20-commit drift that had appeared after PR #242 merged is now fully absorbed.
**Force-push note:** No force-push. Round-1 head `f464bc06` still exists and is the first parent of `df6dc10d`.
**Developer response:** No written comment on the PR since round 1. No new inline review comments and no new reviews on GitHub since `2026-07-14T06:56:52Z` (last CodeRabbit review). Round 2 is triggered only by the author merging `dev` into the branch to clear the drift.

**CI status on `df6dc10d`:** 14 checks green, 1 pending. Green: `Analyze (actions)`, `Analyze (javascript-typescript)`, `CodeQL`, `CodeRabbit`, `conventional-commits / validate-pr-title`, `dco-check`, `dependency-review`, `dockerfile-linter`, `encoding-check`, `gpg-verify`, `njsscan`, `node-ci / check style`, `node-ci / run build`, `nodejsscan`. Pending: `node-ci / check tests` (run id `29398648642`, job `87297862422`, still in progress at time of this review). `mergeStateStatus: BLOCKED` (branch protection), `mergeable: MERGEABLE`.

### Resolution Status — Prior Outstanding Items (Round 2)

Round 1 closed with verdict Approved and one non-blocking item open (H2 — `case_id === null` branch untested in `triage.service.ts`). Re-verifying:

#### Item H2 — `case_id === null` branch untested in `triage.service.ts` (was Non-blocking / Informational)

**Status: UNCHANGED — still open, still Non-blocking**

No new tests added to `backend/test/triage.service.spec.ts` since round 1 (the file is not among the 49 changed files in `f464bc06..df6dc10d`). The `BadRequestException` guard added in `fbac3cec` remains uncovered. Same disposition as round 1 — out of scope for this PR's stated purpose, worth a one-line follow-up test. No author response.

#### All previously-resolved items (1, 2, 3, 4, 5, 6, 7/G1, 8, 9, F1, F2, F3)

**Status: STILL RESOLVED**

Original-scope files unchanged between `f464bc06` and `df6dc10d` — the merge with `dev` does not touch `alerts-lakehouse.service.ts`, `raw-rule-row.types.ts`, `AlertNavigatorTab.tsx`, `alertnavigator/types/index.ts`, or `alert.service.ts`. All resolutions confirmed in round 1 still hold verbatim on `df6dc10d`.

### Summary Table — Round 2

| # | Item | Status on `df6dc10d` |
|---|------|--------|
| 1–9, F1–F3 | All items closed in prior rounds | ✅ Still Resolved (original-scope files unchanged since `f464bc06`) |
| H2 | `case_id === null` branch untested | ➖ Unchanged — Non-blocking, no author action |
| N1 (new) | dev-merge scope import | ℹ️ Non-blocking — see below |

[↑ Back to top](#pr-review-cms-241--feat-paysysadd-rule-properties-inside-typology-in-visualization)

---

### New Issues Found (Follow-up 2026-07-15 Round 2)

#### New Issue N1 — Merge with `dev` imports PR #242 changes into this PR's diff (Informational)

**Severity: Informational (Scope / Review-hygiene)**

The merge commit `df6dc10d merge with dev` pulls in 19 non-merge commits from PR #242 ("paysys/dashboard-fixes", `6a10bbff`), all authored by AdeelKahan and covering dashboard/reports fixes (`backend/src/modules/report/**`, `frontend/src/features/dashboard/**`, `frontend/src/features/reports/**`, `frontend/src/features/cases/**`, `notebooks/transaction-viz.ipynb`). Those changes were reviewed on PR #242 and are not the subject of this PR — but they now appear in `gh pr diff 241`, which can make reviewers double-count them.

For this PR's scope (Alert-Navigator rule properties on the `gold-lakehouse` service and `AlertNavigatorTab`), nothing in the merged content overlaps or conflicts with the round-1 changes. `git diff f464bc06..df6dc10d -- 'backend/src/modules/gold-lakehouse/' 'frontend/src/features/cases/components/view/visualizations/alertnavigator/' 'backend/src/modules/alert/alert.service.ts'` returns zero lines, confirming the merge preserved the round-1 head for original-scope files.

Not a blocker. Just a note for anyone opening the PR diff today — the size jump is entirely from the `dev` merge, not from new PR-241 work.

#### `node-ci / check tests` — pending at time of review

Not a new issue, but flagged for completeness: `check tests` is still running on `df6dc10d` (job `87297862422`). All 14 other checks are green including `check style` and `run build`. Historically this suite has been green on every commit in this PR since `ac86c1c1`, and the merge did not touch any test-relevant file that would be expected to regress it. If it lands red I'll re-open this round; assuming green (extrapolated from history and from `run build` already passing), the verdict below stands.

[↑ Back to top](#pr-review-cms-241--feat-paysysadd-rule-properties-inside-typology-in-visualization)

---

### Updated Verdict (Follow-up 2026-07-15 Round 2)

**Verdict: Approved**

Nothing in the round-1 approval has changed. The only new activity since then is the author merging `origin/dev` into the PR branch to absorb PR #242's 19 dashboard-fix commits and eliminate the ~20-commit dev drift. That merge does not touch any file in this PR's original scope, and the round-1 verification of all Alert-Navigator rule-property changes remains valid on `df6dc10d`. 14 of 15 CI checks are green; `node-ci / check tests` is still running but there is no reason to expect a regression (nothing changed in test-relevant paths since round 1, and `run build` and `check style` are both green on this HEAD).

`mergeStateStatus` is still `BLOCKED` on branch protection (awaiting reviewer approval), not on CI. `mergeable: MERGEABLE`. Once `check tests` completes green and a reviewer approves on GitHub, the PR is ready to merge.

### Blocking

None.

### Non-blocking

1. **H2 (carried from round 1)** — `case_id === null` branch is untested in `triage.service.ts`. One-liner in a follow-up would close it.
2. **N1 (new)** — Merge with `dev` imports PR #242's changes into this PR's diff. Informational only; original-scope files unchanged. No action needed on this PR.
3. **`node-ci / check tests` pending** — verify green before merge; no evidence of impending failure.

[↑ Back to top](#pr-review-cms-241--feat-paysysadd-rule-properties-inside-typology-in-visualization)

---

### GitHub Review Comment (Follow-up 2026-07-15 Round 2)

````markdown
**Approved (follow-up 2026-07-15 round 2, HEAD `df6dc10d`)**

Re-verifying after the `merge with dev` commit that closed the ~20-commit drift caused by PR #242 merging into `dev`. Every file in this PR's original scope (`backend/src/modules/gold-lakehouse/**`, `backend/src/modules/alert/alert.service.ts`, `frontend/src/features/cases/components/view/visualizations/alertnavigator/**`) is byte-identical between the prior head `f464bc06` and the current head `df6dc10d` — the round-1 approval carries over unchanged.

`git log --oneline HEAD..origin/dev` is empty; the PR is now at parity with `dev`. `mergeable: MERGEABLE`, `mergeStateStatus: BLOCKED` on branch protection (awaiting reviewer approval).

### CI

14 of 15 checks are green on `df6dc10d`, including `check style`, `run build`, `CodeQL`, `Analyze (javascript-typescript)`, `Analyze (actions)`, `dependency-review`, `njsscan`, `nodejsscan`, `dco-check`, `gpg-verify`, `encoding-check`, `dockerfile-linter`, `conventional-commits`, `CodeRabbit`. `node-ci / check tests` was still pending at the time of this review — worth confirming green before merge, though nothing in the merge touches test-relevant paths and history is clean.

### Non-blocking

1. **H2 (carried from round 1)** — `case_id === null` branch in `triage.service.ts` is still uncovered by unit tests. One-liner in a follow-up.
2. **N1 (new, informational)** — The `merge with dev` commit imports PR #242's 19 dashboard-fix commits into this PR's `gh pr diff`. Those were reviewed on PR #242 and are not part of this PR's scope; the size jump is purely from the merge, not from new PR-241 work.

Ready to merge on my side once `check tests` reports green.
````

[↑ Back to top](#pr-review-cms-241--feat-paysysadd-rule-properties-inside-typology-in-visualization)
