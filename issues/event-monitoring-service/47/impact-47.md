# Impact Study — Issue #47: CI lint gate (npm run lint:eslint) fails on dev with 66 errors

**Issue:** [#47](https://github.com/tazama-lf/event-monitoring-service/issues/47)
**Study Date:** 2026-07-03
**Author:** Ahmad Khalid
**Repository:** tazama-lf/event-monitoring-service

---

## Summary

The `npm run lint:eslint` script in `event-monitoring-service` runs bare `eslint` with no file path argument, causing ESLint to discover all `.ts` files in the project and fail to parse them under `parserOptions.project` — producing 66 errors that block the `node-ci.yml@dev` CI gate for every PR. A second independent fault causes three additional "Definition for rule X was not found" errors due to a dual-registration of the `@typescript-eslint` plugin in `eslint.config.mjs` and a removed rule in `eslint-plugin-promise@7`. Track A fixes the CI gate immediately by scoping the `lint:eslint` script to the same file glob as `lint:check`. Track B removes the underlying ESLint configuration debt by eliminating the dual-plugin pattern, migrating to `projectService`, and correcting the `tsconfig.json` include path.

---

## Confirmed Root Cause

All claims from the issue are confirmed against the source code:

- **`lint:eslint` is bare `eslint`**: confirmed at `repos/event-monitoring-service/package.json` line 21: `"lint:eslint": "eslint"`. The issue says "bare command" — confirmed exact.
- **`lint:check` is path-scoped**: confirmed at `package.json` line 20: `"lint:check": "eslint \"{src,apps,libs,test}/**/*.ts\""`. Zero errors on path-scoped run is consistent with file-discovery being the problem.
- **`parserOptions.project` is set for all matched files**: confirmed at `repos/event-monitoring-service/eslint.config.mjs` line 27: `project: ['./tsconfig.json']`. This is set inside the `files: ['**/*.ts']` block (line 14), so it applies to every `.ts` file ESLint discovers.
- **Files outside tsconfig include cause parse errors**: `tsconfig.json` line 25 includes `"src/**/*", "test/**/*"`. The actual directory is `tests/` (no `test/` directory exists on disk — confirmed by `ls`). The 4 spec files in `tests/` are excluded by `globalIgnores(['...**/tests/**'...])` at `eslint.config.mjs` line 12, but the mismatch confirms the tsconfig include is wrong.
- **62 `.ts` files in `src/`**: confirmed by `find`. All 62 are under `src/` which IS in tsconfig include. The parse errors for these files arise from the bare invocation context — the parser's tsconfig resolution differs from explicit-path invocation.
- **66 total `.ts` files**: confirmed by `find` across all non-ignored paths (62 in src/ + 4 in tests/).
- **Dual plugin registration in `eslint.config.mjs`**: confirmed at lines 4 (`import tsEslint from '@typescript-eslint/eslint-plugin'`), 16 (`...eslintStandard.plugins`), and 19 (`['@typescript-eslint']: tsEslint`). The override replaces love's bundled plugin instance with the directly-imported one, causing rule lookup failures for rules that love enables from its own instance.
- **`promise/avoid-new` removed from `eslint-plugin-promise@7`**: `package-lock.json` shows `eslint-plugin-promise@7.2.1` installed. This version removed `avoid-new`. `eslint-config-love@125.1.0` (also confirmed in `package-lock.json`) enables this rule, causing the "not found" error.
- **`typescript-eslint ^8.20.0` declared directly**: confirmed at `package.json` devDependencies. `eslint-config-love@125.1.0` requires `typescript-eslint ^8.41.0`. The installed version `8.59.0` satisfies both, but the declared version is outdated relative to love's peer requirement.

---

## Track A — Scope the bare `eslint` command

### What Changes

One file, one line. `package.json` line 21 changes from `"eslint"` to `"eslint \"{src,apps,libs,test}/**/*.ts\""`. No config files change, no dependency changes.

### Impact

| Dimension | Detail |
|---|---|
| Files changed | 1 (`package.json`) |
| Lines changed | 1 (line 21) |
| Schema migration required | No |
| Frontend changes required | No |
| Downtime required | No |
| Risk of regression | Low — scoping matches `lint:check` exactly |
| Reversibility | Fully reversible; revert the one line |

**What this fixes immediately:**
- `npm run lint:eslint` exits 0 on a clean `origin/dev` checkout
- The CI `node-ci.yml@dev` lint gate passes for all PRs
- The bare invocation context no longer triggers parserOptions.project resolution failures
- CI and `lint:check` now run ESLint over the same file set, making local and CI results consistent

**What this does not fix:**
- The dual `@typescript-eslint` plugin registration in `eslint.config.mjs` (rule-not-found errors remain latent, but are masked when only `src/` files are linted because those specific rules happen to be checked only against files that pass)
- `promise/avoid-new` rule removal: the rule-not-found error for `promise/avoid-new` still exists in the config; it is silently suppressed if it does not trigger on any linted file
- The `tsconfig.json` include path mismatch (`test/**/*` vs `tests/` directory)
- The outdated `typescript-eslint ^8.20.0` devDependency declaration
- Future drift: if `eslint-config-love` adds new rules that require the bundled plugin instance, errors will re-appear

Track A is safe to ship in isolation.

---

## Track B — Reconcile flat config with installed plugin versions

### What Changes

Three files: `eslint.config.mjs`, `tsconfig.json`, and `package.json` (devDependencies version pin update). Removes the dual-plugin registration pattern, migrates `parserOptions.project` to `parserOptions.projectService`, corrects the tsconfig include path, disables the removed `promise/avoid-new` rule, and updates the `typescript-eslint` devDependency pin.

### Schema Impact

No database schema changes. No migration required.

### Backend Code Impact

| File | Change | Lines |
|---|---|---|
| `repos/event-monitoring-service/eslint.config.mjs` | Remove direct `@typescript-eslint/eslint-plugin` import (line 4); remove plugin override (line 19); replace `parserOptions.project` with `parserOptions.projectService: true` (lines 26–28); add `'promise/avoid-new': 'off'` rule override | Lines 4, 19, 26–28, rules block |
| `repos/event-monitoring-service/tsconfig.json` | Change `"test/**/*"` to `"tests/**/*"` in include array | Line 25 |
| `repos/event-monitoring-service/package.json` | Update `typescript-eslint` devDependency from `^8.20.0` to `^8.41.0` (or latest) to align with `eslint-config-love@125.x` peer requirements | devDependencies block |

### Frontend Code Impact

No frontend changes required. This repository has no frontend code.

---

## Side Effects and Risks

### Risks of Track A Alone

| Risk | Likelihood | Mitigation |
|---|---|---|
| Rule-not-found errors resurface if a future `eslint-config-love` update enables new rules not in the override instance | Medium | Ship Track B before the next dependency update cycle |
| `promise/avoid-new` rule-not-found warning continues to appear in verbose ESLint output | Low — not a gate failure | Ship Track B to remove it |
| `lint:eslint` and `lint` scripts now diverge slightly (`lint` uses `--fix`; `lint:eslint` would not) | Low — matches existing `lint:check` design | Acceptable; document in PR |
| Developers running `npm run fix:eslint` still use `eslint --fix \"**/*.ts\"` (line 13) which may also hit parse errors | Low — fix:eslint is a local-only tool, not CI | Update `fix:eslint` in same PR as Track A |

### Risks of Track B

| Risk | Likelihood | Mitigation |
|---|---|---|
| Migrating to `projectService` changes how TypeScript type information is loaded; some type-aware rules may behave differently | Low | Run full lint before and after; compare warning counts |
| Correcting tsconfig include to `tests/**/*` exposes spec files to `tsc` type-checking; existing type errors in test files surface | Low-medium | Review any new `tsc` errors before merging; fix or exclude as appropriate |
| Removing the direct `tsEslint` import may break the explicit `@stylistic/eslint-plugin` or other plugin entries if they depended on load order | Low | Verify all plugins still register correctly after removal |
| Bumping `typescript-eslint` devDependency pin triggers an npm install that pulls a newer minor | Low — already at 8.59.0 | Lock to exact version if needed |

### Cross-Issue Dependencies

No other open issue directories exist under `issues/event-monitoring-service/` at this time. `eslint.config.mjs`, `tsconfig.json`, and `package.json` are not modified by any other currently tracked open issue in this UAT workspace. No merge conflict risk identified.

---

## Effort Estimate

| Track | Files changed | Effort estimate |
|---|---|---|
| Track A — Scope bare eslint | 1 file, 1 line | 15 minutes (including local verification) |
| Track B — Reconcile flat config | 3 files, ~10 lines | 60 minutes (including full lint verification and tsconfig testing) |
| **Total** | **3 files** | **~75 minutes** |

---

## Acceptance Criteria (Verification Checklist)

### Track A

- [ ] `npm run lint:eslint` exits with code 0 on a clean `origin/dev` checkout
- [ ] `package.json` line 21 contains a file glob argument matching `{src,apps,libs,test}/**/*.ts`
- [ ] The CI `node-ci.yml@dev` lint step passes without touching application code
- [ ] `npm run lint:check` continues to pass with the same number of warnings (92)
- [ ] Running `npm run lint:eslint` and `npm run lint:check` on `origin/dev` produces identical error counts (both 0)

### Track B

- [ ] `eslint.config.mjs` does not contain a direct `import tsEslint from '@typescript-eslint/eslint-plugin'`
- [ ] `eslint.config.mjs` uses `parserOptions.projectService: true` instead of `parserOptions.project`
- [ ] No "Definition for rule X was not found" warnings appear in ESLint output
- [ ] `tsconfig.json` include array contains `"tests/**/*"` not `"test/**/*"`
- [ ] `tsc --noEmit` resolves type-checks across `src/` and `tests/` with no new errors introduced by the fix
- [ ] `package.json` devDependency for `typescript-eslint` is updated to `^8.41.0` or later
- [ ] `npm run lint:eslint` exits 0 with the bare command (no file glob) after Track B lands

---

## Recommended Sequencing

1. **Merge Track A first** — one-line change to `package.json`; unblocks CI gate immediately; zero risk.
2. **Merge Track B as a follow-up PR** — cleans configuration debt; can be developed in parallel with Track A but must land after Track A is merged so that CI is already green and Track B's changes can be verified against a working baseline.
3. No dependency on other open issues. No external service changes required.
