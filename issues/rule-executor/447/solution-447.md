## Complete Fix — Exact Code Changes

Track A can merge independently and immediately — it is two one-line changes in `tazama-lf/rule-studio-example` and has no dependency on Track B. Track B must land as a single coordinated PR into `dev` on `tazama-lf/rule-executer`; it should not be split because several changes interact (library versions, `Dockerfile` auth mechanism, and the `rule.ts` decision all affect whether the same build succeeds).

---

### Track A — Switch clone target (PR 1 of 2)

#### `.github/workflows/deploy.yml` — `tazama-lf/rule-studio-example`

**Line 39. BEFORE:**
```yaml
          git clone https://github.com/tazama-lf/rule-executer -b feat-paysys
```

**AFTER:**
```yaml
          git clone https://github.com/tazama-lf/rule-executer -b dev
```

This makes every dev/build-and-publish-triggered deploy clone the maintained `dev` branch instead of the diverged `feat-paysys` fork.

---

#### `.github/workflows/deploy-to-uat.yml` — `tazama-lf/rule-studio-example`

**Line 36. BEFORE:**
```yaml
          git clone https://github.com/tazama-lf/rule-executer -b feat-paysys
```

**AFTER:**
```yaml
          git clone https://github.com/tazama-lf/rule-executer -b dev
```

This makes the UAT deploy also clone `dev`. Once `dev` is promoted to `main` via a proper PR, this can be updated to `-b main` for production-grade traceability.

