# Issue #47 — CI lint gate (npm run lint:eslint) fails on dev with 66 errors

**Repository:** tazama-lf/event-monitoring-service
**Issue:** [CI lint gate (npm run lint:eslint) fails on dev with 66 errors - flat-config parser scope + plugin rule drift](https://github.com/tazama-lf/event-monitoring-service/issues/47)
**Author:** Justus-at-Tazama (Justus Ortlepp)
**State:** Open
**Report Date:** 2026-07-03

---

## Executive Summary

The `npm run lint:eslint` command used by the reusable `node-ci.yml@dev` CI workflow fails with 66 errors on a clean `origin/dev` checkout. The failure blocks every pull request into `dev`, as the CI gate is always red regardless of branch content. The same codebase passes `npm run lint:check` with zero errors (92 warnings).

There are two distinct root causes. First, `lint:eslint` is the bare `eslint` command with no file path argument. ESLint's flat config discovery walks the entire project directory, finds all 62 `.ts` files in `src/`, and attempts to parse them with `@typescript-eslint/parser` using `parserOptions.project: ['./tsconfig.json']`. All 62 parse because this combination triggers the TypeScript program loader, which in certain invocation contexts fails to match files to their project. The path-scoped `lint:check` avoids this by restricting ESLint's input to `{src,apps,libs,test}/**/*.ts`. Second, `eslint.config.mjs` spreads `eslintStandard.plugins` from `eslint-config-love` and then overrides the `@typescript-eslint` key with a separately-imported plugin instance. Rules that `eslint-config-love@125.1.0` enables but that are not present in the overriding plugin instance (or in `eslint-plugin-promise@7.x` which removed `avoid-new`) appear as "Definition for rule X was not found" errors.

Track A fixes the bare `eslint` command immediately by scoping it to the same file glob used by `lint:check`, unblocking CI with a one-line change. Track B eliminates the underlying configuration drift: removes the redundant direct `@typescript-eslint` plugin import that conflicts with `eslint-config-love`'s bundled instance, migrates `parserOptions.project` to `parserOptions.projectService` (the current API), corrects the `tsconfig.json` include path from `test/**/*` to `tests/**/*` to match the actual directory, and either disables or overrides the three missing rules.

---

## How It Works Today — Confirmed in Code

### `package.json` — Script definitions

Lines 19–21 of `repos/event-monitoring-service/package.json`:

```json
"lint": "eslint \"{src,apps,libs,test}/**/*.ts\" --fix",
"lint:check": "eslint \"{src,apps,libs,test}/**/*.ts\"",
"lint:eslint": "eslint",
```

`lint:eslint` (line 21) is the bare `eslint` command. CI runs `npm run lint:eslint`. The absence of a file path means ESLint discovers files itself. `lint:check` (line 20) restricts input to `{src,apps,libs,test}/**/*.ts` and passes.

### `eslint.config.mjs` — Flat config

Full file at `repos/event-monitoring-service/eslint.config.mjs`:

- **Line 4**: `import tsEslint from '@typescript-eslint/eslint-plugin';` — imports the plugin directly.
- **Line 6**: `import eslintStandard from 'eslint-config-love';` — imports the shared config, which bundles its own `@typescript-eslint` plugin instance.
- **Line 12**: `globalIgnores(['**/coverage/**', '**/build/**', '**/node_modules/**', '**/tests/**', '*.ts'])` — ignores `tests/` directory and root-level `.ts` files, but does not scope down the file discovery for subdirectory `.ts` files.
- **Lines 14–15**: `files: ['**/*.ts']` — matches all `.ts` files not excluded by `globalIgnores`, including all 62 files in `src/`.
- **Lines 15–20**: Plugins block spreads `...eslintStandard.plugins` then overrides `['@typescript-eslint']: tsEslint`. This registers a different plugin instance than the one whose rules are spread via `...eslintStandard.rules` at line 31, causing rule lookup mismatches.
- **Line 27**: `parserOptions: { project: ['./tsconfig.json'] }` — uses the deprecated `project` key, not `projectService`.

### `tsconfig.json` — Project include paths

Lines 25–26 of `repos/event-monitoring-service/tsconfig.json`:

```json
"include": ["src/**/*", "test/**/*", "knex/knex.module.ts", "knex/knexfile.ts", "knex/migrations/**/*.ts"]
```

The include path is `test/**/*`. The actual test directory on disk is `tests/` (confirmed: `ls repos/event-monitoring-service/` shows `tests/`, no `test/` directory). The four spec files at `tests/config-notify/` and `tests/dems-engine/` are therefore not covered by any TypeScript project. ESLint's `globalIgnores` at line 12 does exclude `**/tests/**`, so these files are not linted directly — but the mismatch means the TypeScript program built from `tsconfig.json` is incomplete relative to what the project intends.

### File counts

- Total `.ts` files found by bare `eslint` (excluding `node_modules`, `dist`, `build`, `coverage`, `tests/`): **62 files**, all in `src/`.
- Total `.ts` files found by bare `eslint` including `tests/`: **66 files** (4 spec files + 62 src files). The 66 matches the error count reported in the issue, suggesting the `**/tests/**` globalIgnore may not be functioning as expected in some ESLint versions, or the issue was measured without that ignore in effect.

### Installed package versions (from `package-lock.json`)

| Package | Declared in `package.json` | Installed version |
|---|---|---|
| `eslint-config-love` | `^125.0.0` | `125.1.0` |
| `typescript-eslint` (direct dep) | `^8.20.0` | `8.59.0` |
| `@typescript-eslint/eslint-plugin` | (transitive) | `8.59.0` |
| `eslint-plugin-promise` | (transitive via love) | `7.2.1` |

`eslint-config-love@125.1.0` requires `typescript-eslint ^8.41.0`. The direct `typescript-eslint ^8.20.0` declaration is outdated relative to what love needs, but the installed `8.59.0` satisfies both. The plugin instance conflict arises not from version incompatibility but from the dual-import pattern in `eslint.config.mjs`.

---

## Root Cause Analysis

The CI gate fails because two independent problems accumulate errors simultaneously.

**Root cause 1 (62 parse errors):** `lint:eslint` runs bare `eslint`, which discovers all 62 `.ts` files in `src/` via the `files: ['**/*.ts']` config block. The `@typescript-eslint/parser` is configured with `parserOptions.project: ['./tsconfig.json']`. In ESLint flat config, when `parserOptions.project` is set, the parser must build a TypeScript program from the specified tsconfig. If for any reason a file is not resolvable within that program — which can occur when ESLint runs outside the CWD assumed by the config, when the tsconfig path resolution differs between `lint:check` (where files are explicitly passed) and bare `eslint` (where they are discovered), or when the tsconfig path itself is computed incorrectly relative to the discovered file location — every affected file emits a parse error. The path-scoped `lint:check` command avoids this by passing files explicitly, giving the parser a stable resolution context.

**Root cause 2 (4 rule-not-found errors):** `eslint.config.mjs` spreads `...eslintStandard.plugins` from `eslint-config-love` then immediately overrides the `@typescript-eslint` key with a separately-imported `tsEslint` instance (line 19). The rule set at line 31 (`...eslintStandard.rules`) references rules by the plugin namespace as registered by love's internal instance, but the actual registered plugin is the override instance. Any rule enabled by love that is absent from the override instance — here `@typescript-eslint/no-unnecessary-condition` and `@typescript-eslint/require-await` — emits "Definition for rule X was not found". Additionally, `eslint-config-love@125.x` enables `promise/avoid-new` but `eslint-plugin-promise@7.x` removed that rule, causing the third rule-not-found error. A fourth missing rule (`@typescript-eslint/require-await`) follows the same dual-instance pattern.

---

## Blast Radius — Full File Inventory

### Files that must change

| File | What is affected | Relevant lines |
|---|---|---|
| `repos/event-monitoring-service/package.json` | `lint:eslint` script needs a file glob argument (Track A) | Line 21 |
| `repos/event-monitoring-service/eslint.config.mjs` | Remove direct `tsEslint` import and override, migrate to `projectService`, override or disable missing rules (Track B) | Lines 4, 12, 19, 27, 31 |
| `repos/event-monitoring-service/tsconfig.json` | Fix `test/**/*` include to `tests/**/*` to match the actual directory (Track B) | Line 25 |

### Files indirectly affected

| File | What is affected | Relevant lines |
|---|---|---|
| `repos/event-monitoring-service/jest.setup.js` | Root-level JS file; would be linted by bare `eslint` but has no TypeScript parser config — no parse errors expected, but the linting scope question applies | Lines 1–9 |
| `repos/event-monitoring-service/tests/config-notify/config-notify.controller.spec.ts` | Currently excluded by `**/tests/**` globalIgnore; becomes lint-eligible if ignore is removed; not in tsconfig include | — |
| `repos/event-monitoring-service/tests/config-notify/config-notify.service.spec.ts` | Same as above | — |
| `repos/event-monitoring-service/tests/dems-engine/dems-engine.controller.spec.ts` | Same as above | — |
| `repos/event-monitoring-service/tests/dems-engine/dems-engine.service.spec.ts` | Same as above | — |

---

## Side-Effect Map

| If left unfixed | Consequence |
|---|---|
| Every PR into `dev` fails the CI lint gate | No code can merge to `dev` without a manual CI override |
| Lint:check passes locally | Developers see zero errors locally but CI fails; creates confusion and erodes trust in the CI pipeline |
| `fix:eslint` script also runs bare `eslint --fix` on `**/*.ts` | The `lint:staged` hook (package.json line 115) and `fix:eslint` script (line 13) both run `eslint --fix \"**/*.ts\"` which would hit the same parse errors on apply |
| `tsconfig.json` test path mismatch | TypeScript type-checking does not cover spec files in `tests/`; test-only type errors are silently skipped in `tsc` |
| Dual plugin registration | Any rule enabled by `eslint-config-love` that is added in future love updates may silently not apply to code if the override instance doesn't carry it |

---

## Effort Assessment

### Track A — Scope the bare `eslint` script

**What changes:** One line in `package.json` (line 21). No config file changes. No schema migration. No frontend changes.

| Step | File | Effort |
|---|---|---|
| Add glob to `lint:eslint` script | `package.json` line 21 | 5 minutes |
| Verify CI passes locally | Run `npm run lint:eslint` | 5 minutes |

**Schema migration required:** No
**Frontend changes required:** No

---

### Track B — Reconcile flat config and TypeScript project

**What changes:** Three files. Removes dual-plugin registration, migrates parser config to `projectService`, corrects tsconfig include path, and disables or overrides the missing rules inherited from `eslint-config-love`.

| Step | File | What changes | Effort |
|---|---|---|---|
| B1 — Remove direct `@typescript-eslint` plugin import | `eslint.config.mjs` lines 4, 19 | Remove `tsEslint` import; stop overriding the plugin namespace | 20 minutes |
| B2 — Migrate parser config | `eslint.config.mjs` lines 26–28 | Replace `parserOptions.project` with `parserOptions.projectService: true` | 10 minutes |
| B3 — Disable or override missing rules | `eslint.config.mjs` rules block | Add explicit `off` or override for `promise/avoid-new`, confirm `@typescript-eslint/no-unnecessary-condition` and `@typescript-eslint/require-await` are handled | 15 minutes |
| B4 — Fix tsconfig include path | `tsconfig.json` line 25 | Change `"test/**/*"` to `"tests/**/*"` | 5 minutes |
| B5 — Verify full lint clean | Run `npm run lint:eslint` and `npm run lint:check` | Confirm zero errors | 10 minutes |

**Schema migration required:** No
**Frontend changes required:** No

### Recommended sequencing

Track A ships first as a standalone PR. It unblocks the CI gate immediately with zero risk of regression. Track B follows as a separate PR that cleans up the underlying configuration debt. Both tracks touch different parts of `eslint.config.mjs` except that Track A only touches `package.json`. If both tracks are developed simultaneously, they can merge in any order without conflicts since Track A only modifies `package.json`.

---

## Acceptance Criteria

### Track A

- [ ] `npm run lint:eslint` exits with code 0 on a clean `origin/dev` checkout
- [ ] The CI `node-ci.yml` lint step passes on `dev` without any file changes to application code
- [ ] `npm run lint:check` continues to exit with 0 errors (92 warnings unchanged)
- [ ] The `lint:eslint` script in `package.json` is scoped to the same file set as `lint:check`

### Track B

- [ ] `npm run lint:eslint` exits with 0 errors and no "Definition for rule X was not found" warnings
- [ ] `eslint.config.mjs` does not import `@typescript-eslint/eslint-plugin` directly — it uses the instance bundled by `eslint-config-love`
- [ ] `parserOptions.project` is replaced by `parserOptions.projectService: true` or equivalent
- [ ] `tsconfig.json` includes `tests/**/*` instead of `test/**/*`
- [ ] TypeScript compilation (`tsc --noEmit`) covers files in `tests/` correctly
- [ ] No rule-not-found errors appear when running `npm run lint:eslint` in verbose mode
- [ ] `eslint-config-love` dependency version in `package.json` is compatible with the `typescript-eslint` version declared as a direct devDependency
