# GitHub Issue & PR Templates — Issue #47

---

## GitHub Issue Body

### Problem

The `npm run lint:eslint` script — the command executed by the reusable `node-ci.yml@dev` CI workflow — fails on a clean `origin/dev` checkout with 66 errors. Every pull request targeting `dev` fails the CI lint gate regardless of branch content. The same codebase passes `npm run lint:check` with zero errors and 92 warnings, so application code is not at fault.

### Root Cause

Two independent faults compound into 66 errors:

1. **Script scope mismatch (62 errors):** `lint:eslint` runs bare `eslint` with no file path argument. ESLint's flat config file discovery walks the project and processes all 62 `.ts` files in `src/`. The `@typescript-eslint/parser` is configured with `parserOptions.project: ['./tsconfig.json']`, which requires building a TypeScript program. When ESLint discovers files by walking the directory rather than receiving them as explicit arguments, the TypeScript project resolution fails for all discovered files, producing parse errors on each one. `lint:check` escapes this because it passes files explicitly: `eslint "{src,apps,libs,test}/**/*.ts"`.

2. **Plugin instance conflict and removed rule (4 errors):** `eslint.config.mjs` imports `@typescript-eslint/eslint-plugin` directly and then overrides the plugin namespace registered by `eslint-config-love`, creating two different plugin instances. Rules enabled by `eslint-config-love@125.1.0` are looked up against the override instance, which does not carry all of them, producing "Definition for rule not found" for `@typescript-eslint/no-unnecessary-condition` and `@typescript-eslint/require-await`. Additionally, `eslint-plugin-promise@7` removed the `avoid-new` rule, but `eslint-config-love` still enables it, adding a third rule-not-found error.

A secondary issue: `tsconfig.json` includes `test/**/*` but the test directory on disk is named `tests/` (with an `s`). No `test/` directory exists. The spec files in `tests/` are therefore outside the TypeScript project.

### Impact

- Every PR into `dev` fails CI without any changes to application code being the cause
- Local `lint:check` passes while CI fails, creating confusion and eroding trust in the CI pipeline
- TypeScript type-checking does not cover the four spec files in `tests/` (tsconfig path mismatch)
- Future `eslint-config-love` updates may expose additional rule-not-found errors due to the dual-plugin registration

### Proposed Fix

**Track A — Scope the bare eslint script (immediate, low risk):**
Add the same file glob to `lint:eslint` that `lint:check` already uses. Change `package.json` line 21 from `"eslint"` to `"eslint \"{src,apps,libs,test}/**/*.ts\""`. This unblocks CI immediately. Only `package.json` changes. No config changes, no dependency changes, no schema changes.

- Files: `package.json` (1 line)
- Migration required: No
- Estimated effort: 15 minutes

**Track B — Reconcile flat config with installed plugin versions (follow-up cleanup):**
Remove the duplicate `@typescript-eslint/eslint-plugin` import from `eslint.config.mjs`, migrate `parserOptions.project` to `parserOptions.projectService: true` (the current API), add an explicit `'promise/avoid-new': 'off'` override to suppress the removed-rule error, correct `tsconfig.json` include from `test/**/*` to `tests/**/*`, and update the `typescript-eslint` devDependency pin from `^8.20.0` to `^8.41.0`.

- Files: `eslint.config.mjs`, `tsconfig.json`, `package.json` (~10 lines total)
- Migration required: No
- Estimated effort: 60 minutes

### Acceptance Criteria

- [ ] `npm run lint:eslint` exits with code 0 on a clean `origin/dev` checkout
- [ ] The CI `node-ci.yml@dev` lint step passes on `dev` without changes to application code
- [ ] `npm run lint:check` continues to pass with 0 errors (92 warnings)
- [ ] No "Definition for rule X was not found" warnings appear in ESLint output (Track B)
- [ ] `eslint.config.mjs` does not contain a direct `@typescript-eslint/eslint-plugin` import (Track B)
- [ ] `tsconfig.json` include contains `tests/**/*` not `test/**/*` (Track B)
- [ ] `tsc --noEmit` succeeds across `src/` and `tests/` with no new errors (Track B)

