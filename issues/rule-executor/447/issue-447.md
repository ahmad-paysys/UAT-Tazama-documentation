# Issue #447 — Rule Studio pipeline builds against diverged, hardcoded feat-paysys branch instead of a maintained branch

**Repository:** tazama-lf/rule-executer
**Issue:** [Rule Studio pipeline builds against diverged, hardcoded feat-paysys branch instead of a maintained branch](https://github.com/tazama-lf/rule-executer/issues/447)
**Author:** Justus-at-Tazama (Justus Ortlepp)
**State:** Open
**Report Date:** 2026-07-03

---

## Executive Summary

Every rule deployed through Rule Studio is built by cloning the `feat-paysys` branch of `tazama-lf/rule-executer`. That branch is a diverged, ad-hoc fork: as of 2026-07-03, it is 28 commits ahead and 35 commits behind `dev`. The clone target is hardcoded in two workflow files inside the `tazama-lf/rule-studio-example` template repository, which every rule repo bootstrapped by Rule Studio inherits.

The practical consequence is that every Studio-generated rule runs on executer code that misses all upstream fixes merged to `dev` since the branch diverged, while also carrying 28 unreviewed local commits that exist nowhere else in the maintained codebase — including verbose debug `console.log` statements, hardcoded UAT IP addresses baked into the `Dockerfile`, pinned pre-release library versions (`0.0.1-psl.0`), and a new `src/controllers/rule.ts` file added directly to `feat-paysys` that `dev` does not contain.

The fix has two tracks: Track A switches the hardcoded branch reference in the two deploy workflows to `dev`/`main`, and Track B reconciles the changes unique to `feat-paysys` — either merging the necessary adaptations into `dev` via a proper PR, or confirming they are obsolete and deleting the branch.

---

## How It Works Today — Confirmed in Code

### Workflow files in `tazama-lf/rule-studio-example`

**`deploy.yml` — line 39:**
```yaml
git clone https://github.com/tazama-lf/rule-executer -b feat-paysys
```
Source: `.github/workflows/deploy.yml:39`

**`deploy-to-uat.yml` — line 36:**
```yaml
git clone https://github.com/tazama-lf/rule-executer -b feat-paysys
```
Source: `.github/workflows/deploy-to-uat.yml:36`

Both workflows (for dev/UAT) clone identical branch names with no variable substitution. There is no mechanism to promote a different branch to a different environment.

### What `feat-paysys` actually contains (confirmed via `git diff origin/dev...origin/feat-paysys`)

| File | What feat-paysys changes |
|---|---|
| `Dockerfile` | Hardcodes UAT IP `10.10.80.18` for all DB hosts; ports `15432`; disables APM (`APM_ACTIVE=false`); sets `SERVER_URL=10.10.80.18:14222`; uses plain `ARG GH_TOKEN` instead of BuildKit secret mount |
| `package.json` | Pins `@tazama-lf/frms-coe-lib` at `0.0.1-psl.0` (a Paysys-internal pre-release); drops the `rule` dependency entirely; version `3.0.0` vs `4.0.0-rc.2` on `dev` |
| `.npmrc` | Adds `@psl-copilot:registry=https://npm.pkg.github.com` registry scope |
| `src/controllers/execute.ts` | Adds 20+ verbose debug `loggerService.log('[L16]…')` trace statements; casts `databaseManager` as `any` |
| `src/controllers/rule.ts` | **New file not present on `dev`**: provides a `handleTransaction` implementation that handles `BaseMessage` transactions with amount-band logic; contains raw `console.log("hello bhai…")` debug output |
| `src/config.ts` | Whitespace-only reformatting of a type alias (no semantic change) |
| `.husky/pre-commit` | Hook syntax differences (confirmed diverged, not read in detail) |
| `simple-rule2-test.js` | Not present on `dev`; test/debug file added to branch |

### The `sed` no-op problem

`deploy.yml` line 66 targets:
```bash
sed -E -i 's|"rule": "npm:@[^/]+/rule-placeholder@latest"|...|g'
```

`feat-paysys/package.json` has **no `rule` dependency** (confirmed). The `sed` finds no match and silently does nothing. The workflow then deletes and reinstalls the `rule` dependency via `npm pkg delete` / `npm install` (lines 111–111), which is why this functions at all — the `sed` is dead code in the context of `feat-paysys`.

---

## Root Cause Analysis

The root cause is a single hardcoded branch name in two workflow files: `git clone … -b feat-paysys`. `feat-paysys` was originally created as a working branch for Paysys-internal UAT customisation and was never reconciled with `dev` or placed on a promotion path. Because the clone target was set to this branch and never updated, it became the de facto release artifact for all Studio-generated rules, accumulating local-only commits with no upstream review.

Every downstream problem flows directly from this: the missing 35 `dev` commits, the unreviewed 28 branch-only commits, the hardcoded IP addresses, the debug logging in production executables, and the `@psl-copilot` registry scope that `tazama-lf` rules cannot resolve without special credentials.

---

## Blast Radius — Full File Inventory

### Files that must change

| File | Repository | What changes | Lines |
|---|---|---|---|
| `.github/workflows/deploy.yml` | `tazama-lf/rule-studio-example` | Replace `-b feat-paysys` with `-b dev` | Line 39 |
| `.github/workflows/deploy-to-uat.yml` | `tazama-lf/rule-studio-example` | Replace `-b feat-paysys` with `-b dev` | Line 36 |
| `Dockerfile` | `tazama-lf/rule-executer` (`feat-paysys`) | Remove hardcoded IPs, restore empty ENV defaults, re-enable APM, restore BuildKit secret mount | Lines 47–82 |
| `package.json` | `tazama-lf/rule-executer` (`feat-paysys`) | Align library versions with `dev`; add back `rule` placeholder dependency | Lines 25–34 |
| `.npmrc` | `tazama-lf/rule-executer` (`feat-paysys`) | Remove `@psl-copilot` scope if not needed on `dev`; or document and add to `dev` | Line 4 |
| `src/controllers/execute.ts` | `tazama-lf/rule-executer` (`feat-paysys`) | Remove debug `[L16]`-prefixed log statements; remove `as any` cast | Multiple |
| `src/controllers/rule.ts` | `tazama-lf/rule-executer` (`feat-paysys`) | Decision: merge cleaned version into `dev` or discard | Entire file |

### Files indirectly affected

| File | Repository | Indirect effect |
|---|---|---|
| `.github/workflows/deploy.yml` (all inheriting rule repos) | All Rule Studio rule repos | Must re-run to pick up corrected clone target (no file change, re-run triggers it) |
| `src/controllers/execute.ts` | `tazama-lf/rule-executer` (`dev`) | If `rule.ts` `handleTransaction` is needed, `execute.ts` import path changes |
| Any rule repo already deployed from `feat-paysys` | Per-rule repos | Containers built from `feat-paysys` must be rebuilt from corrected branch |

---

## Side-Effect Map

| Consequence | Triggered by |
|---|---|
| Rules miss upstream security/bug fixes in `dev` (35 commits) | `feat-paysys` is 35 commits behind `dev` |
| Rules run executer code with debug console output in production | `console.log("hello bhai…")` in `feat-paysys/src/controllers/rule.ts` |
| Dockerfile bakes UAT IPs into the image; container ignores runtime env vars | Hardcoded `ENV` values in `feat-paysys/Dockerfile` |
| APM disabled globally for all Studio rules | `APM_ACTIVE=false` in `feat-paysys/Dockerfile` |
| `@psl-copilot` packages unreachable for `tazama-lf`-org rules | `.npmrc` adds `@psl-copilot` scope without corresponding credentials for non-psl runners |
| `sed` patching of `package.json` silently does nothing | `feat-paysys` carries no `rule` placeholder dependency |
| Branch can receive further unreviewed commits with no gate | `feat-paysys` has no PR requirement pointing to `dev` |

---

## Effort Assessment

### Track A — Switch clone target (immediate fix)

Two one-line changes in two workflow files in `tazama-lf/rule-studio-example`.

| Step | File | Change | Effort |
|---|---|---|---|
| A1 | `.github/workflows/deploy.yml:39` | `-b feat-paysys` → `-b dev` | < 5 min |
| A2 | `.github/workflows/deploy-to-uat.yml:36` | `-b feat-paysys` → `-b dev` | < 5 min |

- Schema migration required: **No**
- Frontend changes required: **No**
- Estimated effort: **30 minutes** (including PR review and propagation check)

After Track A, every new rule deployment clones `dev`. Previously deployed containers are not affected until they are rebuilt.

### Track B — Reconcile `feat-paysys` with `dev`

A full reconciliation: audit each `feat-paysys`-only commit, decide merge vs discard, open a PR into `dev`, then delete the branch.

| Step | File(s) | Change | Effort |
|---|---|---|---|
| B1 | `Dockerfile` | Remove hardcoded IPs; restore empty ENV defaults; restore BuildKit secret mount (`--mount=type=secret`) | 1 hr |
| B2 | `package.json` | Align versions to `dev` (`frms-coe-lib 8.2.0-rc.6`, `frms-coe-startup-lib 3.1.0-rc.8`); restore `rule` placeholder | 30 min |
| B3 | `.npmrc` | Decide if `@psl-copilot` scope is needed on `dev`; add or remove | 15 min |
| B4 | `src/controllers/execute.ts` | Strip all debug `[L##]`-prefixed log statements; remove `as any` cast | 1 hr |
| B5 | `src/controllers/rule.ts` | Assess: if `BaseMessage` handling is required by the pipeline, clean (remove `console.log`) and open as a proper PR to `dev`; otherwise delete | 2–4 hr |
| B6 | `simple-rule2-test.js` | Delete (test/debug file, not present on `dev`) | 5 min |
| B7 | `.husky/pre-commit` | Align with `dev` hook | 15 min |
| B8 | Delete `feat-paysys` | Post-merge | 5 min |

- Schema migration required: **No**
- Frontend changes required: **No**
- Estimated effort: **1–2 days** depending on `rule.ts` decision

### Recommended sequencing

1. Ship Track A immediately — it is a two-line change and stops new deployments from using the diverged branch.
2. Open a parallel audit of the 28 `feat-paysys`-only commits: determine which are required by the Rule Studio pipeline (especially `rule.ts`) and which are debug/UAT-only artifacts.
3. Ship Track B as a reviewed PR into `dev` once the audit is complete.
4. Delete `feat-paysys` after the Track B PR merges.

---

## Acceptance Criteria

### Track A
- [ ] `deploy.yml:39` clones `rule-executer` from branch `dev`
- [ ] `deploy-to-uat.yml:36` clones `rule-executer` from branch `dev`
- [ ] A newly triggered Rule Studio deploy workflow builds against `dev` (verify via workflow log showing the correct branch SHA)
- [ ] No change made to `feat-paysys` or any rule repo

### Track B
- [ ] `Dockerfile` on `dev` contains no hardcoded IP addresses in ENV defaults
- [ ] `Dockerfile` uses BuildKit secret mount (`--mount=type=secret,id=GH_TOKEN`) not plain `ARG GH_TOKEN`
- [ ] `APM_ACTIVE` defaults to `true` on `dev`
- [ ] `package.json` on `dev` carries a `rule` placeholder dependency matching the `sed` pattern in `deploy.yml`
- [ ] `src/controllers/execute.ts` contains no `[L##]`-prefixed debug log statements and no `as any` cast
- [ ] `src/controllers/rule.ts` has been either cleanly merged into `dev` (with `console.log` debug lines removed) or confirmed absent and not needed
- [ ] `simple-rule2-test.js` is deleted from the branch
- [ ] `feat-paysys` branch is deleted from the remote
- [ ] A rebuilt rule container from `dev` passes end-to-end execution in UAT
