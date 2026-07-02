# Issue #91 — Backend Build & Tests Not Reproducible from Clean Checkout

**Repository:** tazama-lf/connection-studio  
**Issue:** [#91 — Backend build & tests not reproducible from clean checkout - quality-gate gap (Prisma schema absent, test ENV required, tcs-lib RC breaking)](https://github.com/tazama-lf/connection-studio/issues/91)  
**Author:** Justus-at-Tazama  
**State:** Open  
**Report Date:** 2026-07-02  

---

## Executive Summary

A routine dependency-security refresh on `backend/` surfaced four independent failures that collectively mean the backend cannot be built or fully tested from a clean checkout without machine-local artifacts and undocumented environment variables. The CI pipeline may be passing against a non-reproducible state. This is not a single bug — it is a quality-gate gap that has allowed a broken build baseline to persist undetected on `dev`.

The four findings are:

1. **No `schema.prisma` in the repository** — the Prisma client cannot be generated from a clean clone, so `nest build` fails on any machine without a pre-existing local `generated/prisma` artifact.
2. **Six of twelve test suites fail to load without an undocumented `ENCRYPTION_KEY`** — a partial jest run (226 tests, 6 suites) can silently appear green while masking half the test coverage.
3. **`@tazama-lf/tcs-lib` `1.0.157-rc.0` is a breaking change** against current `dev` code — `related_transaction` was removed from `Config` / `CreateConfigDto`, breaking `config.service.ts` at the type-check level.
4. **The CI workflow (`node-ci.yml@dev`) is not confirmed to build and fully test `backend/`** — it may be skipping the generate step or feeding a pre-baked schema artifact.

---

## How It Works Today — Confirmed in Code

### Finding 1 — Missing Prisma Schema

The Prisma client is generated into `backend/generated/prisma`, which is excluded from version control by `.gitignore`:

```
# backend/.gitignore
/generated/prisma
```

There is **no `schema.prisma` anywhere in the repository** (confirmed by full-tree search). Without the schema, `npx prisma generate` has nothing to generate from.

The root `package.json` scripts make this visible:

```json
"backend:lint:eslint": "cd backend && npm ci && npx prisma generate && npm run lint",
"backend:build":       "cd backend && npm ci && npm run build"
```

- `backend:lint:eslint` calls `prisma generate` — but there is no schema, so generate fails on a clean clone.
- `backend:build` **does not call generate at all** — it relies on the `generated/prisma` folder already existing locally.

`nest build` type-checks against types re-exported through `@tazama-lf/tcs-lib` and the generated Prisma client. A clean checkout with no `generated/prisma` folder cannot pass the TypeScript compilation step.

**Net effect:** The backend only builds on a machine that already has a locally-generated Prisma client from an out-of-band source. That artifact is invisible to CI and to any new contributor.

### Finding 2 — Undocumented `ENCRYPTION_KEY` Required for Tests

Running `npm test` in `backend/` executes jest across 12 suites. Six suites fail **at load time** (before any assertions run) with:

```
TypeError: ENCRYPTION_KEY must be 32 bytes for aes-256-cbc
```

The failing suites include:
- `config/config.service.spec.ts`
- `job/job.service.spec.ts`
- `dry-run/dry-run.service.spec.ts`
- (and three others)

There is no `.env.test`, no jest `setupFiles`, and no documented requirement that sets a 32-byte `ENCRYPTION_KEY` before running tests. The remaining 6 suites (226 tests) pass cleanly, so a CI run that does not surface the load-time failures can report `226 passed` as if the test run were complete.

**Net effect:** Half the backend test coverage is silently absent on any machine without the correct `ENCRYPTION_KEY` pre-set. A green-looking partial run masks the gap entirely.

### Finding 3 — `tcs-lib` `1.0.157-rc.0` Removes `related_transaction`

Current `dev` pins `@tazama-lf/tcs-lib` at `1.0.156`. That version's `Config` and `CreateConfigDto` types include `related_transaction`.

`1.0.157-rc.0` **removes `related_transaction`** from both types. The current code in `src/config/config.service.ts:178` reads `dto.related_transaction` and writes it into the Prisma create input:

```ts
// config.service.ts:178
data: {
  ...dto,
  related_transaction: dto.related_transaction,  // broken on 1.0.157-rc.0
  ...
}
```

Bumping tcs-lib to `1.0.157-rc.0` produces two TypeScript errors:

```
TS2353: Object literal may only specify known properties,
        'related_transaction' does not exist in type 'Omit<Config, ...>'

TS2339: Property 'related_transaction' does not exist on type 'CreateConfigDto'
```

Reverting only the tcs-lib pin from `1.0.157-rc.0` back to `1.0.156` clears both errors. The RC was not adopted blindly — but any future realign to `1.0.157-rc.x` or later will require source changes here first.

### Finding 4 — CI Quality-Gate Gap

Findings 1 and 2 combine to raise a structural concern about the current CI configuration (`node-ci.yml@dev`):

- If CI runs `npm run build` (which does not call `prisma generate`), it requires a pre-existing `generated/prisma` folder. That folder is git-ignored. Where does CI get it?
- If CI runs `npm test` without setting `ENCRYPTION_KEY`, 6 suites fail to load. Is the partial pass (`226 tests, 6 failed suites`) being reported as green?

One of three things must be true:
1. CI is fed a pre-baked schema/client artifact from outside the repo (invisible, fragile).
2. CI is not running the build or full test suite against `backend/` at all.
3. The `ENCRYPTION_KEY` failure is being suppressed or ignored in the test reporter config.

Any of these represents a gap between what CI reports and what the codebase's actual health is.

---

## Root Cause Analysis

The four findings share a single root: **the `backend/` package was developed against machine-local artifacts that were never committed or documented, and CI was not configured to reproduce the build from scratch.**

The individual causes:

| Finding | Root Cause |
|---|---|
| Missing Prisma schema | `schema.prisma` was never committed; `generated/prisma` was git-ignored without providing a way to regenerate it cleanly |
| Missing `ENCRYPTION_KEY` | A required test environment variable was never added to a `.env.test` or jest `setupFiles`; its absence causes hard failures, not skips |
| `tcs-lib` RC breakage | `related_transaction` was removed from the public type surface in the RC without a corresponding update to `config.service.ts` |
| CI gap | `node-ci.yml` was not audited against a clean-state build; it likely runs against a workspace that already has `generated/prisma` present |

---

## Blast Radius — Affected Files

### Directly Broken (build or tests fail without fix)

| File | Problem |
|---|---|
| `backend/prisma/` (missing) | No `schema.prisma` → `prisma generate` cannot run |
| `backend/generated/prisma/` (git-ignored) | Generated client absent on clean clone |
| `backend/package.json` `backend:build` script | Does not call `prisma generate` before `nest build` |
| `backend/src/config/config.service.ts:178` | Reads `dto.related_transaction` — broken on `tcs-lib` `1.0.157-rc.0` |
| `backend/src/config/config.service.spec.ts` | Fails to load — `ENCRYPTION_KEY` not set |
| `backend/src/job/job.service.spec.ts` | Fails to load — `ENCRYPTION_KEY` not set |
| `backend/src/dry-run/dry-run.service.spec.ts` | Fails to load — `ENCRYPTION_KEY` not set |
| (3 further test suites) | Fail to load — `ENCRYPTION_KEY` not set |

### Indirectly Affected (CI / developer experience)

| File | Problem |
|---|---|
| `.github/workflows/node-ci.yml` | May be building against a non-clean state or ignoring suite-load failures |
| `backend/.gitignore` | Excludes `generated/prisma` without providing a documented regeneration path |
| Repository root `README` / contributor docs | No documented `ENCRYPTION_KEY` requirement for running tests |

---

## Side-Effect Map

| What breaks if left unfixed | Impact |
|---|---|
| New contributor clones repo and runs `npm run build` | Build fails silently (no schema, no generated client); no actionable error message in the README |
| CI builds `dev` from clean state | Either fails (if truly clean) or passes against a stale artifact (if pre-baked) — both outcomes are wrong |
| Any `npm test` run without `ENCRYPTION_KEY` | 6 suites fail to load; 226 tests pass; reporter may show this as a passing run |
| Adoption of `tcs-lib` `1.0.157-rc.0` without source update | Two TypeScript build errors in `config.service.ts`; build cannot compile |
| Schema divergence | If the DB schema evolves and `schema.prisma` is never committed, drift is undetectable until a production deployment fails |

---

## Recommended Actions (from Issue Author)

1. **Commit the Prisma schema** (or document a reproducible `prisma db pull` / schema-fetch step) so `prisma generate` can run deterministically in CI and on fresh clones. Update `backend:build` to run `prisma generate` before `nest build`.
2. **Provide test env setup** — add a `.env.test` or jest `setupFiles` that supplies a 32-byte `ENCRYPTION_KEY` — so all 12 suites run. Fail CI if any suite fails to load.
3. **Audit `node-ci.yml@dev`** — confirm it builds and fully tests `backend/` from a clean state; close any gap that allowed a non-reproducible build to merge to `dev`.
4. **Do not adopt `tcs-lib` `1.0.157-rc.0`** until `related_transaction` usage in `config.service.ts:178` is reconciled with the new type surface. Document the breaking change for the backend team.

---

## Effort Assessment

### Track A — Unblock Builds and Tests (1–2 days)

| Step | Files | Effort |
|---|---|---|
| A1: Commit or document `schema.prisma` | `backend/prisma/schema.prisma` | 2–4h (schema recovery or `db pull` + commit) |
| A2: Update `backend:build` script to run `prisma generate` | `backend/package.json` | 15min |
| A3: Add `.env.test` or jest `setupFiles` with a test-safe `ENCRYPTION_KEY` | New file or `jest.config.ts` | 1h |
| A4: Verify all 12 suites load and pass | Local + CI | 1–2h |

**Outcome:** Clean-checkout builds and full test runs work. CI reports accurate pass/fail.

### Track B — CI Hardening (1 day)

| Step | Files | Effort |
|---|---|---|
| B1: Audit `node-ci.yml` build step for `backend/` | `.github/workflows/node-ci.yml` | 2h |
| B2: Add CI step that fails on any jest suite load failure | CI config | 1h |
| B3: Confirm `tcs-lib` pin policy and document breaking-change process | `backend/package.json`, contributor docs | 1h |

**Outcome:** CI cannot silently pass a non-reproducible or partially-tested build.

### Recommended Sequencing

1. **Ship Track A first** — unblocks all contributors and makes CI trustworthy.
2. **Track B immediately after** — closes the CI gap before the next `dev` merge.
3. **`tcs-lib` `1.0.157-rc.0`** — do not adopt until `config.service.ts` is updated to remove `related_transaction` usage; this is a separate, small PR.

---

## Acceptance Criteria

### Track A

- [ ] `git clone` + `npm ci` + `npm run build` (in `backend/`) completes without error on a machine with no pre-existing `generated/prisma` folder.
- [ ] `npm test` (in `backend/`) runs all 12 suites; 0 suites fail to load.
- [ ] No undocumented environment variable is required to run the test suite.

### Track B

- [ ] CI (`node-ci.yml@dev`) builds `backend/` from a clean state (no pre-baked artifacts).
- [ ] CI fails if any jest suite fails to load, not only if assertions fail.
- [ ] `@tazama-lf/tcs-lib` `1.0.157-rc.0` (or later) is not merged to `dev` until `config.service.ts:178` no longer references `related_transaction`.
- [ ] The `schema.prisma` (or a documented, automated schema-fetch step) is present in version control and kept in sync with the database schema.

---
