# User Stories — Issue #91

---

## US-91-1 — Commit `.env.test` for backend jest suite

**Body**
Six of twelve backend jest suites currently fail to load on a clean checkout because `src/utils/helpers.ts` throws at import when `ENCRYPTION_KEY` is not a 32-byte string, and jest reads env only from an uncommitted `.env` file. Committing a `backend/.env.test` file with a non-secret 32-byte key and pointing jest at it makes the full suite reproducible from any fresh clone.

**Scope**
- Track: A
- Files: `backend/.env.test` (new), `backend/test/jest.env.ts` (new), `backend/jest.config.ts`
- Change: config-only; no source-code behaviour change

**Acceptance Criteria**
- [ ] `backend/.env.test` exists in the repo with `ENCRYPTION_KEY` of exactly 32 bytes and `IV_LENGTH=16`.
- [ ] `backend/test/jest.env.ts` loads `.env.test` via `dotenv.config({ path: ... })`.
- [ ] `backend/jest.config.ts` `setupFiles` references `<rootDir>/test/jest.env.ts` and no longer relies on `dotenv/config`'s implicit `.env` lookup.
- [ ] `cd backend && npm ci && npm test` on a fresh clone reports 12 of 12 suites loaded.

**Testing**
- Fresh clone of `dev`, then `cd backend && npm ci && npm test`. Confirm no `ENCRYPTION_KEY must be 32 bytes` errors and 12 of 12 suites reported.
- Temporarily rename `.env.test` and re-run — expect a loud, obvious failure (not a silent partial-green).

---

## US-91-2 — Remove vestigial `@prisma/client` dependency

**Body**
`backend/package.json` declares `@prisma/client` and `prisma`, and the root `backend:lint:eslint` script runs `npx prisma generate` — but no source file in `backend/src/` imports Prisma anywhere. This dead weight caused the initial misdiagnosis in issue #91 and should be removed so future dependency sweeps do not repeat the confusion.

**Scope**
- Track: A
- Files: `backend/package.json`, `package.json` (root)
- Change: dependency removal; no source-code change

**Acceptance Criteria**
- [ ] `@prisma/client` is removed from `backend/package.json` dependencies.
- [ ] `prisma` is removed from `backend/package.json` devDependencies.
- [ ] Root `package.json` `backend:lint:eslint` no longer contains `npx prisma generate`.
- [ ] `npm ci` and `npm run build` in `backend/` still succeed.

**Testing**
- `grep -n prisma backend/package.json` returns nothing.
- `grep -n "prisma generate" package.json` returns nothing.
- `cd backend && npm ci && npm run build` completes without errors.

---

## US-91-3 — Document the test-env contract in the backend README

**Body**
New contributors have no way to know that `backend/.env.test` is authoritative for jest and that any new module reading env at import time must add a placeholder there. A short README section makes the contract explicit.

**Scope**
- Track: A
- Files: `backend/README.md`
- Change: documentation

**Acceptance Criteria**
- [ ] `backend/README.md` contains a "Running Tests" section that names `.env.test`, explains it is a committed non-secret, and instructs contributors to add placeholders for any new env vars read at import time.

**Testing**
- Manual review of the README section.

---

## US-91-4 — Defer `ENCRYPTION_KEY` validation off module-import

**Body**
`src/utils/helpers.ts` currently evaluates `ENCRYPTION_KEY` and throws at the module's top level if the value is missing or not 32 bytes. This side effect is what forced the workaround in US-91-1: any file that transitively imports `helpers.ts` fails to *load* when env is missing. Moving the check into a lazy `getKey()` used by `encrypt`/`decrypt` removes the failure mode at the design level.

**Scope**
- Track: B
- Files: `backend/src/utils/helpers.ts`
- Change: refactor `helpers.ts` to defer key resolution to first use

**Acceptance Criteria**
- [ ] `helpers.ts` has no top-level throw. Importing the module with `ENCRYPTION_KEY` unset does not throw.
- [ ] `encrypt` and `decrypt` throw a clear error on first call if `ENCRYPTION_KEY` is missing or wrong-length.
- [ ] A new unit test in `backend/test/utils/helpers.spec.ts` asserts both behaviours.
- [ ] All existing suites still pass.

**Testing**
- `node -e "require('./dist/utils/helpers')"` succeeds with no env set.
- New unit test: importing succeeds without env; calling `encrypt` with no env throws with a clear message.
- Existing `job`, `dry-run`, `sftp`, `notification`, `scheduler` suites still pass.

---

## US-91-5 — Enforce `ENCRYPTION_KEY` length at Nest boot

**Body**
Even with the deferred check in US-91-4, a wrong-length `ENCRYPTION_KEY` in production would only be detected the first time `encrypt` is called. Adding `@Length(32, 32)` to `ENCRYPTION_KEY` in the env validator ensures the failure surfaces at Nest boot instead — before any traffic is served.

**Scope**
- Track: B
- Files: `backend/src/config/env.validation.ts`
- Change: add validator decorator

**Acceptance Criteria**
- [ ] `ENCRYPTION_KEY` in `env.validation.ts` has `@Length(32, 32)` with a clear error message.
- [ ] Starting Nest with `ENCRYPTION_KEY=short` fails boot with a message that names `ENCRYPTION_KEY` and states the required length.
- [ ] Starting Nest with a valid 32-byte key boots as before.

**Testing**
- Start locally with `ENCRYPTION_KEY=short npm run start`. Expect a boot-time validator error.
- Start locally with the value from `.env.test`. Expect normal boot.

---

## US-91-6 — Clean-checkout backend smoke workflow

**Body**
There is no CI signal today that would fail if the backend regresses to a non-reproducible state. Adding a `backend-smoke.yml` GitHub workflow that runs `npm ci && npm run build && npm test -- --ci` on a fresh runner — refusing to proceed if `backend/generated/` is present — locks in the fix.

**Scope**
- Track: B
- Files: `.github/workflows/backend-smoke.yml` (new)
- Change: new CI workflow

**Acceptance Criteria**
- [ ] `backend-smoke.yml` triggers on PRs touching `backend/**` or the workflow itself.
- [ ] The workflow refuses to proceed if `backend/generated/` exists on the runner (guards against smuggled artefacts).
- [ ] `npm test -- --ci` is used so any suite load error fails CI.
- [ ] The workflow runs green on `dev` after Track A merges.

**Testing**
- Open a throwaway PR that deletes `.env.test` — expect the smoke workflow to fail.
- Open a throwaway PR that imports `helpers.ts` from a new file with no env — expect the workflow to still pass (Track B has removed the import-time throw).

---

## US-91-7 — Audit reusable `node-ci.yml` for silent-skip behaviour

**Body**
The reporter asked whether the reusable `node-ci.yml@dev` actually builds and tests `backend/`. Even after Track A + Track B land, a `--passWithNoTests` or missing `npm test` invocation in the reusable workflow could still mask suite-load failures. This story is an audit-and-fix pass on the shared workflow.

**Scope**
- Track: B
- Files: reusable `node-ci.yml` (owned outside this repo — coordinate with CI owners)
- Change: remove any `--passWithNoTests`; confirm `npm test` runs for `backend/`

**Acceptance Criteria**
- [ ] Confirmed that `node-ci.yml@dev` runs `npm test` for `backend/`.
- [ ] Confirmed the invocation does not include `--passWithNoTests`.
- [ ] If either is missing, a PR is opened against the workflow's home repo with the fix.

**Testing**
- Read the current `node-ci.yml` invocation for connection-studio.
- Simulate: temporarily rename a suite so it fails to load — confirm CI now fails.
