## Complete Fix — Exact Code Changes

Track A can merge independently. It touches only `package.json` and unblocks CI immediately. Track B must land as a coordinated change across `eslint.config.mjs`, `tsconfig.json`, and the `typescript-eslint` devDependency pin; all three changes are in the same repository and can ship as a single PR after Track A.

---

### Track A — Scope the bare eslint command (PR 1 of 2)

#### `repos/event-monitoring-service/package.json`

**BEFORE** (line 21):
```json
"lint:eslint": "eslint",
```

**AFTER** (line 21):
```json
"lint:eslint": "eslint \"{src,apps,libs,test}/**/*.ts\"",
```

This aligns `lint:eslint` with `lint:check` (line 20). ESLint now receives an explicit file set — the same 62 `.ts` files that `lint:check` processes — instead of discovering files from the working directory. The TypeScript parser's `parserOptions.project` resolution works reliably when files are passed explicitly rather than discovered, eliminating all 62 parse errors.

While making this change, also update `fix:eslint` (line 13) for consistency:

**BEFORE** (line 13):
```json
"fix:eslint": "eslint --fix \"**/*.ts\"",
```

**AFTER** (line 13):
```json
"fix:eslint": "eslint --fix \"{src,apps,libs,test}/**/*.ts\"",
```

**Impact of Track A alone:** CI passes. All 66 errors are eliminated in the CI context. The rule-not-found warnings remain latent in the config but do not surface as gate failures because they are not triggered by the explicitly-scoped file set.

---

### Track B — Reconcile flat config with installed plugin versions (PR 2 of 2)

#### Step B1 — Remove direct `@typescript-eslint` plugin import and override

**File:** `repos/event-monitoring-service/eslint.config.mjs`

**BEFORE** (lines 1–20):
```js
// SPDX-License-Identifier: Apache-2.0
import eslintPluginEslintComments from '@eslint-community/eslint-plugin-eslint-comments';
import stylistic from '@stylistic/eslint-plugin';
import tsEslint from '@typescript-eslint/eslint-plugin';
import tsParser from '@typescript-eslint/parser';
import eslintStandard from 'eslint-config-love';
import eslintConfigPrettier from 'eslint-config-prettier/flat';
import { defineConfig, globalIgnores } from 'eslint/config';

export default defineConfig([
  eslintConfigPrettier,
  globalIgnores(['**/coverage/**', '**/build/**', '**/node_modules/**', '**/tests/**', '*.ts']),
  {
    files: ['**/*.ts'],
    plugins: {
      ...eslintStandard.plugins,
      ['@eslint-community/eslint-comments']: eslintPluginEslintComments,
      ['@stylistic']: stylistic,
      ['@typescript-eslint']: tsEslint,
    },
```

**AFTER** (lines 1–20):
```js
// SPDX-License-Identifier: Apache-2.0
import eslintPluginEslintComments from '@eslint-community/eslint-plugin-eslint-comments';
import stylistic from '@stylistic/eslint-plugin';
import tsParser from '@typescript-eslint/parser';
import eslintStandard from 'eslint-config-love';
import eslintConfigPrettier from 'eslint-config-prettier/flat';
import { defineConfig, globalIgnores } from 'eslint/config';

export default defineConfig([
  eslintConfigPrettier,
  globalIgnores(['**/coverage/**', '**/build/**', '**/node_modules/**', '**/tests/**', '*.ts']),
  {
    files: ['**/*.ts'],
    plugins: {
      ...eslintStandard.plugins,
      ['@eslint-community/eslint-comments']: eslintPluginEslintComments,
      ['@stylistic']: stylistic,
    },
```

The `tsEslint` import (old line 4) is removed. The `['@typescript-eslint']: tsEslint` plugin override (old line 19) is removed. The `@typescript-eslint` plugin namespace is now provided exclusively by `...eslintStandard.plugins`, which is the instance that `eslint-config-love@125.1.0` uses internally. This ensures the rules spread by `...eslintStandard.rules` are looked up against the same plugin instance that registered them, eliminating the "Definition for rule X was not found" errors for `@typescript-eslint/no-unnecessary-condition` and `@typescript-eslint/require-await`.

Note: `@typescript-eslint/eslint-plugin` can be removed from `devDependencies` in `package.json` once this change lands, since the plugin is now consumed only through `eslint-config-love`'s transitive dependency.

