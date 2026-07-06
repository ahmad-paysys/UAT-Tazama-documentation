# Impact Study — Issue #91: Backend build & tests not reproducible from clean checkout

**Issue:** [#91](https://github.com/tazama-lf/connection-studio/issues/91)
**Study Date:** 2026-07-03
**Author:** Ahmad Khalid
**Repository:** tazama-lf/connection-studio

---

## Summary

Issue #91 originally raised three findings against `backend/`: a missing Prisma schema, six jest suites that fail to load without a 32-byte `ENCRYPTION_KEY`, and a breaking `@tazama-lf/tcs-lib` `1.0.157-rc.0`. In a follow-up comment the reporter retracted Finding #1 (backend does not import Prisma anywhere), and Finding #3 is being handled through a tcs-lib pin (rule-studio precedent, tracked in `tazama-lf/tcs-lib#51`). The substantive work for this issue is Finding #2 — the test-env gap — plus removing the vestigial `@prisma/client` dependency. Track A is a config/docs fix; Track B moves the encryption-key validation off the module import path and hardens CI.

---

## Confirmed Root Cause

- [src/utils/helpers.ts:16-22](../../../repos/connection-studio/backend/src/utils/helpers.ts#L16-L22) evaluates `ENCRYPTION_KEY` and throws if `Buffer.from(ENCRYPTION_KEY, 'utf8').length !== 32` at module-import time — verified by reading the file.
- [backend/jest.config.ts:25-26](../../../repos/connection-studio/backend/jest.config.ts#L25-L26) loads env via `setupFiles: ['dotenv/config']`, which reads `.env` only. `.env` is git-ignored; `.env.example` contains a 32-byte key but jest does not read it. Verified by reading `jest.config.ts` and `ls -a`.
- Six suites transitively import `utils/helpers.ts` via `SftpService` / `NotificationService` / `JobService` / `DryRunService` / `SchedulerService`. Verified by `grep -rln "utils/helpers"`.
- Finding #1 (Prisma) is retracted: `grep -rn "prisma" src` finds zero matches inside `src/`. `@prisma/client` is declared at [backend/package.json:41](../../../repos/connection-studio/backend/package.json#L41) but never imported.
- Finding #3 (tcs-lib RC) is real but out of scope for this issue's fix — pinning strategy is documented in the issue comments.

---

## Track A — Test-Env Fix + Prisma Cleanup

### What Changes

- Add `backend/.env.test` (new file, ~5 lines) with a 32-byte `ENCRYPTION_KEY` and `IV_LENGTH=16`.
- Update `backend/jest.config.ts` (~2 lines) to load `.env.test` explicitly.
- Remove `@prisma/client` and `prisma` entries from `backend/package.json`.
- Remove `npx prisma generate` from the root `package.json` `backend:lint:eslint` script.
- Add a short section to `backend/README.md` documenting the test-env expectation.

### Impact

| Category | Value |
|----------|-------|
| Files changed | 5 |
| Schema migration required | No |
| Frontend changes required | No |
| Downtime required | No |
| Risk of regression | Low |
| Reversibility | Trivial (single-commit revert) |

**What Track A fixes immediately**

- All 12 backend jest suites load from a clean checkout.
- Partial-green runs that hid six missing suites can no longer happen.
- The `@prisma/client` red herring is removed, so the next dependency sweep will not repeat the misdiagnosis.

**What Track A does not fix**

- `src/utils/helpers.ts` still throws at import time if `ENCRYPTION_KEY` is unset — merely committing `.env.test` masks the symptom, it does not remove the top-level side effect.
- CI configuration (`node-ci.yml`) is not audited; a future workflow change could still mask a suite-load failure.
- `env.validation.ts` still permits any string for `ENCRYPTION_KEY`; a wrong-length value would only be caught at runtime.

Track A is safe to ship in isolation.

---

## Track B — Reproducibility Hardening

### What Changes

Track A plus source and CI hardening so the failure mode cannot recur: move the 32-byte check off module-import, enforce length in the config validator, and add a clean-checkout smoke workflow.

### Schema Impact

| Change | Migration | Risk |
|--------|-----------|------|
| None | Not required | None |

### Backend Code Impact

**Core rewrites**

| File | Change |
|------|--------|
| `backend/src/utils/helpers.ts` | Move the 32-byte length check into `encrypt`/`decrypt` (or a lazy `getKey()` helper); remove top-level throw. |
| `backend/src/config/env.validation.ts` | Add a custom validator that enforces `ENCRYPTION_KEY.length === 32` at boot. |

**New files**

| File | Purpose |
|------|---------|
| `backend/.env.test` | Committed test-env source (Track A carry-over). |
| `.github/workflows/backend-smoke.yml` (or equivalent) | Clean-checkout smoke build + full test. |

**Enum/reference sweep files** — none.

### Frontend Code Impact

| File | Change |
|------|--------|
| — | No frontend changes. |

Test fixture scope: no additional fixtures required; existing suites already exercise the encryption paths.

---

## Side Effects and Risks

### Risks of Track A Alone

| Risk | Likelihood | Mitigation |
|------|------------|------------|
| A future refactor removes `.env.test` without noticing, re-introducing the failure | Low | Add a CI check that fails if `.env.test` is missing. |
| Developers assume the test key is production-safe | Low | README note calling out that `.env.test` is a test artefact and must not be reused in prod. |
| The module top-level throw still bites anyone running scripts (not jest) without env | Low | Documented; Track B addresses it. |

### Risks of Track B

| Risk | Likelihood | Mitigation |
|------|------------|------------|
| Deferred key check delays discovery of misconfigured prod | Low | `env.validation.ts` enforces length at boot, so misconfigured prod fails at start-up, not at first `encrypt` call. |
| Refactor introduces a regression in encrypt/decrypt | Low | Existing suites cover the codepath; add a targeted unit test for the deferred-check path. |
| CI workflow changes conflict with concurrent CI work | Low | Coordinate with whoever owns `node-ci.yml`. |

### Cross-Issue Dependencies

- Issue #91 does not share files with any other currently open issue directory under `issues/connection-studio/`.
- Finding #3 (tcs-lib RC) is coordinated separately in `tazama-lf/tcs-lib#51` and via the tcs-lib pin at [backend/package.json:46](../../../repos/connection-studio/backend/package.json#L46). No source change here.

---

## Effort Estimate

| Track | Files | Effort |
|-------|-------|--------|
| Track A | 5 | ~0.5 day |
| Track B | 4 (plus Track A carry-over) | ~1 day |
| **Total** | **9** | **~1.5 days** |

---

## Acceptance Criteria (Verification Checklist)

### Track A

- [ ] `backend/.env.test` is committed with a 32-byte `ENCRYPTION_KEY` and `IV_LENGTH=16`.
- [ ] `backend/jest.config.ts` `setupFiles` loads `backend/.env.test` explicitly.
- [ ] `cd backend && npm ci && npm test` on a clean checkout reports 12 of 12 suites loaded.
- [ ] `@prisma/client` and `prisma` are removed from `backend/package.json`.
- [ ] Root `package.json` `backend:lint:eslint` no longer runs `npx prisma generate`.
- [ ] `backend/README.md` documents the test-env expectation.

### Track B

- [ ] `src/utils/helpers.ts` no longer throws at module-import time.
- [ ] `env.validation.ts` fails boot when `ENCRYPTION_KEY` is not 32 bytes.
- [ ] A clean-checkout CI smoke workflow builds and tests `backend/` with no local artefacts.
- [ ] CI fails on any jest suite load error, not only on assertion failures.

---

## Recommended Sequencing

1. Merge Track A first (single PR). It closes the immediate reproducibility gap and is a low-risk config/docs change.
2. Merge Track B in a follow-up PR after Track A is green on `dev`.
3. Do not adopt `@tazama-lf/tcs-lib` `1.0.157-rc.x` until `tazama-lf/tcs-lib#51` lands and `config.service.ts:178` is reconciled with the new type surface — tracked separately.
