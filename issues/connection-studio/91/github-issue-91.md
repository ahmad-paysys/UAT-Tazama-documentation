# GitHub Issue & PR Templates — Issue #91

---

## GitHub Issue Body

### Problem

On a clean checkout of `dev`, `cd backend && npm ci && npm test` reports "226 tests passed" — but six of twelve jest suites never actually load. They fail at import time with `TypeError ... ENCRYPTION_KEY must be 32 bytes for aes-256-cbc` because `src/utils/helpers.ts` runs a hard 32-byte length check at the top of the module. Jest is configured to read env via `setupFiles: ['dotenv/config']`, which loads `.env` — a file that is git-ignored and does not exist on a fresh clone. A partial-green run masks that half the suites never executed.

Separately, `backend/package.json` still declares `@prisma/client` and `prisma`, and the root `backend:lint:eslint` script runs `npx prisma generate`, even though no source file in `backend/src/` imports Prisma. This vestigial dependency caused the initial (retracted) misdiagnosis in this issue and should be removed to prevent recurrence.

The third finding from the original report (`@tazama-lf/tcs-lib` `1.0.157-rc.0` removing `related_transaction`) is being handled by holding tcs-lib at `1.0.156-rc.x` per the rule-studio precedent and is tracked in `tazama-lf/tcs-lib#51`. It is out of scope for this fix.

### Root Cause

`src/utils/helpers.ts` performs a side-effectful, throwing validation of `ENCRYPTION_KEY` at module-import time. Combined with jest loading env from an uncommitted `.env` file, any suite that transitively imports `helpers.ts` fails to load on a fresh clone. There is no committed test-env source that would make the suite reproducible in CI or on a new contributor's machine.

### Impact

- 6 of 12 backend jest suites fail to *load* on a clean checkout (suite-load errors, not assertion failures).
- Partial-green jest runs mask the failures — 226 tests appear to pass while six suites are silently omitted.
- New contributors cannot run the full test suite without out-of-band env setup.
- Future dependency sweeps will keep tripping over the vestigial `@prisma/client` declaration.
- No CI signal today catches the reproducibility gap.

### Proposed Fix

**Track A — Test-env fix + Prisma cleanup** (config/docs, ~0.5 day, no migration, no frontend)

- Commit `backend/.env.test` with a 32-byte `ENCRYPTION_KEY` and `IV_LENGTH=16`.
- Update `backend/jest.config.ts` to load `.env.test` explicitly (via a small `test/jest.env.ts`).
- Remove `@prisma/client` and `prisma` from `backend/package.json`.
- Remove `npx prisma generate` from the root `backend:lint:eslint` script.
- Add a short section to `backend/README.md` documenting the test-env contract.

Files: `backend/.env.test` (new), `backend/jest.config.ts`, `backend/test/jest.env.ts` (new), `backend/package.json`, `package.json` (root), `backend/README.md`.

**Track B — Reproducibility hardening** (source + CI, ~1 day, no migration, no frontend)

- Move the 32-byte check in `src/utils/helpers.ts` off module-import into a lazy `getKey()` used by `encrypt`/`decrypt`.
- Add `@Length(32, 32)` to `ENCRYPTION_KEY` in `src/config/env.validation.ts` so a misconfigured production deployment fails at Nest boot.
- Add a `.github/workflows/backend-smoke.yml` clean-checkout smoke job that runs `npm ci && npm run build && npm test -- --ci`, ensuring any future regression is caught by CI.
- Audit `node-ci.yml@dev` to confirm it does not pass `--passWithNoTests` and fails on any suite load error.

Files: `backend/src/utils/helpers.ts`, `backend/src/config/env.validation.ts`, `.github/workflows/backend-smoke.yml` (new), plus a possible tweak to the reusable `node-ci.yml`.

### Acceptance Criteria

**Track A**
- [ ] `backend/.env.test` is committed with a 32-byte `ENCRYPTION_KEY` and `IV_LENGTH=16`.
- [ ] `backend/jest.config.ts` `setupFiles` loads `backend/.env.test` explicitly.
- [ ] `cd backend && npm ci && npm test` on a clean checkout reports 12 of 12 suites loaded.
- [ ] `@prisma/client` and `prisma` are removed from `backend/package.json`.
- [ ] Root `package.json` `backend:lint:eslint` no longer runs `npx prisma generate`.
- [ ] `backend/README.md` documents the test-env expectation.

**Track B**
- [ ] `src/utils/helpers.ts` no longer throws at module-import time.
- [ ] `env.validation.ts` fails Nest boot when `ENCRYPTION_KEY` is not 32 bytes.
- [ ] A clean-checkout CI smoke workflow builds and tests `backend/` with no local artefacts.
- [ ] CI fails on any jest suite load error, not only on assertion failures.

---

## PR Descriptions

### PR 1 of 2 — Track A

#### PR Title
fix(backend): make jest reproducible from clean checkout; remove vestigial prisma dep

#### PR Body

**Summary**
- Commits `backend/.env.test` with a non-secret 32-byte `ENCRYPTION_KEY` and `IV_LENGTH=16`.
- Adds `backend/test/jest.env.ts` and updates `jest.config.ts` to load `.env.test` explicitly.
- Removes `@prisma/client` and `prisma` from `backend/package.json` — no source file imports either.
- Removes the no-op `npx prisma generate` step from the root `backend:lint:eslint` script.
- Adds a short "Running Tests" section to `backend/README.md` documenting the test-env contract.

**Test Plan**
- [ ] Fresh clone: `cd backend && npm ci && npm test` reports 12 of 12 suites loaded, no `ENCRYPTION_KEY` errors.
- [ ] `npm run lint` from the repo root still passes.
- [ ] `grep -n prisma backend/package.json` returns nothing.
- [ ] `grep -n "prisma generate" package.json` returns nothing.

**Related Issue**
Closes part of #91 (Track A).

### PR 2 of 2 — Track B

#### PR Title
harden(backend): defer ENCRYPTION_KEY validation and add clean-checkout smoke CI

#### PR Body

**Summary**
- Moves the 32-byte length check in `src/utils/helpers.ts` off module-import into a lazy `getKey()`; `encrypt`/`decrypt` now perform the check on first use.
- Adds `@Length(32, 32)` to `ENCRYPTION_KEY` in `env.validation.ts` so misconfigured production deployments fail at Nest boot.
- Adds `.github/workflows/backend-smoke.yml`: clean-runner `npm ci && npm run build && npm test -- --ci`.
- Adds a unit test asserting that `helpers.ts` no longer throws at import.

**Test Plan**
- [ ] `npm test` from `backend/` passes locally.
- [ ] Running `node -e "require('./dist/utils/helpers')"` with no `ENCRYPTION_KEY` set does not throw.
- [ ] Starting Nest with `ENCRYPTION_KEY=short` fails boot with a clear validator error.
- [ ] The new `backend-smoke.yml` workflow runs green on this PR.

**Related Issue**
Closes #91 (Track B). Requires Track A to be merged first.

**Sequencing Note**
Must merge after Track A — the smoke workflow depends on the committed `.env.test`.
