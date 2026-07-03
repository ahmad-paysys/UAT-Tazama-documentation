# Issue #47 — User Stories: CI lint gate fails on dev with 66 errors

**Assigned developer:** 1 developer
**Tracks:** Track A (immediate fix) → Track B (full config reconciliation)
**Merge order:** Track A ships independently and immediately. Track B follows after Track A is merged.

---

## US-47-01 — Scope lint:eslint to unblock CI gate

**Title:** Scope the bare `eslint` command in `lint:eslint` to the correct file glob

**Body:**
The `npm run lint:eslint` script currently runs bare `eslint` with no file path argument. When the CI workflow (`node-ci.yml@dev`) executes this script, ESLint discovers all 62 `.ts` files in `src/` by walking the project directory and attempts to parse each one using `@typescript-eslint/parser` with `parserOptions.project: ['./tsconfig.json']`. This invocation context causes the TypeScript project resolver to fail for all discovered files, producing 62 parse errors. Four additional errors (rule-not-found) bring the total to 66. The CI lint gate is permanently red on `dev`, blocking all PRs.

The fix is to add the same file glob that `lint:check` already uses — `"{src,apps,libs,test}/**/*.ts"` — to the `lint:eslint` script in `package.json` (line 21). Passing files explicitly instead of relying on ESLint's directory walker gives the TypeScript parser a stable resolution context and eliminates all parse errors. The `fix:eslint` script (line 13) should be updated in the same PR from `"**/*.ts"` to `"{src,apps,libs,test}/**/*.ts"` for consistency.

**Scope (Track A):**
- File: `repos/event-monitoring-service/package.json`
- Lines: 13, 21
- Change: Add `"{src,apps,libs,test}/**/*.ts"` glob argument to `lint:eslint` and `fix:eslint` scripts
- No ESLint config changes, no dependency changes, no schema migration, no frontend changes

**Acceptance Criteria:**
- [ ] `npm run lint:eslint` exits with code 0 on a clean `origin/dev` checkout
- [ ] `package.json` line 21 contains `"{src,apps,libs,test}/**/*.ts"` as the file argument
- [ ] The CI `node-ci.yml@dev` lint step passes on `dev` without touching application code
- [ ] `npm run lint:check` continues to exit with 0 errors and 92 warnings unchanged
- [ ] Running `npm run lint:eslint` and `npm run lint:check` on `origin/dev` produces identical zero-error results
- [ ] A deliberate `console.log` introduced into `src/main.ts` is caught by `npm run lint:eslint` (validates scope is correct)

**Testing:**
- Check out `origin/dev` on a clean clone; run `npm install`; run `npm run lint:eslint`; confirm exit code 0 and zero errors in stdout
- Push a branch to confirm CI `node-ci.yml@dev` lint gate is green
- Introduce a deliberate lint violation in `src/main.ts`; confirm `npm run lint:eslint` catches it; revert

---

## US-47-02 — Remove dual @typescript-eslint plugin registration

**Title:** Remove the direct `@typescript-eslint/eslint-plugin` import that conflicts with `eslint-config-love`'s bundled instance

**Body:**
`eslint.config.mjs` imports `@typescript-eslint/eslint-plugin` directly at line 4 (`import tsEslint from '@typescript-eslint/eslint-plugin'`) and then at line 19 overrides the `@typescript-eslint` plugin namespace that was spread from `...eslintStandard.plugins` with this directly-imported instance. `eslint-config-love@125.1.0` spreads its own bundled plugin instance via `...eslintStandard.rules`, but the registered plugin at runtime is the override instance. Any rule that love enables from its own instance but that is absent from the override instance produces "Definition for rule not found" at lint time. This affects `@typescript-eslint/no-unnecessary-condition` and `@typescript-eslint/require-await`.

The fix removes the `import tsEslint` line (line 4) and the `['@typescript-eslint']: tsEslint` entry (line 19) from the plugins block. The `@typescript-eslint` plugin namespace is then provided solely by `...eslintStandard.plugins`, matching the instance that `...eslintStandard.rules` was built against. All rule lookups become deterministic.