**Impact of Track A alone:** All future Rule Studio deploys get the 35 upstream `dev` commits (including security fixes from PRs #443, #445, #446). Containers already running are unaffected until they are rebuilt.

---

### Track B — Reconcile feat-paysys with dev (PR 2 of 2)

All changes below apply to the `tazama-lf/rule-executer` repository, targeting the `dev` branch.

---

#### Step B1 — Dockerfile: remove hardcoded infrastructure, restore BuildKit secret

**File:** `Dockerfile`

**BEFORE (feat-paysys lines 44–82):**
```dockerfile
ENV FUNCTION_NAME="rule-executer"
ENV RULE_VERSION="2.1.0"
ENV RULE_NAME="rule-901"
ENV NODE_ENV=production
ENV MAX_CPU=1

# Apm
ENV APM_ACTIVE=false
ENV APM_URL=http://apm-server.development.svc.cluster.local:8200/
ENV APM_SECRET_TOKEN=
ENV APM_SERVICE_NAME=rule-901

# Database
ENV RAW_HISTORY_DATABASE=raw_history
ENV RAW_HISTORY_DATABASE_HOST=10.10.80.18
ENV RAW_HISTORY_DATABASE_PORT=15432
ENV RAW_HISTORY_DATABASE_USER=postgres
ENV RAW_HISTORY_DATABASE_PASSWORD=postgres
ENV RAW_HISTORY_DATABASE_CERT_PATH=/usr/local/share/ca-certificates/ca-certificates.crt

ENV CONFIGURATION_DATABASE=configuration
ENV CONFIGURATION_DATABASE_HOST=10.10.80.18
ENV CONFIGURATION_DATABASE_PORT=15432
ENV CONFIGURATION_DATABASE_USER=postgres
ENV CONFIGURATION_DATABASE_PASSWORD=postgres
ENV CONFIGURATION_DATABASE_CERT_PATH=/usr/local/share/ca-certificates/ca-certificates.crt

ENV EVENT_HISTORY_DATABASE=event_history
ENV EVENT_HISTORY_DATABASE_HOST=10.10.80.18
ENV EVENT_HISTORY_DATABASE_PORT=15432
ENV EVENT_HISTORY_DATABASE_USER=postgres
ENV EVENT_HISTORY_DATABASE_PASSWORD=postgres
ENV CONFIGURATION_DATABASE_CERT_PATH=/usr/local/share/ca-certificates/ca-certificates.crt

#Nats
ENV STARTUP_TYPE=nats
ENV SERVER_URL=10.10.80.18:14222
```

**AFTER (align with `dev`):**
```dockerfile
ENV FUNCTION_NAME="rule-executer"
ENV RULE_VERSION="2.1.0"
ENV RULE_NAME="901"
ENV NODE_ENV=production
ENV MAX_CPU=1

# Apm
ENV APM_ACTIVE=true
ENV APM_URL=http://apm-server.development.svc.cluster.local:8200/
ENV APM_SECRET_TOKEN=
ENV APM_SERVICE_NAME=rule-901

# Database
ENV RAW_HISTORY_DATABASE=raw_history
ENV RAW_HISTORY_DATABASE_HOST=
ENV RAW_HISTORY_DATABASE_PORT=
ENV RAW_HISTORY_DATABASE_USER=
ENV RAW_HISTORY_DATABASE_PASSWORD=
ENV RAW_HISTORY_DATABASE_CERT_PATH=/usr/local/share/ca-certificates/ca-certificates.crt

ENV CONFIGURATION_DATABASE=configuration
ENV CONFIGURATION_DATABASE_HOST=
ENV CONFIGURATION_DATABASE_PORT=
ENV CONFIGURATION_DATABASE_USER=
ENV CONFIGURATION_DATABASE_PASSWORD=
ENV CONFIGURATION_DATABASE_CERT_PATH=/usr/local/share/ca-certificates/ca-certificates.crt

ENV EVENT_HISTORY_DATABASE=event_history
ENV EVENT_HISTORY_DATABASE_HOST=
ENV EVENT_HISTORY_DATABASE_PORT=
ENV EVENT_HISTORY_DATABASE_USER=
ENV EVENT_HISTORY_DATABASE_PASSWORD=
ENV EVENT_HISTORY_DATABASE_CERT_PATH=/usr/local/share/ca-certificates/ca-certificates.crt

#Nats
ENV STARTUP_TYPE=nats
ENV SERVER_URL=0.0.0.0:4222
```

Also restore the BuildKit secret mount in the `npm ci` instructions. **BEFORE (`feat-paysys`):**
```dockerfile
ARG GH_TOKEN
RUN npm ci --ignore-scripts
```

**AFTER (align with `dev`):**
```dockerfile
RUN --mount=type=secret,id=GH_TOKEN,env=GH_TOKEN npm ci --ignore-scripts
```

Repeat for both Stage 1 (builder) and Stage 2 (dep-resolver). Hardcoded infrastructure values override any runtime env vars injected by `docker run`, silently breaking rule containers in non-UAT environments.

---

#### Step B2 — package.json: align library versions and restore rule placeholder

**File:** `package.json`

**BEFORE (feat-paysys):**
```json
"dependencies": {
  "@tazama-lf/frms-coe-lib": "0.0.1-psl.0",
  "@tazama-lf/frms-coe-startup-lib": "3.0.2-rc.5",
  "dotenv": "^17.2.3",
  "node-cache": "^5.1.2",
  "ts-node": "^10.9.2",
  "tslib": "^2.8.1"
}
```

**AFTER (align with `dev`):**
```json
"dependencies": {
  "@tazama-lf/frms-coe-lib": "8.2.0-rc.6",
  "@tazama-lf/frms-coe-startup-lib": "3.1.0-rc.8",
  "dotenv": "^17.2.3",
  "node-cache": "^5.1.2",
  "rule": "npm:@tazama-lf/rule-901@4.0.0-rc.6",
  "ts-node": "^10.9.2",
  "tslib": "^2.8.1"
}
```

`0.0.1-psl.0` is a Paysys-internal pre-release that is not available from the `tazama-lf` npm registry. Restoring the `rule` placeholder entry makes the `deploy.yml` `sed` substitution (line 66) functional again.

---

#### Step B3 — .npmrc: remove psl-copilot scope if not needed on dev

**File:** `.npmrc`

**BEFORE (feat-paysys):**
```
# SPDX-License-Identifier: Apache-2.0
@frmscoe:registry=https://npm.pkg.github.com
@tazama-lf:registry=https://npm.pkg.github.com
@psl-copilot:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${GH_TOKEN}
```

**AFTER (if `@psl-copilot` packages not needed on `dev`):**
```
# SPDX-License-Identifier: Apache-2.0
@frmscoe:registry=https://npm.pkg.github.com
@tazama-lf:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${GH_TOKEN}
```

If any Studio rule package is published under `@psl-copilot`, keep the line and add it to `dev`'s `.npmrc` via a reviewed PR. The decision must be explicit — not inherited from an unreviewed branch.

---

#### Step B4 — src/controllers/execute.ts: strip debug log statements and any cast

**File:** `src/controllers/execute.ts`

Remove all `[L##]`-prefixed log messages added in `feat-paysys`. The `dev` version of this file is clean; the reconciliation is to confirm `dev` already has the correct version. Key reversal: the `feat-paysys` version also contains:

```typescript
ruleRes = await handleTransaction(request, determineOutcome, ruleRes, loggerService, ruleConfig, databaseManager as any);
```

**AFTER (align with dev):**
```typescript
ruleRes = await handleTransaction(request, determineOutcome, ruleRes, loggerService, ruleConfig, databaseManager);
```

The `as any` cast suppresses a type error that likely indicates a type mismatch between the `feat-paysys` version of `frms-coe-lib` and the function signature in `rule/lib`. On `dev` with the correct library version this cast is not needed.

---

#### Step B5 — src/controllers/rule.ts: decision point

**File:** `src/controllers/rule.ts` (exists only on `feat-paysys`)

This file provides a local `handleTransaction` that handles `BaseMessage` transactions with amount-band logic. The `feat-paysys` version of `execute.ts` still imports `handleTransaction` from `'rule/lib'` (confirmed: `import { handleTransaction } from 'rule/lib'` at line 6 of `feat-paysys/src/controllers/execute.ts`), so `rule.ts` is **not imported by the executer itself** — it appears to be an experimental/standalone file.

**If the `BaseMessage` path is not needed by the current pipeline (recommended):**
Do not add `src/controllers/rule.ts` to `dev`. Delete it from `feat-paysys` before deleting the branch.

**If `BaseMessage` handling is required:**
Clean the file (remove all `console.log` statements) and open a separate PR to `dev` with tests. The cleaned version would look like:

```typescript
// BEFORE (feat-paysys) — contains debug output:
console.log("hello bhai the req ", req)
console.log("hello bhai the trxn ", transaction)
console.log('hi, this is trx', JSON.stringify(transaction))
console.log('hello bhai')
console.log('hi, this is trx', JSON.stringify(amountRaw))

// AFTER — remove all console.log lines, keep logic only
```

---

#### Step B6 — Delete simple-rule2-test.js

**File:** `simple-rule2-test.js` (exists only on `feat-paysys`)

Delete. This is a one-off test/debug file not present on `dev` and not part of any test suite.

---

#### Step B7 — Delete feat-paysys branch

After the Track B PR merges into `dev`:
```bash
git push origin --delete feat-paysys
```

---

### Test Cases

#### Unit tests (new tests to add)

```typescript
// Verify deploy.yml clones correct branch
// (shell-level test — verify by grepping the workflow file)
import { readFileSync } from 'fs';

test('deploy.yml clones dev branch', () => {
  const content = readFileSync('.github/workflows/deploy.yml', 'utf8');
  expect(content).toContain('-b dev');
  expect(content).not.toContain('-b feat-paysys');
});

test('deploy-to-uat.yml clones dev branch', () => {
  const content = readFileSync('.github/workflows/deploy-to-uat.yml', 'utf8');
  expect(content).toContain('-b dev');
  expect(content).not.toContain('-b feat-paysys');
});
```

#### Integration / E2E test scenarios

1. Trigger `deploy.yml` workflow via `workflow_dispatch` in a Rule Studio–generated rule repo.
2. In the workflow log, confirm the clone step outputs `Cloning into 'rule-executer'...` followed by a commit SHA from `dev` (not `feat-paysys`).
3. Confirm the built Docker image contains the correct `frms-coe-lib` version: `docker run --rm <image> node -p "require('/home/app/node_modules/@tazama-lf/frms-coe-lib/package.json').version"` — should return `8.2.0-rc.6`.
4. Confirm APM is enabled by default: `docker inspect <container> | jq '.[0].Config.Env'` — should show `APM_ACTIVE=true`.
5. Confirm no `console.log("hello bhai")` output appears in rule container logs during a test transaction.
6. Submit a test transaction through NATS; confirm the rule container produces a valid `RuleResult` forwarded to the Typology Processor.

#### Manual / UAT checks

| # | Scenario | Steps | Expected Result |
|---|---|---|---|
| 1 | Verify clone target | Trigger `deploy.yml`; inspect workflow log step "Clone Rule Executer Repo" | Log shows `dev` branch SHA, not `feat-paysys` |
| 2 | Verify library version | `docker run --rm <built-image> node -p "require('/home/app/node_modules/@tazama-lf/frms-coe-lib/package.json').version"` | Returns `8.2.0-rc.6` |
| 3 | Verify APM enabled | `docker inspect <container>` → env | `APM_ACTIVE=true` |
| 4 | Verify no hardcoded IP | `docker inspect <container>` → env | `RAW_HISTORY_DATABASE_HOST` is empty (overridden at `docker run` time) |
| 5 | Verify no debug output | Run a test transaction; check container logs | No `[L##]`-prefixed messages, no `hello bhai` output |
| 6 | End-to-end rule execution | Submit transaction matching rule config; observe Typology Processor receives result | `RuleResult` with correct `subRuleRef` and `cfg` produced |
| 7 | feat-paysys deleted | `gh api repos/tazama-lf/rule-executer/branches/feat-paysys` | 404 response |

---

### Overall Impact of the Fix

| Area | Before | After |
|---|---|---|
| Clone target | `feat-paysys` (diverged, unreviewed) | `dev` (maintained, reviewed) |
| Upstream fix coverage | Missing 35 `dev` commits | All `dev` commits included |
| APM | Disabled (`APM_ACTIVE=false`) | Enabled by default |
| Infrastructure config | Hardcoded UAT IPs baked into image | Empty defaults; all values injected at runtime |
| Debug output | `console.log("hello bhai…")` in production containers | No debug output |
| Library version | `frms-coe-lib 0.0.1-psl.0` (internal, unavailable to `tazama-lf` runners) | `frms-coe-lib 8.2.0-rc.6` (published, audited) |
| `sed` patching | Silent no-op (no `rule` key in `package.json`) | Functional substitution |
| Branch hygiene | Active diverged branch used as release artifact | Branch deleted; `dev` is the sole source of truth |

---

### Fix Summary

The Rule Studio deploy pipeline was cloning a feature branch (`feat-paysys`) that had diverged significantly from the `dev` mainline and was being used as an unofficial release artifact. Every Studio-generated rule was therefore shipping executer code that was 35 commits behind `dev` (missing security fixes and bug fixes), while also carrying 28 unreviewed commits that included hardcoded UAT infrastructure IPs baked into the Docker image, a globally disabled APM configuration, a pinned pre-release library from an internal Paysys registry unavailable to standard `tazama-lf` runners, and raw `console.log` debug output visible in production container logs.

Track A is a two-line change: replace `-b feat-paysys` with `-b dev` in the two deploy workflow files in `tazama-lf/rule-studio-example`. This immediately redirects all new builds to the maintained branch and is safe to ship without any application code changes.

Track B is a full reconciliation of `feat-paysys` with `dev`: removing hardcoded infrastructure, restoring BuildKit secret handling in the Dockerfile, aligning library versions, stripping debug log statements, making a documented decision about the experimental `rule.ts` `BaseMessage` handler, and deleting the branch once the PR merges. This track requires a parallel audit of the 28 branch-only commits to determine which (if any) represent required adaptations versus UAT-only hacks.

Shipping Track A first and Track B second is the recommended sequence. Track A stops the bleeding immediately; Track B closes the technical debt properly.
