# Complete Fix — Exact Code Changes

Track A can merge independently as a single PR. Track B builds on Track A and should merge as a second PR after Track A lands green on `dev`. Neither track requires a schema migration.

---

## Track A — Test-Env Fix + Prisma Cleanup (PR 1 of 2)

### File: `backend/.env.test` (new file)

**BEFORE**

_(file does not exist)_

**AFTER**

```env
# Test-only env — committed intentionally so `npm test` is reproducible from a clean checkout.
# The key below is NOT a secret. It is a 32-byte string used only by unit tests.
ENCRYPTION_KEY=8fj29dkd82hs91kd93kd82hs91kd83ks
IV_LENGTH=16
NODE_ENV=test
```

The 32-byte length satisfies the check at [src/utils/helpers.ts:20-22](../../../repos/connection-studio/backend/src/utils/helpers.ts#L20-L22). Committing a test-only key is safe because the file is scoped to jest and documented as a non-secret.

---

### File: `backend/jest.config.ts`

**BEFORE** ([jest.config.ts:25-26](../../../repos/connection-studio/backend/jest.config.ts#L25-L26))

```ts
  // Load .env before all tests
  setupFiles: ['dotenv/config'],
```

**AFTER**

```ts
  // Load committed .env.test so the suite is reproducible from a clean checkout.
  // (Do not fall back to .env — that hides missing test-env config.)
  setupFiles: ['<rootDir>/test/jest.env.ts'],
```

Then add a new `backend/test/jest.env.ts`:

```ts
import * as dotenv from 'dotenv';
import * as path from 'node:path';

dotenv.config({ path: path.resolve(__dirname, '..', '.env.test') });
```

Explicitly loading `.env.test` — rather than relying on `dotenv/config`'s implicit `.env` — makes the source of test env obvious and stops a missing `.env` on a fresh clone from silently producing six failing suites.

---

### File: `backend/package.json`

**BEFORE** ([backend/package.json:41](../../../repos/connection-studio/backend/package.json#L41), and the corresponding devDependency line)

```json
    "@prisma/client": "^6.12.0",
```

```json
    "prisma": "^6.13.0",
```

**AFTER**

_(both lines removed)_

The `@prisma/client` and `prisma` packages are not imported anywhere in `src/` (verified by `grep -rn "prisma" src` finding zero matches). Removing them prevents future dependency sweeps from misdiagnosing a Prisma-schema gap that does not exist.

---

### File: `package.json` (root)

**BEFORE** ([package.json:10](../../../repos/connection-studio/package.json#L10))

```json
    "backend:lint:eslint": "cd backend && npm ci && npx prisma generate && npm run lint",
```

**AFTER**

```json
    "backend:lint:eslint": "cd backend && npm ci && npm run lint",
```

`npx prisma generate` has no tracked `schema.prisma` to consume; the step is a no-op today. Removing it aligns the script with reality.

---

### File: `backend/README.md`

**BEFORE**

_(no test-env section)_

**AFTER** (append)

```md
## Running Tests

Jest loads `backend/.env.test` via `test/jest.env.ts`. That file is committed and contains
a non-secret 32-byte `ENCRYPTION_KEY` and `IV_LENGTH=16` used only by the unit suite.

If you add a new module that reads env at import time, add a placeholder value to
`.env.test` so all suites remain loadable from a clean checkout.
```

Documents the test-env expectation so future contributors do not repeat the diagnosis in issue #91.

---

**Impact of Track A alone**: on a clean checkout, `cd backend && npm ci && npm test` reports 12 of 12 suites loaded. `@prisma/client` is no longer declared as a dependency, so `npm ci` output no longer implies Prisma is in use. No source-code behaviour changes.

---

## Track B — Reproducibility Hardening (PR 2 of 2)

### Step B1 — Defer encryption-key validation off module-import

**File:** `backend/src/utils/helpers.ts`

**BEFORE** ([src/utils/helpers.ts:16-22](../../../repos/connection-studio/backend/src/utils/helpers.ts#L16-L22))

```ts
const { ENCRYPTION_KEY, IV_LENGTH } = process.env;

const key = Buffer.from(ENCRYPTION_KEY!, 'utf8');

if (key.length !== 32) {
  throw new Error('ENCRYPTION_KEY must be 32 bytes for aes-256-cbc');
}
```

**AFTER**

```ts
let cachedKey: Buffer | null = null;

function getKey(): Buffer {
  if (cachedKey) return cachedKey;
  const raw = process.env.ENCRYPTION_KEY;
  if (!raw) {
    throw new Error('ENCRYPTION_KEY is not set');
  }
  const buf = Buffer.from(raw, 'utf8');
  if (buf.length !== 32) {
    throw new Error('ENCRYPTION_KEY must be 32 bytes for aes-256-cbc');
  }
  cachedKey = buf;
  return cachedKey;
}
```

Then, inside `encrypt` and `decrypt`, replace the top-level `key` reference with `getKey()`. This removes the top-level side effect, so importing `helpers.ts` never throws. Misconfiguration is still caught — on first `encrypt`/`decrypt` call — but tests that never call those functions can now load cleanly.

### Step B2 — Enforce length at boot in the config validator

**File:** `backend/src/config/env.validation.ts`

**BEFORE** ([src/config/env.validation.ts:38-39](../../../repos/connection-studio/backend/src/config/env.validation.ts#L38-L39))

```ts
  @IsString()
  ENCRYPTION_KEY: string;
```

**AFTER**

```ts
  @IsString()
  @Length(32, 32, { message: 'ENCRYPTION_KEY must be exactly 32 bytes' })
  ENCRYPTION_KEY: string;
```

Add `Length` to the `class-validator` import at the top of the file. Combined with removing `skipMissingProperties: true` for this field (or adding an explicit presence check), a misconfigured production deployment fails at Nest boot rather than at first encrypt.

### Step B3 — Clean-checkout smoke workflow

**File:** `.github/workflows/backend-smoke.yml` (new)

```yaml
name: backend-smoke
on:
  pull_request:
    paths:
      - 'backend/**'
      - '.github/workflows/backend-smoke.yml'
jobs:
  smoke:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - name: Fail if any local artefacts exist
        run: test ! -d backend/generated || (echo "unexpected local artefacts" && exit 1)
      - run: cd backend && npm ci
      - run: cd backend && npm run build
      - run: cd backend && npm test -- --ci
```

Runs on every PR that touches `backend/`. `npm test -- --ci` fails on any suite load error, closing the "partial-green" gap.

### Step B4 — Confirm the reusable `node-ci.yml` gate

Audit `node-ci.yml@dev` to ensure it invokes `npm test` for `backend/` and does not pass `--passWithNoTests`. If it does, remove that flag. No code snippet — this is a review step against the shared workflow file in the `.github` repo.

**Caveat / ordering:** B3 depends on B1 landing first, because a fresh CI runner has no `.env` — Track A's `.env.test` handles that, so Track A must be merged before Track B's smoke workflow is added.

---

## Data migration

Not applicable. No schema changes in either track.

---

## Test Cases

### Unit tests (new)

**File:** `backend/test/utils/helpers.spec.ts` (new — Track B only)

```ts
import { encrypt, decrypt } from '../../src/utils/helpers';

describe('utils/helpers — deferred key validation', () => {
  it('imports without throwing when ENCRYPTION_KEY is unset', () => {
    // If the module top-level throws, this file would never execute.
    expect(true).toBe(true);
  });

  it('throws only when encrypt is called with no ENCRYPTION_KEY', () => {
    const previous = process.env.ENCRYPTION_KEY;
    delete process.env.ENCRYPTION_KEY;
    expect(() => encrypt('hello')).toThrow(/ENCRYPTION_KEY is not set/);
    process.env.ENCRYPTION_KEY = previous;
  });

  it('round-trips a value with a 32-byte key', () => {
    process.env.ENCRYPTION_KEY = 'a'.repeat(32);
    process.env.IV_LENGTH = '16';
    const cipher = encrypt('hello');
    expect(decrypt(cipher)).toBe('hello');
  });
});
```

### Integration / E2E scenarios

1. On a fresh clone, run `cd backend && npm ci && npm test`. Expect: 12 of 12 suites loaded, no `ENCRYPTION_KEY must be 32 bytes` errors.
2. Delete `.env.test` locally, re-run `npm test`. Expect: clear failure that names the missing file (Track A: a jest bootstrap error; Track B: a Nest-boot validation error citing `ENCRYPTION_KEY`).
3. Set `ENCRYPTION_KEY` to a 10-byte value and run `npm run start`. Expect (Track B): Nest fails to boot with a validator error citing `ENCRYPTION_KEY must be exactly 32 bytes`.
4. Push a branch that removes `.env.test`. Expect: the new `backend-smoke.yml` workflow fails the PR.

### Data migration validation SQL

Not applicable.

### Manual / UAT checks

| # | Scenario | Steps | Expected Result |
|---|----------|-------|-----------------|
| 1 | Clean-checkout test run | Fresh `git clone`, `cd backend`, `npm ci`, `npm test` | 12 of 12 suites load; all pass |
| 2 | Missing test env is loud | Delete `.env.test`, `npm test` | Fails with a clear "missing .env.test" or ENCRYPTION_KEY error, not a silent partial-green |
| 3 | Prisma removed | `grep -n prisma backend/package.json` | No matches |
| 4 | Root script cleaned | `grep -n "prisma generate" package.json` | No matches |
| 5 | Production boot with wrong-length key (Track B) | Start Nest with `ENCRYPTION_KEY=short` | Boot fails on env validation, before any request is served |
| 6 | Import-time safety (Track B) | Run a script that imports `utils/helpers` with no env | Import succeeds; only calling `encrypt` throws |

---

## Overall Impact of the Fix

| Area | Before | After |
|------|--------|-------|
| Jest suites loaded on clean checkout | 6 of 12 | 12 of 12 |
| Test-env source | Local `.env` (uncommitted) | Committed `backend/.env.test` |
| `helpers.ts` import behaviour (Track B) | Throws at import when env missing | Import safe; throws only on first `encrypt`/`decrypt` |
| Wrong-length key in prod (Track B) | Detected at first crypto call | Detected at Nest boot |
| Vestigial Prisma dep | `@prisma/client` + `prisma` declared, no-op `npx prisma generate` step | Removed |
| Clean-checkout CI (Track B) | Not enforced | `backend-smoke.yml` runs `npm ci && npm build && npm test --ci` |

---

## Fix Summary

The reporter opened issue #91 after a dependency-security sweep suggested that `backend/` could not be built or tested from a clean checkout. In a follow-up comment they corrected the record: the backend does not use Prisma at all — `nest build` passes on a clean tree, and `@prisma/client` is a dependency declared in `package.json` but never imported anywhere in `src/`. What remains is a genuine test-environment reproducibility gap: six of twelve jest suites fail to *load* on a fresh clone because `src/utils/helpers.ts` throws at module-import time when `ENCRYPTION_KEY` is not a 32-byte string, and jest loads its env from an un-tracked `.env` file rather than a committed test artefact.

**Track A** commits a `backend/.env.test` file containing a non-secret 32-byte key, changes jest's setup to load it explicitly, removes the vestigial `@prisma/client` dependency and the no-op `npx prisma generate` step from the root lint script, and adds a short README note explaining the test-env contract. It is a config-and-docs PR with no source-code behaviour change, and it closes the immediate "half the suites silently skipped" hole.

**Track B** hardens the underlying design so this class of failure cannot recur: it moves the encryption-key validation out of `helpers.ts`'s top level (making imports safe), pushes a proper length check into the Nest env validator (so a misconfigured production deployment fails at boot, not at first crypto call), and adds a `backend-smoke.yml` CI workflow that builds and tests `backend/` from a clean runner with no local artefacts.

Neither track needs a schema migration or any frontend change. Track A is safe to ship first; Track B lands on top once Track A is green on `dev`.