**Scope (Track B — step 1):**
- File: `repos/event-monitoring-service/eslint.config.mjs`
- Lines: 4 (import removal), 19 (plugin override removal)
- Change type: Remove 2 lines; no new code added
- `@typescript-eslint/eslint-plugin` can subsequently be removed from `devDependencies` since it is now consumed transitively through `eslint-config-love`

**Acceptance Criteria:**
- [ ] `eslint.config.mjs` does not contain `import tsEslint from '@typescript-eslint/eslint-plugin'`
- [ ] The plugins block in `eslint.config.mjs` does not contain `['@typescript-eslint']: tsEslint`
- [ ] `npm run lint:eslint 2>&1 | grep "no-unnecessary-condition"` produces no output
- [ ] `npm run lint:eslint 2>&1 | grep "require-await"` produces no output
- [ ] All existing TypeScript lint rules continue to fire correctly (verify by introducing a deliberate `any` cast or unused variable)

**Testing:**
- Run `npm run lint:eslint` after the change; confirm no "Definition for rule not found" lines in output
- Confirm `@typescript-eslint` rules still apply by checking that a known violation (e.g., use of `any` where restricted) is still reported as a warning or error

---

## US-47-03 — Migrate parserOptions.project to projectService

**Title:** Replace deprecated `parserOptions.project` with `parserOptions.projectService: true`

**Body:**
`eslint.config.mjs` line 27 uses `parserOptions.project: ['./tsconfig.json']` — the legacy API for TypeScript project loading in `@typescript-eslint/parser`. When ESLint discovers files by walking the project directory (bare `eslint` invocation), the project path is resolved relative to the file being linted rather than the ESLint config location, causing resolution failures for all discovered files. The current API, `parserOptions.projectService: true`, uses the TypeScript Language Service to find the appropriate `tsconfig.json` for each file automatically, without path ambiguity.

The fix replaces `project: ['./tsconfig.json']` with `projectService: true` and adds `tsconfigRootDir: import.meta.dirname` to anchor the root directory to the location of `eslint.config.mjs`.

**Scope (Track B — step 2):**
- File: `repos/event-monitoring-service/eslint.config.mjs`
- Lines: 26–28 (parserOptions block)
- Change type: Replace 1 property key-value pair; add 1 new property

**Acceptance Criteria:**
- [ ] `eslint.config.mjs` parserOptions block contains `projectService: true` and not `project: [...]`
- [ ] `eslint.config.mjs` parserOptions block contains `tsconfigRootDir: import.meta.dirname`
- [ ] `npm run lint:eslint` (bare command) exits 0 with no parse errors
- [ ] Type-aware rules (e.g., `@typescript-eslint/no-floating-promises`) continue to be evaluated correctly

**Testing:**
- Run `npm run lint:eslint` (bare, no file argument) and confirm zero errors
- Compare warning count with `npm run lint:check` — both should be 0 errors; warning counts should be consistent

---

## US-47-04 — Disable removed promise/avoid-new rule

**Title:** Suppress "Definition for rule not found" error for `promise/avoid-new` removed in `eslint-plugin-promise@7`

**Body:**
`eslint-config-love@125.1.0` enables the `promise/avoid-new` rule, but `eslint-plugin-promise@7.2.1` (the version installed, as confirmed in `package-lock.json`) removed this rule. At lint time, ESLint reports "Definition for rule 'promise/avoid-new' was not found" as an error. Since the rule no longer exists in the installed plugin, it cannot be applied; the only correct resolution is to override it to `'off'` in `eslint.config.mjs` to suppress the error without affecting any other promise-related rules.

**Scope (Track B — step 3):**
- File: `repos/event-monitoring-service/eslint.config.mjs`
- Location: rules block (after existing overrides, before closing brace)
- Change type: Add one rule override line: `'promise/avoid-new': 'off'`