---

## PR Descriptions

---

### Track A PR — Scope lint:eslint to match lint:check

#### PR Title

```
fix(lint): scope lint:eslint script to match lint:check file glob
```

#### PR Body

**Summary:**
- `npm run lint:eslint` currently runs bare `eslint` with no file path argument, causing ESLint to discover all 62 `.ts` files in `src/` and fail to parse them under `parserOptions.project` (66 errors total)
- The CI `node-ci.yml@dev` workflow runs `npm run lint:eslint`, making the lint gate permanently red on `dev`
- This PR adds the same `"{src,apps,libs,test}/**/*.ts"` glob to `lint:eslint` that `lint:check` already uses, giving the TypeScript parser explicit file paths and eliminating all parse errors
- Also updates `fix:eslint` from `"**/*.ts"` to `"{src,apps,libs,test}/**/*.ts"` for consistency
- No application code changes, no config file changes, no dependency changes

**Test Plan:**
- [ ] Check out `origin/dev` on a clean clone
- [ ] Run `npm install`
- [ ] Run `npm run lint:eslint` — confirm exit code 0 and zero errors in output
- [ ] Run `npm run lint:check` — confirm exit code 0 and 92 warnings (unchanged)
- [ ] Push this branch; confirm CI `node-ci.yml@dev` lint step is green
- [ ] Introduce a deliberate `console.log` in `src/main.ts`; confirm `npm run lint:eslint` catches it (exit code 1); revert

**Related Issue:** https://github.com/tazama-lf/event-monitoring-service/issues/47

---

### Track B PR — Reconcile ESLint flat config with installed plugin versions

#### PR Title

```
fix(eslint): remove dual-plugin import, migrate to projectService, fix tsconfig include
```

#### PR Body

**Summary:**
- Removes the direct `import tsEslint from '@typescript-eslint/eslint-plugin'` from `eslint.config.mjs` (line 4) and the corresponding plugin override (line 19) that was creating two conflicting `@typescript-eslint` plugin instances; rules are now loaded exclusively from `eslint-config-love`'s bundled instance
- Replaces deprecated `parserOptions.project: ['./tsconfig.json']` with `parserOptions.projectService: true` and `tsconfigRootDir: import.meta.dirname` — the current `typescript-eslint` recommendation for flat config, which resolves the TypeScript program correctly regardless of invocation style
- Adds `'promise/avoid-new': 'off'` to the rules block to suppress the "Definition for rule not found" error caused by `eslint-plugin-promise@7` having removed this rule while `eslint-config-love@125.x` still enables it
- Corrects `tsconfig.json` include from `"test/**/*"` to `"tests/**/*"` to match the actual test directory on disk (no `test/` directory exists; the directory is `tests/`)
- Updates `typescript-eslint` devDependency from `^8.20.0` to `^8.41.0` to align with `eslint-config-love@125.1.0` peer requirements

**Test Plan:**
- [ ] Run `npm run lint:eslint` (bare command) — confirm exit code 0 and zero errors
- [ ] Run `npm run lint:check` — confirm exit code 0 and warning count consistent with baseline
- [ ] Run `npm run lint:eslint 2>&1 | grep "Definition for rule"` — confirm no output
- [ ] Run `npx tsc --noEmit` — confirm no new type errors from `tests/` files being added to the project
- [ ] Push this branch; confirm CI `node-ci.yml@dev` lint step remains green
- [ ] Review ESLint output for any new warnings introduced by the `projectService` migration

**Related Issue:** https://github.com/tazama-lf/event-monitoring-service/issues/47

**Sequencing Note:** This PR must be merged after Track A (the `package.json` glob fix). Track A unblocks CI; Track B cleans up the underlying configuration debt. Both PRs touch `package.json` but on different lines (line 21 for Track A, devDependencies for Track B) — no merge conflict expected if Track A lands first.