---

#### Step B2 — Migrate `parserOptions.project` to `parserOptions.projectService`

**File:** `repos/event-monitoring-service/eslint.config.mjs`

**BEFORE** (lines 21–29):
```js
    languageOptions: {
      ...eslintStandard.languageOptions,
      parser: tsParser,
      ecmaVersion: 2022,
      sourceType: 'module',
      parserOptions: {
        project: ['./tsconfig.json'],
      },
    },
```

**AFTER** (lines 21–29):
```js
    languageOptions: {
      ...eslintStandard.languageOptions,
      parser: tsParser,
      ecmaVersion: 2022,
      sourceType: 'module',
      parserOptions: {
        projectService: true,
        tsconfigRootDir: import.meta.dirname,
      },
    },
```

`projectService: true` enables the TypeScript Language Service API, which correctly discovers the nearest `tsconfig.json` for each linted file without requiring an explicit path. `tsconfigRootDir: import.meta.dirname` anchors the root to the directory containing `eslint.config.mjs`. This is the current `typescript-eslint` recommendation for flat config and eliminates the resolution ambiguity that causes parse errors when ESLint discovers files by walking the directory tree.

---

#### Step B3 — Disable `promise/avoid-new` and verify missing rule overrides

**File:** `repos/event-monitoring-service/eslint.config.mjs`

**BEFORE** — rules block (lines 30–65), relevant excerpt:
```js
    rules: {
      ...eslintStandard.rules,
      ...eslintPluginEslintComments.configs.recommended.rules,
      // ... existing overrides ...
      'eslint-comments/no-unused-enable': 'off',
    },
```

**AFTER** — rules block, add at the end of the existing overrides:
```js
    rules: {
      ...eslintStandard.rules,
      ...eslintPluginEslintComments.configs.recommended.rules,
      // ... existing overrides unchanged ...
      'eslint-comments/no-unused-enable': 'off',
      /* promise/avoid-new was removed from eslint-plugin-promise v7; disable to prevent
         "Definition for rule not found" errors from eslint-config-love enabling it */
      'promise/avoid-new': 'off',
    },
```