**Acceptance Criteria:**
- [ ] `eslint.config.mjs` rules block contains `'promise/avoid-new': 'off'`
- [ ] `npm run lint:eslint 2>&1 | grep "avoid-new"` produces no output
- [ ] All other `promise/*` rules from `eslint-config-love` continue to apply unchanged
- [ ] No new promise-related lint errors appear in `src/` files after this change

**Testing:**
- Run `npm run lint:eslint` and confirm no "Definition for rule 'promise/avoid-new' was not found" in output
- Verify remaining `promise/*` rules still fire by checking that a `new Promise()` usage (if any exists in `src/`) is still evaluated by the linter

---

## US-47-05 — Correct tsconfig include path from test/ to tests/

**Title:** Fix `tsconfig.json` include path to match the actual `tests/` directory

**Body:**
`tsconfig.json` line 25 includes `"test/**/*"` but the test directory on disk is `tests/` (with an `s`). There is no `test/` directory in the repository (confirmed by directory listing). The four spec files at `tests/config-notify/config-notify.controller.spec.ts`, `tests/config-notify/config-notify.service.spec.ts`, `tests/dems-engine/dems-engine.controller.spec.ts`, and `tests/dems-engine/dems-engine.service.spec.ts` are therefore not included in the TypeScript project. This means `tsc --noEmit` does not type-check these files and any type errors in tests are silently missed.

The fix is a single-character change: `"test/**/*"` becomes `"tests/**/*"` in the include array. After this change, `tsc` will type-check all four spec files. Any type errors surfaced should be resolved before merging.

**Scope (Track B — step 4):**
- File: `repos/event-monitoring-service/tsconfig.json`
- Line: 25
- Change type: One character (`test` → `tests` in the include array)
- No dependency changes, no application code changes

**Acceptance Criteria:**
- [ ] `tsconfig.json` line 25 contains `"tests/**/*"` not `"test/**/*"`
- [ ] `npx tsc --noEmit` runs without errors (fix any pre-existing type errors in `tests/` files before merging)
- [ ] The four spec files in `tests/` are included in the TypeScript program (verify with `tsc --listFiles | grep tests`)
- [ ] No application `src/` files are affected by this change

**Testing:**
- Run `npx tsc --noEmit` before and after; compare error output
- Run `npx tsc --listFiles | grep tests/` and confirm the four spec files are listed
- Confirm Jest still finds and runs tests: `npm test`

---

## US-47-06 — Align typescript-eslint devDependency with eslint-config-love peer requirements

**Title:** Update `typescript-eslint` devDependency from `^8.20.0` to `^8.41.0` to match `eslint-config-love@125.x` peer requirements

**Body:**
`package.json` declares `"typescript-eslint": "^8.20.0"` as a devDependency. `eslint-config-love@125.1.0` (confirmed in `package-lock.json`) declares `typescript-eslint ^8.41.0` as a peer dependency. The installed version is `8.59.0`, which satisfies both ranges, so there is no functional breakage today. However, the declared minimum `^8.20.0` is misleading: it would allow npm to install a version that is known to be incompatible with `eslint-config-love@125.x`. Aligning the declared range prevents future `npm install` runs (e.g., on a fresh CI node or after `npm dedupe`) from resolving to an incompatible version.

**Scope (Track B — step 5):**
- File: `repos/event-monitoring-service/package.json`
- Location: devDependencies block
- Change type: Version range update from `"^8.20.0"` to `"^8.41.0"` (no install change required; `8.59.0` is already installed)

**Acceptance Criteria:**
- [ ] `package.json` devDependencies contains `"typescript-eslint": "^8.41.0"` or higher
- [ ] `npm install` runs without peer dependency warnings for `typescript-eslint`
- [ ] `npm run lint:eslint` continues to exit 0 after the version range update
- [ ] `npm ls typescript-eslint` confirms the installed version satisfies the updated range

**Testing:**
- Run `npm install` after updating the range; confirm no warnings
- Run `npm run lint:eslint` and `npm run lint:check`; confirm both exit 0
- Run `npm ls typescript-eslint` to confirm resolved version
