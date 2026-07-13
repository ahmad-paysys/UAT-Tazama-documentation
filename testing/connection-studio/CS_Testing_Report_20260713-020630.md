# Testing Report

## Connection Studio — Backend Test-Environment Reproducibility & Encryption-Key Hardening

**Prepared for:** Product Owner
**Date:** 13 July 2026
**Development scope:** Connection Studio issue **[#91](https://github.com/tazama-lf/connection-studio/issues/91)**
**Implementation:** [Connection Studio PR #98](https://github.com/tazama-lf/connection-studio/pull/98) (MERGED)

---

## 1. Executive Summary

This report defines the acceptance testing scope for a **quality-gate hardening** delivery in the Connection Studio backend. The work does not add or change any user-visible feature. It closes a class of failure that undermined the trustworthiness of the automated tests protecting Connection Studio and, indirectly, made every future change on this component riskier.

The Product Owner should understand this delivery as **infrastructure hygiene**, not as a new capability. The testing described in this report is therefore not a browser walk-through — it is a set of clearly defined checks that either the QA team or the development team can execute on a laptop or against CI, with unambiguous pass/fail outcomes.

| Track | Business Problem | What Changed |
|-------|------------------|--------------|
| **Track A — Reproducible test environment** | On a clean clone of the repository, running `npm test` in the backend reported "6 of 12 test suites failed" — but not as real assertion failures. Six suites **failed to even load** because they required a 32-byte `ENCRYPTION_KEY` environment variable that lived only in each developer's un-committed local `.env`. The remaining six suites passed, so a partially-green run masked the fact that half of the safety net was silently switched off. | A committed, non-secret test environment file (`backend/.env.test`) is now the single source of test env. Jest loads it explicitly. From a clean checkout, all 12 (now 13) test suites load and pass. A vestigial Prisma dependency that had misled a prior investigator was also removed. |
| **Track B — Hardening so this class of failure cannot recur** | The reason the six suites could not even load was that the encryption-key check ran at **module-import time** — the very act of loading the file threw an error when the environment was misconfigured. In production, the same design meant a wrong-length key was only surfaced deep inside the first cryptographic call, not at application startup. | The encryption-key check is now deferred to the first `encrypt` / `decrypt` call, so importing the helper is always safe. Independently, a length-and-presence validator on `ENCRYPTION_KEY` now runs at Nest boot — a misconfigured production deployment fails to start with a clear, early error, rather than crashing at first request. |

**Overall status**

| Track | Delivery status | Outstanding |
|-------|-----------------|-------------|
| Track A | Merged in PR #98 | None. |
| Track B | Merged in PR #98 (same PR, same feature branch) | None. |
| `tcs-lib 1.0.157-rc.0` upgrade | Explicitly out of scope for this delivery | Tracked under [tazama-lf/tcs-lib#51](https://github.com/tazama-lf/tcs-lib/issues/51). Handled by holding `tcs-lib` at `1.0.156-rc.x`, consistent with the rule-studio precedent (see rule-studio#127). |

---

## 2. Business Context (What the Product Owner Needs to Know)

### 2.1 What went wrong

The Connection Studio backend has a suite of automated tests that run on every code change. Those tests exist for one reason: to give the team, and by extension the Product Owner, confidence that a change has not broken something.

Before this delivery, that confidence was **partly illusory**:

- Half the test suites contained real, valuable checks — but from a fresh check-out of the code, those suites could not even start running.
- The reason: a hard check at the top of a shared utility file that required an environment variable (`ENCRYPTION_KEY`) to be exactly 32 characters long. If the variable was missing or the wrong length, the file threw an error the instant it was loaded.
- Every developer worked around this by keeping the correct value in their **own private `.env` file**, which is intentionally not committed to source control. New joiners, CI runners, and fresh clones did not have that file — so on their machines, six of twelve test suites silently failed to load.
- The remaining six suites ran and passed. A run of `npm test` therefore looked mostly-green, and it was easy to miss that half of the safety net was switched off.

A separate but related design issue: in production, the same environment-check ran only when the code actually needed to encrypt or decrypt something. A misconfigured server with the wrong-length key would boot up cleanly and only fail on the first real request — potentially long after deployment.

### 2.2 What is now true

- The correct test environment is **committed to the repository** as `backend/.env.test`. This is a non-secret file — the key inside it is used only by the automated tests, never by any real system. From now on, a clean clone of the repository plus `npm ci && npm test` reproduces the full test run without any hidden local state.
- The utility file no longer throws at import. It waits until an encryption call is actually made, and if the environment is misconfigured at that point, it produces a clear error.
- In production, the environment validator at Nest start-up enforces the 32-byte length. A wrong-length key now stops the service from booting — with an explicit error — rather than lurking until the first customer request.
- The vestigial `@prisma/client` dependency that led the original reporter to briefly misdiagnose the issue has been removed, along with the corresponding no-op step in the root lint script. A future dependency sweep will not fall into the same trap.
- A short section has been added to `backend/README.md` documenting the test-env contract so future contributors do not have to rediscover it.

### 2.3 Why the Product Owner should care

Two reasons:

1. **The safety net is now real.** From this delivery onwards, when a Connection Studio pull request reports "tests pass", the Product Owner can trust that statement in a way that was not fully true before. Half the suites are no longer silently skipped.
2. **Production failure mode is earlier and clearer.** A misconfigured production deployment — for example, if an operations team pastes a truncated encryption key into an environment file — will now be caught at boot with a specific, named error, instead of surfacing as a mysterious crash on the first cryptographic call.

There is **no change to any end-user workflow** in the Connection Studio UI. The Product Owner will not see anything different in the browser.

---

## 3. Acceptance Criteria

Because this delivery does not change user-visible behaviour, the acceptance criteria are stated in terms of **observable, reproducible outcomes** on a clean checkout of the repository or in CI. Each criterion is designed to be checkable by a QA engineer or a developer without needing to read any code.

### 3.1 Track A — Reproducible test environment

**A1. Committed test-env file exists**
- The file `backend/.env.test` exists in the tracked repository.
- Its contents include a **32-character** `ENCRYPTION_KEY`, `IV_LENGTH=16`, and `NODE_ENV=test`.
- The `.gitignore` files in the repository explicitly allow `.env.test` while continuing to ignore other `.env*` variants (so real secrets remain untracked).

**A2. Clean-checkout test run is green in full**
- After a fresh `git clone` (no prior local files) and `cd backend && npm ci && npm test`, the report shows **all 13 test suites loaded**.
- No suite fails to load. No suite emits "ENCRYPTION_KEY must be 32 bytes for aes-256-cbc" during load.
- All 564 individual tests pass.

**A3. Jest loads the committed file explicitly (no fallback to any un-tracked `.env`)**
- If `backend/.env.test` is deleted or renamed locally and the test run is re-attempted, the failure surfaces as a **clear, named** error (either a Jest bootstrap error or a Nest-boot validator error citing `ENCRYPTION_KEY`) — not as a silent partial green.

**A4. Vestigial Prisma dependency removed**
- `backend/package.json` contains **no** entry for `@prisma/client` and **no** entry for `prisma`.
- The root `package.json` script `backend:lint:eslint` does **not** contain `npx prisma generate`.

**A5. README documents the test-env contract**
- `backend/README.md` includes a short section explaining that Jest loads `backend/.env.test` via `test/jest.env.ts`, that this file is non-secret, and that any new module reading env at import time must add a placeholder value here.

### 3.2 Track B — Hardening

**B1. Utility imports are safe when the environment is unset**
- Importing `backend/src/utils/helpers.ts` in an environment with no `ENCRYPTION_KEY` set **does not throw** at import time.
- The dedicated test `backend/test/utils/helpers.spec.ts` covers this contract and passes.

**B2. Encrypting with an unset key produces a clear, named error**
- Calling `encrypt(...)` with no `ENCRYPTION_KEY` throws an error whose message includes the phrase **"ENCRYPTION_KEY is not set"**.
- Calling `decrypt(...)` on the output of `encrypt(...)` with a valid 32-byte key returns the original plaintext (round-trip check).

**B3. Nest boot fails with a clear message when the key is the wrong length**
- Starting the backend with `ENCRYPTION_KEY` set to any value **other than 32 bytes** results in the service failing to boot, with an error message identifying `ENCRYPTION_KEY` as the offending variable and stating the 32-byte requirement.
- Starting the backend with `ENCRYPTION_KEY` unset likewise produces a clear, early failure — not a delayed crash on first cryptographic call.

**B4. Tests remain robust to their own environment**
- The new `helpers.spec.ts` suite passes even when `backend/.env.test` is temporarily removed. The tests manage their own environment state internally and do not rely on the committed file.

### 3.3 Cross-cutting

**X1. No user-facing regression in Connection Studio**
- After deploying this change, the Connection Studio UI continues to behave exactly as before: logging in, viewing configurations, running dry-runs, and any other pre-existing workflow succeed with no visible difference.

**X2. Explicit non-goals of this delivery**
- No new UI screens.
- No new backend endpoints.
- No database schema migration.
- No change to `@tazama-lf/tcs-lib` version. The `1.0.157-rc.0` `related_transaction` question is explicitly out of scope and is tracked in [tazama-lf/tcs-lib#51](https://github.com/tazama-lf/tcs-lib/issues/51).

---

## 4. Test Environment & Preparation

Because this is an infrastructure delivery, the "test environment" is a developer laptop or a CI runner rather than a browser session. The tester needs:

1. A machine with **Node.js 20** installed (matching the version used in CI).
2. Git access to the `tazama-lf/connection-studio` repository.
3. A shell (bash, zsh, PowerShell) with `git`, `npm`, and standard command-line tools.
4. **No** pre-existing local `backend/.env` file — the whole point of the test is to prove the flow works from a genuinely clean state. If a `.env` exists from prior work, either rename it or use a clean directory.
5. For **Scenario 5 (production boot)**, the ability to start the backend locally: `cd backend && npm run start`. A running admin-service is not required for this specific check — the Nest env validator runs before any external calls.
6. For **Scenario 8 (CI observation)**, GitHub access to the `tazama-lf/connection-studio` repository actions/checks tab.

No specific test data is required — the affected code paths operate purely on environment variables and in-process cryptographic primitives.

---

## 5. Test Scenarios

Each scenario is a self-contained walkthrough. Steps are given as commands the tester runs and results the tester should observe.

### Scenario 1 — Clean-checkout test run is fully green

**Acceptance criteria covered:** A2, A1

1. From an empty directory, clone the repository:
   `git clone https://github.com/tazama-lf/connection-studio.git`
2. Enter the backend package:
   `cd connection-studio/backend`
3. Confirm no local `.env` file exists in this directory (only `.env.test` should be present).
4. Install dependencies:
   `npm ci`
5. Run the test suite:
   `npm test`

**Expected result**
- The Jest summary at the end reports **13 test suites**, all passing. Suites listed include `sftp`, `notification`, `job`, `dry-run`, `config`, `config-workflow`, plus the new `helpers` suite.
- **No** suite reports a failure to load. **No** occurrence of the message "ENCRYPTION_KEY must be 32 bytes for aes-256-cbc" appears in the output.
- Total individual tests reported: **564 passed**.

---

### Scenario 2 — Removing the committed test-env is a loud failure, not silent partial green

**Acceptance criteria covered:** A3

1. Continuing from Scenario 1, temporarily rename the committed file:
   `mv .env.test .env.test.bak`
2. Re-run:
   `npm test`

**Expected result**
- The run fails clearly. Either a Jest bootstrap error names the missing `.env.test` path, or the Nest-boot validator reports `ENCRYPTION_KEY` is missing / not 32 bytes.
- The failure is **not** a partial green with six silent skips. The nature of the problem is obvious from the console output.

3. Restore the file:
   `mv .env.test.bak .env.test`
4. Re-run `npm test` and confirm the green result of Scenario 1 is reproduced.

---

### Scenario 3 — Prisma vestigial dependency is gone

**Acceptance criteria covered:** A4

1. Inspect the backend manifest:
   `grep -n "prisma" backend/package.json`
2. Inspect the root manifest:
   `grep -n "prisma" package.json`

**Expected result**
- The `grep` on `backend/package.json` returns **no results** — no `@prisma/client`, no `prisma`.
- The `grep` on the root `package.json` returns **no results** — the `backend:lint:eslint` script no longer contains `npx prisma generate`.

---

### Scenario 4 — README documents the test-env contract

**Acceptance criteria covered:** A5

1. Open `backend/README.md` in any text viewer.

**Expected result**
- A section titled "Running Tests" (or similar) is present, explaining that Jest loads `backend/.env.test` via `test/jest.env.ts`, that this file is committed and non-secret, and that new modules reading env at import time must add a placeholder to `.env.test`.

---

### Scenario 5 — Nest boot fails early on a wrong-length production key

**Acceptance criteria covered:** B3

1. Ensure a genuine environment file exists for a local dev boot (`backend/.env`), populated with the values Connection Studio normally needs to start, but with `ENCRYPTION_KEY` deliberately set to a **short** value (for example, `ENCRYPTION_KEY=short`).
2. Attempt to boot the service:
   `cd backend && npm run start`

**Expected result**
- The service **fails to boot** almost immediately.
- The console output contains a validator error identifying `ENCRYPTION_KEY` as the failing variable and stating the 32-byte length requirement.
- No HTTP endpoint becomes available. The failure is visibly a start-up validation issue, not a runtime cryptography crash.

3. Correct `ENCRYPTION_KEY` to a 32-byte value and rerun. Confirm the service now boots normally (subject to whatever other environment or dependencies the local run needs).

---

### Scenario 6 — Importing the helper is safe when the environment is unset

**Acceptance criteria covered:** B1, B2

This is the check that the underlying root cause of Track A cannot recur.

1. In the backend package, run the dedicated helper suite:
   `npx jest test/utils/helpers.spec.ts`

**Expected result**
- Three tests run, all pass:
  - Importing `utils/helpers` with no `ENCRYPTION_KEY` set does **not** throw.
  - Calling `encrypt(...)` with no `ENCRYPTION_KEY` throws an error whose message contains `ENCRYPTION_KEY is not set`.
  - With a valid 32-byte key set, `decrypt(encrypt("hello"))` returns `"hello"`.

---

### Scenario 7 — The helper suite is self-sufficient (no hidden dependency on the committed file)

**Acceptance criteria covered:** B4

1. Temporarily rename the committed file:
   `mv backend/.env.test backend/.env.test.bak`
2. Run only the helper suite:
   `npx jest test/utils/helpers.spec.ts`

**Expected result**
- All three tests still pass. The suite manages its own environment state internally.

3. Restore the file:
   `mv backend/.env.test.bak backend/.env.test`

---

### Scenario 8 — CI observation (spot check)

**Acceptance criteria covered:** A2, A3 (indirectly)

1. Open the `tazama-lf/connection-studio` repository on GitHub.
2. Navigate to the **Actions** tab and open the most recent successful pipeline run against the branch this delivery merged into.

**Expected result**
- The backend build and test steps completed successfully.
- The test job output reports 13 suites loaded (matching Scenario 1 locally), with no "6 failed of 12" pattern that characterised runs before this delivery.

---

### Scenario 9 — No user-facing regression

**Acceptance criteria covered:** X1

1. Deploy the change to the UAT environment (or the environment nominated by the Product Owner for acceptance).
2. Sign in to Connection Studio as any authorised user.
3. Exercise the standard pre-existing workflows the Product Owner uses on this system — for example: listing configurations, opening a configuration, running a dry-run, publishing a change (as authorised by role).

**Expected result**
- Every workflow behaves exactly as before this delivery. No UI element has moved, no new error is displayed, no additional confirmation is required.

---

## 6. Regression Areas to Watch

While executing the scenarios above, the tester should also confirm the following pre-existing behaviours remain healthy:

- **All Connection Studio UI flows** continue to function as before this delivery. This delivery is invisible to end users.
- **Dependency install** (`npm ci`) completes cleanly and no console warning implies Prisma is still expected.
- **CI**: subsequent pull requests to `dev` continue to run the full test suite and continue to report all 13 suites — the improvement is durable, not one-off.
- **Encryption / decryption in production**: any pre-existing cryptographic feature of Connection Studio (for example, storage of a credential) continues to encrypt and decrypt correctly. The refactor is behaviour-preserving for the happy path; it only changes what happens on misconfiguration.

---

## 7. Definition of Done for This Delivery

The Product Owner may accept this delivery when:

- [ ] Scenarios 1–9 have been executed and passed.
- [ ] Every acceptance criterion in Section 3 has been observed as true at least once.
- [ ] Regression areas in Section 6 have been spot-checked without discovering new issues.
- [ ] Any defects raised during the checks are triaged and either fixed or explicitly deferred with Product Owner agreement.

---

## 8. Out of Scope for This Delivery

The following items are **not** part of this delivery and should not block acceptance:

- **`@tazama-lf/tcs-lib 1.0.157-rc.0` upgrade** — the removal of `related_transaction` in that release conflicts with current Connection Studio code (`config.service.ts` still consumes it). This delivery deliberately holds `tcs-lib` at `1.0.156-rc.x`, matching the precedent set in rule-studio. The library-side reconciliation is tracked separately in [tazama-lf/tcs-lib#51](https://github.com/tazama-lf/tcs-lib/issues/51). No action is required in Connection Studio until that upstream item is resolved.
- **Broader dependency-security sweep** for Connection Studio — this delivery unblocks the sweep by proving the test suite is reproducible, but the sweep itself is a separate exercise.
- **Any change to production encryption or key-management strategy** — the encryption-key contract itself (32 bytes for `aes-256-cbc`) is unchanged. Only *when* the check runs has been improved.
- **Any change to the Connection Studio frontend.**

---

## 9. Sign-off

| Role | Name | Signature | Date |
|------|------|-----------|------|
| Product Owner | | | |
| QA Lead | | | |
| Engineering Lead | | | |

---

*Document prepared based on Connection Studio issue #91 and pull request #98. For the full technical rationale, refer to the impact and solution documents held in the UAT board repository under `issues/connection-studio/91/`.*