`promise/avoid-new` was removed from `eslint-plugin-promise@7.x`. `eslint-config-love@125.1.0` still enables it, producing a "Definition for rule not found" error at runtime. Explicitly setting it to `'off'` suppresses the error without altering any linting behaviour (the rule was being enabled but couldn't be loaded anyway).

After Step B1 removes the plugin instance conflict, verify that `@typescript-eslint/no-unnecessary-condition` and `@typescript-eslint/require-await` load correctly from `eslint-config-love`'s bundled plugin. If they still appear as "not found" (unlikely after B1), add explicit overrides as `'off'` entries using the same pattern shown above.

---

#### Step B4 — Correct `tsconfig.json` include path

**File:** `repos/event-monitoring-service/tsconfig.json`

**BEFORE** (line 25):
```json
  "include": ["src/**/*", "test/**/*", "knex/knex.module.ts", "knex/knexfile.ts", "knex/migrations/**/*.ts"],
```

**AFTER** (line 25):
```json
  "include": ["src/**/*", "tests/**/*", "knex/knex.module.ts", "knex/knexfile.ts", "knex/migrations/**/*.ts"],
```

The repository's test directory is `tests/` (confirmed on disk). The current `test/**/*` pattern matches nothing, leaving the four spec files in `tests/config-notify/` and `tests/dems-engine/` outside the TypeScript program. After this change, `tsc --noEmit` will type-check the spec files. Any type errors surfaced in the spec files should be fixed before merging this PR.

---

#### Step B5 — Update `typescript-eslint` devDependency pin

**File:** `repos/event-monitoring-service/package.json`

**BEFORE** (devDependencies):
```json
"typescript-eslint": "^8.20.0"
```

**AFTER** (devDependencies):
```json
"typescript-eslint": "^8.41.0"
```

`eslint-config-love@125.1.0` declares `typescript-eslint ^8.41.0` as a peer dependency. The direct devDependency pin of `^8.20.0` is outdated. The installed version `8.59.0` already satisfies the corrected range, so no `npm install` change is needed — this is a declaration alignment only.

---

## Test Cases

### Unit tests

No new unit tests are introduced — this is a tooling configuration fix. The existing test suite in `tests/` should be run to confirm no regressions.

After Step B4 (tsconfig include correction), run:
```bash
npx tsc --noEmit
```
Confirm no new type errors in `tests/` files. If errors appear, they predate this fix and should be corrected in a follow-up.

After all changes, run:
```bash
npm run lint:eslint
npm run lint:check
```
Both must exit 0.

### Integration / E2E test scenarios

1. Check out `origin/dev` on a clean clone.
2. Run `npm install`.
3. Run `npm run lint:eslint` — confirm exit code 0 and no errors in output.
4. Run `npm run lint:check` — confirm exit code 0 and warning count matches baseline (92 warnings).
5. Introduce a deliberate lint violation in `src/main.ts` (e.g. `console.log('test')`).
6. Run `npm run lint:eslint` — confirm the violation is detected (exit code 1, error on `no-console`).
7. Revert the deliberate violation.
8. Push a test branch, open a PR to `dev`, confirm CI lint gate passes.

### Manual / UAT checks

| # | Scenario | Steps | Expected Result |
|---|---|---|---|
| 1 | CI gate passes on clean dev | Push any commit to a branch targeting `dev`; observe CI | `npm run lint:eslint` step is green |
| 2 | Local lint matches CI | Run `npm run lint:eslint` locally on `origin/dev` | Exit code 0, zero errors |
| 3 | No rule-not-found warnings | Run `npm run lint:eslint 2>&1 \| grep "Definition for rule"` | No output |
| 4 | TypeScript covers test files | Run `npx tsc --noEmit` after Track B | No "File is not under 'rootDir'" or similar errors for `tests/` files |
| 5 | Warnings unchanged | Run `npm run lint:check 2>&1 \| grep "warning"` | Warning count consistent with baseline |
| 6 | `fix:eslint` runs cleanly | Run `npm run fix:eslint` on a clean checkout | No parse errors; only expected fixable warnings |

---

## Overall Impact of the Fix

| Area | Before | After |
|---|---|---|
| CI lint gate | Red on every PR into `dev` (66 errors) | Green on clean `origin/dev` checkout (0 errors) |
| `npm run lint:eslint` | 66 errors: 62 parse errors + 4 rule-not-found | 0 errors |
| `npm run lint:check` | 0 errors, 92 warnings | Unchanged: 0 errors, 92 warnings |
| Plugin registration | Dual `@typescript-eslint` instances; rule lookup unreliable | Single instance from `eslint-config-love`; rule lookup deterministic |
| Parser config | Deprecated `parserOptions.project`; file-discovery-sensitive | `projectService: true`; resolution correct for any invocation style |
| `tsconfig.json` test coverage | `test/**/*` matches nothing (no `test/` directory) | `tests/**/*` matches all 4 spec files |
| `typescript-eslint` devDep | `^8.20.0` (outdated relative to `eslint-config-love@125.x`) | `^8.41.0` (aligned with peer requirements) |

---

## Fix Summary

The `event-monitoring-service` CI pipeline has been failing the lint gate on `dev` because `npm run lint:eslint` runs bare `eslint` with no file path argument. ESLint discovers all TypeScript files in the project and attempts to parse them using a TypeScript project reference (`parserOptions.project`), which fails for all 62 source files when invoked without an explicit file list. A second fault causes three additional errors: the ESLint config imports the `@typescript-eslint` plugin twice — once bundled inside `eslint-config-love` and once directly — and the direct import overrides the bundled one, breaking rule lookups. A removed rule in `eslint-plugin-promise@7` adds a fourth error.

Track A is a single-line fix: the `lint:eslint` script in `package.json` gains the same file glob that `lint:check` already uses. This makes CI green immediately. The underlying config problems remain but do not surface as gate failures once the file scope is controlled.

Track B cleans up the underlying configuration debt accumulated over time. It removes the duplicate plugin import from `eslint.config.mjs`, migrates the parser configuration to the current `projectService` API, disables the removed `promise/avoid-new` rule, and corrects the `tsconfig.json` include path to point at `tests/` (the actual directory) instead of the non-existent `test/`. These changes make the ESLint setup correct by construction, so future CI failures from config drift are far less likely.

Both tracks are limited to tooling configuration files. No application logic, no API contracts, no database schemas, and no external service integrations are affected. The fix can be reviewed and merged within a single sprint.
