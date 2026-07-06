# Issue #91 — Backend build & tests not reproducible from clean checkout - quality-gate gap (Prisma schema absent, test ENV required, tcs-lib RC breaking)

**Repository:** tazama-lf/connection-studio
**Issue:** [Backend build & tests not reproducible from clean checkout](https://github.com/tazama-lf/connection-studio/issues/91)
**Author:** Justus-at-Tazama (Justus Ortlepp)
**State:** Open
**Report Date:** 2026-07-03

---

## Executive Summary

The reporter opened this issue after a routine dependency-security refresh on `backend/` surfaced three problems that they interpreted as a CI quality-gate gap. In a follow-up comment the author retracted Finding #1 (Prisma) after re-running the reproduction and confirming that `nest build` exits 0 on a clean checkout — there are zero Prisma imports anywhere in `backend/src`. Finding #3 (tcs-lib `1.0.157-rc.0` removing `related_transaction`) is real but is being handled repository-wide by pinning tcs-lib to `1.0.156-rc.x`, mirroring the rule-studio precedent, and is tracked in `tazama-lf/tcs-lib#51`.

The substantive item that remains for this issue is Finding #2: 6 of 12 jest suites in `backend/` fail to *load* (not fail assertions) on a clean checkout because [src/utils/helpers.ts:20-22](../../../repos/connection-studio/backend/src/utils/helpers.ts#L20-L22) throws at module-import time when `ENCRYPTION_KEY` is not a 32-byte string. Jest's `setupFiles: ['dotenv/config']` loads `.env`, which is not committed; `.env.example` contains the key but is not read by jest. A partial-green jest run can therefore mask that half the suites never executed.

A secondary cleanup called out by the author is the vestigial `@prisma/client` dependency in [backend/package.json:41](../../../repos/connection-studio/backend/package.json#L41) and the no-op `npx prisma generate` step in the root [package.json:10](../../../repos/connection-studio/package.json#L10) `backend:lint:eslint` script — neither has a tracked `schema.prisma` to generate from and no source file imports the client.

---

## How It Works Today — Confirmed in Code

### Encryption key is required at module-import time

[src/utils/helpers.ts:16-22](../../../repos/connection-studio/backend/src/utils/helpers.ts#L16-L22):

```ts
const { ENCRYPTION_KEY, IV_LENGTH } = process.env;

const key = Buffer.from(ENCRYPTION_KEY!, 'utf8');

if (key.length !== 32) {
  throw new Error('ENCRYPTION_KEY must be 32 bytes for aes-256-cbc');
}
```

This runs at import — the moment any file transitively imports `utils/helpers.ts`, node evaluates the top level and throws if `ENCRYPTION_KEY` is missing or wrong-length. Because the check is at module top level, a test suite that only mocks the collaborators still fails to *load*.

### Jest setup loads `.env`, not `.env.example`

[backend/jest.config.ts:25-26](../../../repos/connection-studio/backend/jest.config.ts#L25-L26):

```ts
// Load .env before all tests
setupFiles: ['dotenv/config'],
```

`dotenv/config` reads `.env` from the current working directory. There is no `.env` file committed (it is git-ignored). `.env.example` is present and contains `ENCRYPTION_KEY=8fj29dkd82hs91kd93kd82hs91kd83ks` and `IV_LENGTH=16`, but jest does not read it.

### Files that transitively import `utils/helpers.ts`

Confirmed with `grep -rln "utils/helpers"` in `src/` and `test/`:

- `src/dry-run/dry-run.service.ts:16`
- `src/scheduler/scheduler.service.ts`
- `src/job/job.service.ts`
- `src/notification/notification.service.ts`
- `src/sftp/sftp.service.ts`
- `test/dry-run/dry-run.service.spec.ts`
- `test/job/job.service.spec.ts`
- `test/notification/notification.service.spec.ts`
- `test/sftp/sftp.service.spec.ts`

The `config.service.spec.ts` and `config-workflow.service.spec.ts` suites reach `helpers.ts` transitively through their imports of `SftpService` / `NotificationService`. That matches the reporter's "6 failed of 12" — the six suites that touch this dependency chain fail at load.

### Env schema declares ENCRYPTION_KEY but does not seed it

[src/config/env.validation.ts:38-39](../../../repos/connection-studio/backend/src/config/env.validation.ts#L38-L39):

```ts
@IsString()
ENCRYPTION_KEY: string;
```

The schema uses `skipMissingProperties: true` ([env.validation.ts:55](../../../repos/connection-studio/backend/src/config/env.validation.ts#L55)), so a missing value is tolerated by validation — but the hard length check in [helpers.ts:20-22](../../../repos/connection-studio/backend/src/utils/helpers.ts#L20-L22) still throws.

### Vestigial Prisma dependency

[backend/package.json:41](../../../repos/connection-studio/backend/package.json#L41):

```json
"@prisma/client": "^6.12.0",
```

Plus `"prisma": "^6.13.0"` in devDependencies. A repo-wide search confirms no source file imports either symbol.

Root [package.json:10](../../../repos/connection-studio/package.json#L10):

```json
"backend:lint:eslint": "cd backend && npm ci && npx prisma generate && npm run lint",
```

`npx prisma generate` has no tracked `schema.prisma` to consume; the step is a no-op. `backend:build` does not invoke it at all.

---

## Root Cause Analysis

The root cause is that `src/utils/helpers.ts` performs a hard, side-effectful validation of `ENCRYPTION_KEY` at module-import time, and the jest configuration loads env only from an un-tracked `.env` file. Every consequence downstream flows from that: six suites that transitively import `helpers.ts` fail to *load* on a fresh checkout; partial green results can mask the failures because jest reports "226 tests passed" while silently omitting six suites; and there is no committed test-env source that would let CI or a new contributor run the full suite deterministically.

The Prisma dependency is unrelated to the failure — it is dead weight that the reporter initially misdiagnosed and has since retracted, but is worth removing to prevent the same misdiagnosis in future.

---

## Blast Radius — Full File Inventory

### Files that must change

| File | What is affected | Lines |
|------|------------------|-------|
| `backend/.env.test` (new) | New test-env source with a 32-byte `ENCRYPTION_KEY` and `IV_LENGTH=16` | new file |
| `backend/jest.config.ts` | Point `setupFiles` at `.env.test` instead of relying on `.env` | 25-26 |
| `backend/src/utils/helpers.ts` | Optional: defer the encryption-key validation until first use, so importing the module does not throw | 16-22 |
| `backend/package.json` | Remove `@prisma/client` and `prisma` devDependency | 41, ~101 |
| `package.json` (root) | Remove the no-op `npx prisma generate` from `backend:lint:eslint` | 10 |
| `backend/README.md` | Document the test-env requirement (32-byte `ENCRYPTION_KEY`) | new section |
| `.github/workflows/*` (CI) | Ensure CI does not `--passWithNoTests` and fails on any suite load error | as applicable |

### Files indirectly affected (verified, unchanged in Track A)

| File | Why it is in scope | Lines |
|------|--------------------|-------|
| `backend/src/dry-run/dry-run.service.ts` | Imports `utils/helpers` (`isValidText`) | 16 |
| `backend/src/scheduler/scheduler.service.ts` | Imports `utils/helpers` | — |
| `backend/src/job/job.service.ts` | Imports `utils/helpers` | — |
| `backend/src/notification/notification.service.ts` | Imports `utils/helpers` | — |
| `backend/src/sftp/sftp.service.ts` | Imports `utils/helpers` | — |
| `backend/test/config/config.service.spec.ts` | Transitively pulls `helpers.ts` via SftpService/NotificationService | — |
| `backend/test/config/config-workflow.service.spec.ts` | Same transitive path | — |
| `backend/src/config/env.validation.ts` | Declares `ENCRYPTION_KEY` — kept as-is | 38-39 |

---

## Side-Effect Map

| Consequence | Trigger | Where it surfaces |
|-------------|---------|-------------------|
| Six jest suites fail to load on clean checkout | Missing `.env` on fresh clone | `npm test` in `backend/` |
| Partial green run masks failures | Jest reports passing tests while six suites never ran | CI logs, developer terminals |
| New contributors cannot reproduce the test suite | No `.env.test` committed and no docs | On-boarding, PR reviews |
| Same misdiagnosis of Prisma will recur | Vestigial `@prisma/client` and no-op `prisma generate` in root script | Future dependency sweeps |
| Silent tcs-lib RC upgrade will break the build | `1.0.157-rc.0` removes `related_transaction` used at `config.service.ts:178` | Any bump past `1.0.156` |

---

## Effort Assessment

### Track A — Minimal, Immediate Fix

Add a committed `.env.test`, point jest at it, and remove the vestigial Prisma pieces. No source-code behaviour changes.

| Step | File | Effort |
|------|------|--------|
| A1 | Add `backend/.env.test` with a 32-byte `ENCRYPTION_KEY` and `IV_LENGTH=16` | 5 min |
| A2 | Update `backend/jest.config.ts` `setupFiles` to load `.env.test` | 5 min |
| A3 | Remove `@prisma/client` and `prisma` from `backend/package.json` | 5 min |
| A4 | Remove `npx prisma generate` from the root `backend:lint:eslint` script | 2 min |
| A5 | Document env requirement in `backend/README.md` | 10 min |
| A6 | Run `npm ci && npm test` from a clean state and confirm 12/12 suites load | 10 min |

- Schema migration required: No
- Frontend changes required: No

### Track B — Full Correct Fix (Reproducibility Hardening)

Track A plus source-level and CI hardening so the failure mode cannot recur.

| Step | File | Effort |
|------|------|--------|
| B1 | Everything in Track A | (as above) |
| B2 | Defer the 32-byte key check in `src/utils/helpers.ts` from module top-level into `encrypt`/`decrypt` (so `import` never throws) | 20 min |
| B3 | Add a jest `setupFilesAfterEach` (or `setupFiles`) that fails fast if `ENCRYPTION_KEY` is unset when a test needs it | 15 min |
| B4 | Update `env.validation.ts` to enforce `ENCRYPTION_KEY.length === 32` at boot so misconfigured prod is caught by the config validator, not by a runtime import | 20 min |
| B5 | Confirm the reusable `node-ci.yml@dev` job runs `npm test` for `backend/` and fails on any suite load error (add `--ci` and remove any `--passWithNoTests` if present) | 30 min |
| B6 | Add a smoke workflow: fresh checkout → `npm ci` → `npm run build` → `npm test`, no local artefacts | 30 min |

- Schema migration required: No
- Frontend changes required: No

### Recommended Sequencing

1. Ship Track A first — it is a config/documentation fix, safe in isolation, and closes the "half the suites silently skipped" gap.
2. Ship Track B in a follow-up PR after Track A merges, so the hardening lands on a green baseline.
3. tcs-lib `1.0.157-rc.x` adoption is deferred until `tazama-lf/tcs-lib#51` lands; not part of this issue.

---

## Acceptance Criteria

### Track A

- [ ] `backend/.env.test` is committed with a 32-byte `ENCRYPTION_KEY` and `IV_LENGTH=16`.
- [ ] `backend/jest.config.ts` `setupFiles` loads `backend/.env.test` explicitly.
- [ ] On a clean checkout, `cd backend && npm ci && npm test` reports 12 of 12 suites loaded (no `TypeError ... ENCRYPTION_KEY must be 32 bytes` errors).
- [ ] `@prisma/client` and `prisma` are removed from `backend/package.json`.
- [ ] Root `package.json` `backend:lint:eslint` no longer runs `npx prisma generate`.
- [ ] `backend/README.md` documents the test-env expectation.

### Track B

- [ ] `src/utils/helpers.ts` no longer throws at module-import time; the 32-byte check runs on first `encrypt`/`decrypt` call.
- [ ] `env.validation.ts` fails boot when `ENCRYPTION_KEY` is not 32 bytes.
- [ ] CI runs a clean-checkout smoke workflow that builds and tests `backend/` without any local artefacts.
- [ ] CI fails if any jest suite fails to load, not only on assertion failures.

---
