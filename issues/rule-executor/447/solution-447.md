## Complete Fix — Exact Code Changes

The two tracks are independent and can be merged in either order. Track A is a two-line change in the `rule-studio-example` deployment workflows that stops targeting an obsolete branch of `rule-executer`; it ships as a standalone PR. Track B is a coordinated cleanup PR in `rule-executer` that reconciles `feat-paysys` with `dev` (or, equivalently, deletes `feat-paysys` after porting anything worth keeping). Track B must land as a single PR because the changes are cross-cutting (Dockerfile, package.json, .npmrc, controllers, husky hook, dead files) and are only meaningful as a set.

---

### Track A — Retarget deploy workflows to `dev` (PR 1 of 2)

Both workflow files in `rule-studio-example` clone `rule-executer` from the abandoned `feat-paysys` branch. Track A retargets them to `dev`. The change must be applied on **both** `main`/`dev` (line 39) **and** `feat-paysys` (line 41) of `rule-studio-example`, since `feat-paysys` in that repo is 19 commits ahead and still carries the same hardcoded clone plus additional Docker Hub push logic.

#### File: `.github/workflows/deploy.yml` (rule-studio-example)

Line 39 on `main`/`dev`; line 41 on `feat-paysys`.

BEFORE:
```yaml
      - name: Clone Rule Executer Repo
        run: |
          echo "Cloning Rule Executer repo and removing package-lock.json file..."
          git clone https://github.com/tazama-lf/rule-executer -b feat-paysys
          cd rule-executer
          rm -f package-lock.json
          cd ..
          echo "Clone completed."
```

AFTER:
```yaml
      - name: Clone Rule Executer Repo
        run: |
          echo "Cloning Rule Executer repo and removing package-lock.json file..."
          git clone https://github.com/tazama-lf/rule-executer -b dev
          cd rule-executer
          rm -f package-lock.json
          cd ..
          echo "Clone completed."
```

This is a one-token change (`feat-paysys` -> `dev`). It stops the pipeline from resurrecting an abandoned branch on every deploy and pins the source of truth to `dev`, which is the branch Tazama actually maintains.

#### File: `.github/workflows/deploy-to-uat.yml` (rule-studio-example)

Line 36 on `main`/`dev`; equivalent line on `feat-paysys`.

BEFORE:
```yaml
      - name: Clone Rule Executer Repo
        run: |
          echo "Cloning Rule Executer repo and removing package-lock.json file..."
          git clone https://github.com/tazama-lf/rule-executer -b feat-paysys
          cd rule-executer
          rm -f package-lock.json
          cd ..
          echo "Clone completed."
```

AFTER:
```yaml
      - name: Clone Rule Executer Repo
        run: |
          echo "Cloning Rule Executer repo and removing package-lock.json file..."
          git clone https://github.com/tazama-lf/rule-executer -b dev
          cd rule-executer
          rm -f package-lock.json
          cd ..
          echo "Clone completed."
```

Note: `deploy-to-uat.yml` is Paysys-specific. Line 104 hardcodes `npm install --save-exact rule@npm:@psl-copilot/$RULE_NAME@latest`, meaning the UAT workflow installs rule packages from the `@psl-copilot` scope. Retargeting the clone to `dev` does not remove that hardcoding — `deploy-to-uat.yml` will still resolve rule packages from `@psl-copilot`. Track A only fixes the clone target; scope hardcoding is a separate concern.

#### Verify the authoritative template repo

`tazama-lf/rule-studio-devtestops` sets in `.env.sample`:
```
GITHUB_TEMPLATE_REPO=rule-template
GITHUB_TEMPLATE_OWNER=psl-copilot
```

The production template Rule Studio actually clones may be `psl-copilot/rule-template` (a fork or copy of `tazama-lf/rule-studio-example`) rather than `tazama-lf/rule-studio-example`. Before closing Track A, verify which template repo the Rule Studio backend authoritatively pulls from, and apply the same two-line fix there if it also carries `-b feat-paysys`.

#### Impact of Track A alone

Every new rule generated after Track A ships will start from `rule-executer@dev`. Existing generated rules that were cloned from `feat-paysys` are unaffected until they are regenerated. Track A does not remove any code from `rule-executer` and does not touch `feat-paysys` — it just stops feeding new consumers into it.

---

### Track B — Reconcile `feat-paysys` with `dev` (PR 2 of 2)

Track B lands as one PR against `rule-executer` `dev`. Every step below references the current HEAD of `feat-paysys` as the BEFORE and the current HEAD of `dev` as the AFTER. The end state is: (a) `dev` remains authoritative, (b) `feat-paysys` is deleted, (c) any Paysys-specific bits that must remain live inside `dev` in an env-driven, non-hardcoded form.

#### Step B1 — Dockerfile: remove hardcoded UAT values and restore build-secret pattern

File: `Dockerfile`

Multiple hunks. Locations are approximate (feat-paysys line numbers).

BEFORE (feat-paysys, lines 15-16, 27):
```dockerfile
COPY .npmrc ./
ARG GH_TOKEN

RUN npm ci --ignore-scripts
...
ARG GH_TOKEN
RUN npm ci --omit=dev --ignore-scripts
```

AFTER (dev):
```dockerfile
COPY .npmrc ./
RUN --mount=type=secret,id=GH_TOKEN,env=GH_TOKEN npm ci --ignore-scripts
...
RUN --mount=type=secret,id=GH_TOKEN,env=GH_TOKEN npm ci --omit=dev --ignore-scripts
```

BEFORE (feat-paysys, lines 47-49, 52):
```dockerfile
ENV FUNCTION_NAME="rule-executer"
ENV RULE_VERSION="2.1.0"
ENV RULE_NAME="rule-901"
...
ENV APM_ACTIVE=false
```

AFTER (dev):
```dockerfile
ENV FUNCTION_NAME="rule-executer"
ENV RULE_VERSION="2.1.0"
ENV RULE_NAME="901"
...
ENV APM_ACTIVE=true
```

BEFORE (feat-paysys, lines 60-77, DB blocks):
```dockerfile
ENV RAW_HISTORY_DATABASE_HOST=10.10.80.18
ENV RAW_HISTORY_DATABASE_PORT=15432
ENV RAW_HISTORY_DATABASE_USER=postgres
ENV RAW_HISTORY_DATABASE_PASSWORD=postgres
...
ENV CONFIGURATION_DATABASE_HOST=10.10.80.18
ENV CONFIGURATION_DATABASE_PORT=15432
ENV CONFIGURATION_DATABASE_USER=postgres
ENV CONFIGURATION_DATABASE_PASSWORD=postgres
...
ENV EVENT_HISTORY_DATABASE_HOST=10.10.80.18
ENV EVENT_HISTORY_DATABASE_PORT=15432
ENV EVENT_HISTORY_DATABASE_USER=postgres
ENV EVENT_HISTORY_DATABASE_PASSWORD=postgres
```

AFTER (dev):
```dockerfile
ENV RAW_HISTORY_DATABASE_HOST=
ENV RAW_HISTORY_DATABASE_PORT=
ENV RAW_HISTORY_DATABASE_USER=
ENV RAW_HISTORY_DATABASE_PASSWORD=
...
ENV CONFIGURATION_DATABASE_HOST=
ENV CONFIGURATION_DATABASE_PORT=
ENV CONFIGURATION_DATABASE_USER=
ENV CONFIGURATION_DATABASE_PASSWORD=
...
ENV EVENT_HISTORY_DATABASE_HOST=
ENV EVENT_HISTORY_DATABASE_PORT=
ENV EVENT_HISTORY_DATABASE_USER=
ENV EVENT_HISTORY_DATABASE_PASSWORD=
```

BEFORE (feat-paysys, line 86):
```dockerfile
ENV SERVER_URL=10.10.80.18:14222
```

AFTER (dev):
```dockerfile
ENV SERVER_URL=0.0.0.0:4222
```

Framing note: in the Rule Studio pipeline these values are overridden at `docker run` time by `-e` flags the workflow already supplies, so the hardcoded IPs do **not** break the pipeline. They matter when someone runs the built image outside the workflow (local test, ad-hoc container). Removing the hardcoding removes a footgun; it does not unbreak the pipeline.

#### Step B2 — package.json: align versions and dependencies with dev

File: `package.json`

BEFORE (feat-paysys):
```json
{
  "name": "rule-executer",
  "version": "3.0.0",
  ...
  "scripts": {
    ...
    "cleanup": "npx rimraf lib node_modules coverage package-lock.json",
    ...
  },
  "dependencies": {
    "@tazama-lf/frms-coe-lib": "0.0.1-psl.0",
    "@tazama-lf/frms-coe-startup-lib": "3.0.2-rc.5",
    "dotenv": "^17.2.3",
    "node-cache": "^5.1.2",
    "ts-node": "^10.9.2",
    "tslib": "^2.8.1"
  }
}
```

AFTER (dev):
```json
{
  "name": "rule-executer",
  "version": "4.0.0-rc.2",
  ...
  "scripts": {
    ...
    "cleanup": "rm -rf build template jest.config.js jest.config.js.map node_modules package-lock.json",
    ...
  },
  "dependencies": {
    "@tazama-lf/frms-coe-lib": "8.2.0-rc.6",
    "@tazama-lf/frms-coe-startup-lib": "3.1.0-rc.8",
    "dotenv": "^17.2.3",
    "node-cache": "^5.1.2",
    "rule": "npm:@tazama-lf/rule-901@4.0.0-rc.6",
    "ts-node": "^10.9.2",
    "tslib": "^2.8.1"
  },
  "overrides": {
    "uuid": "^11.1.1",
    "@opentelemetry/core": "^2.8.0"
  }
}
```

Key changes: `frms-coe-lib` moves from the private `0.0.1-psl.0` build to the public `8.2.0-rc.6`; `frms-coe-startup-lib` moves from `3.0.2-rc.5` (a real published version, not the psl fork) to `3.1.0-rc.8`; the `rule` dependency placeholder is restored so pipelines and local builds work; `overrides` block is picked up; version bumps to `4.0.0-rc.2`; cleanup script switches from `rimraf lib` to `rm -rf build template ...`.

#### Step B3 — .npmrc: decide on `@psl-copilot` scope

File: `.npmrc`

BEFORE (feat-paysys):
```
# SPDX-License-Identifier: Apache-2.0
@frmscoe:registry=https://npm.pkg.github.com
@tazama-lf:registry=https://npm.pkg.github.com
@psl-copilot:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${GH_TOKEN}
```

AFTER (dev, current):
```
# SPDX-License-Identifier: Apache-2.0
@frmscoe:registry=https://npm.pkg.github.com
@tazama-lf:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${GH_TOKEN}
```

Two options, decision needed at PR review:

- **Option 3a (keep `@psl-copilot` on dev).** Add the `@psl-copilot:registry=...` line to `dev`'s `.npmrc`. This is required if `rule-studio-example`'s `deploy-to-uat.yml` (line 104: `npm install --save-exact rule@npm:@psl-copilot/$RULE_NAME@latest`) must keep working after Track A retargets its clone to `dev`.
- **Option 3b (remove).** Leave `dev`'s `.npmrc` as-is (no `@psl-copilot`). Then `deploy-to-uat.yml` must be updated in a follow-up to install from `@tazama-lf` (or another agreed scope) instead of `@psl-copilot`.

Pick 3a if UAT deploys must continue unbroken on merge day; pick 3b if the plan is to remove the `@psl-copilot` scope entirely and follow up in `rule-studio-example`.

#### Step B4 — src/controllers/execute.ts: remove `[L##]` logs and the `as any` cast

File: `src/controllers/execute.ts`

The `feat-paysys` copy is peppered with debug-style logger calls of the form `loggerService.log('[L16] ...', ...)`, plus a `databaseManager as any` cast at the `handleTransaction` call (line 119). `dev`'s copy is clean.

BEFORE (feat-paysys, selected lines):
```typescript
loggerService.log('[L16] Function entry - Starting execute request handler', context, configuration.functionName);
...
loggerService.log('[L22] Starting request parsing and validation', context, configuration.functionName);
...
loggerService.log(`[L32] Request transaction data: ${JSON.stringify(message.transaction)}`, context, configuration.functionName);
...
loggerService.log('[L35] Request parsing completed successfully', context, configuration.functionName);
...
const failMessage = '[L38] Failed to parse execution request.';
loggerService.error(failMessage, err, context, configuration.functionName);
loggerService.log('[L40] Early exit due to request parsing failure', context, configuration.functionName);
```

AFTER (dev):
```typescript
loggerService.log('Start - Handle execute request', context, configuration.functionName);
...
// (parsing block: no debug logs)
...
const failMessage = 'Failed to parse execution request.';
loggerService.error(failMessage, err, context, configuration.functionName);
loggerService.log('End - Handle execute request', context, configuration.functionName);
```

BEFORE (feat-paysys, line 119):
```typescript
ruleRes = await handleTransaction(request, determineOutcome, ruleRes, loggerService, ruleConfig, databaseManager as any);
```

AFTER (dev):
```typescript
ruleRes = await handleTransaction(request, determineOutcome, ruleRes, loggerService, ruleConfig, databaseManager);
```

The `as any` cast was a workaround for a type mismatch that no longer exists once `frms-coe-lib` is bumped to `8.2.0-rc.6` (Step B2). Removing it restores type safety at the call boundary. The `[L##]`-prefixed logs were left-over debug prints; `dev` already has the clean version, so this step is really "adopt `dev`'s file verbatim".

#### Step B5 — src/controllers/rule.ts: delete (dead code)

File: `src/controllers/rule.ts`

This file exists only on `feat-paysys`. It is **never imported** — `execute.ts` imports `handleTransaction` from the `rule/lib` npm package (line 6: `import { handleTransaction } from 'rule/lib';`), not from `./rule`. The file's contents duplicate what `rule/lib` provides, with an additional `isBaseMessageTransaction` branch. Because nothing consumes it, it is orphaned dead code.

Action: **delete** `src/controllers/rule.ts`. Do not attempt to "merge it into dev". If the team wants `BaseMessage` handling in the shared `rule/lib`, they can open a separate feature PR against `frms-coe-lib`/`rule/lib` with proper tests. Deleting is the default; porting is out of scope for this reconciliation.

#### Step B6 — Delete `simple-rule2-test.js`

File: `simple-rule2-test.js`

A 124-line standalone script that lives at the repo root on `feat-paysys` only. It is not referenced by `jest.config.ts`, not required by `package.json` scripts, and not part of the test suite (its first line is `const { handleTransaction } = require('rule/lib');` — it is an ad-hoc harness). Delete it.

#### Step B7 — .husky/pre-commit: fix broken command

File: `.husky/pre-commit`

BEFORE (feat-paysys):
```
npx lint - staged
```

AFTER (dev):
```
npx lint-staged
```

The extra spaces on `feat-paysys` mean the pre-commit hook has been silently broken (it invokes `npx lint` with `-` and `staged` as arguments). Aligning with `dev` restores the hook.

#### Step B8 — src/config.ts: whitespace-only realignment

File: `src/config.ts`

BEFORE (feat-paysys):
```typescript
export type RuleExecutorConfig = Required<
  Pick<ManagerConfig, 'rawHistory' | 'eventHistory' | 'configuration' | 'localCacheConfig'>
>;
```

AFTER (dev):
```typescript
export type RuleExecutorConfig = Required<Pick<ManagerConfig, 'rawHistory' | 'eventHistory' | 'configuration' | 'localCacheConfig'>>;
```

Pure formatting difference. Include for parity with `dev` so a subsequent `prettier --check` is clean.

#### Step B9 — Delete the `feat-paysys` branch

Once the PR against `dev` is merged and CI is green, delete the abandoned branch so it cannot be re-targeted:

```bash
git push origin --delete feat-paysys
```

Ordering caveat: Step B9 must come **after** Track A ships. If `feat-paysys` is deleted while any Rule Studio workflow still points at it, the next deploy fails with `fatal: Remote branch feat-paysys not found`. The safe sequence is: Track A merged -> Track B merged -> `feat-paysys` deleted.

---

### Note on `deploy.yml` line 66 sed

An earlier draft of this document claimed the `sed` invocation around line 66 of `rule-studio-example/.github/workflows/deploy.yml` was "a silent no-op that breaks the pipeline". This is wrong. Lines 108-111 of the workflow run:

```yaml
npm pkg delete dependencies.rule
npm install --save-exact rule@npm:@$ORG/$RULE_NAME@latest
```

which installs the rule regardless of what the `sed` did or did not do. The `sed` is dead / cleanup-lag, not a functional bug. Restoring its placeholder is a nice-to-have for consistency, **not** blocking, and is not part of Track A or Track B.

---

## Test Cases

### Unit Tests

Track B changes controller code (`src/controllers/execute.ts`) and dependency versions, so unit tests should verify the clean logger path and the removed cast still compile and behave.

```typescript
// src/controllers/__tests__/execute.test.ts (add or extend)
describe('execute()', () => {
  it('logs the clean Start/End markers without [L##] prefixes', async () => {
    const spy = jest.spyOn(loggerService, 'log');
    await execute(validRequest);
    const messages = spy.mock.calls.map(([msg]) => msg);
    expect(messages).toContain('Start - Handle execute request');
    expect(messages).toContain('End - Handle execute request');
    expect(messages.some((m) => /\[L\d+\]/.test(String(m)))).toBe(false);
  });

  it('calls handleTransaction with databaseManager passed unchanged (no cast)', async () => {
    const spy = jest.fn().mockResolvedValue(baseRuleRes);
    jest.doMock('rule/lib', () => ({ handleTransaction: spy }));
    await execute(validRequest);
    const passedDbm = spy.mock.calls[0][5];
    expect(passedDbm).toBe(databaseManager);
  });
});
```

The husky change (B7) should be verified by `git commit`ing a `.ts` file with a lint issue and confirming the hook rejects it.

### Integration / E2E Scenarios

1. **Track A retarget verification.** In `rule-studio-example`, trigger the `deploy.yml` workflow for a test rule. Observe the "Clone Rule Executer Repo" step logs; confirm the clone URL ends in `-b dev`, not `-b feat-paysys`.
2. **UAT deploy still works.** Trigger `deploy-to-uat.yml` for a rule whose `@psl-copilot/$RULE_NAME` package exists. Confirm the install step at line 104 succeeds. If it fails with `E401`/`E404` for `@psl-copilot`, Option 3a (keep the scope in `dev`'s `.npmrc`) is required.
3. **Track B image build.** Run `docker build --secret id=GH_TOKEN,env=GH_TOKEN .` against the merged `dev`. Confirm the build succeeds using the `--mount=type=secret` pattern and does not leak `GH_TOKEN` into any layer.
4. **Runtime with env overrides.** Start the built image with all DB/NATS env vars supplied externally. Confirm the service connects — proves the empty defaults in the Dockerfile do not break real runs.
5. **Runtime with no env overrides.** Start the built image with no DB/NATS env vars. Confirm the service fails fast with a clear config error (not by silently connecting to `10.10.80.18`).
6. **Branch is gone.** After Step B9, `git ls-remote --heads origin feat-paysys` in `rule-executer` returns nothing.
7. **Generated rule roundtrip.** In Rule Studio, generate a new rule end-to-end (create -> deploy) after Track A. Confirm the generated rule folder was cloned from `dev` and boots cleanly.

### Data Migration Validation

No schema migration. Both tracks are code-only / workflow-only changes; no database, no persisted state.

### Manual / UAT Checks

| # | Scenario | Steps | Expected Result |
|---|---|---|---|
| 1 | Deploy workflow retargeted | Trigger `deploy.yml` in `rule-studio-example` on `main` after Track A | Clone step logs `-b dev`; deploy completes |
| 2 | UAT workflow retargeted | Trigger `deploy-to-uat.yml` on `main` after Track A | Clone step logs `-b dev`; rule install at line 104 succeeds |
| 3 | Track A applied to feat-paysys of studio-example | `git show origin/feat-paysys:.github/workflows/deploy.yml` in `rule-studio-example` | Line 41 reads `-b dev` |
| 4 | Docker build uses build secret | `docker build --secret id=GH_TOKEN,env=GH_TOKEN .` locally | Build succeeds; `docker history` shows no plaintext token |
| 5 | Runtime respects supplied env | Run image with real DB env vars | Service connects using supplied host, not `10.10.80.18` |
| 6 | Husky hook works | Stage a `.ts` file with a lint error and `git commit` | Commit rejected by `lint-staged` |
| 7 | Orphan file removed | `git ls-tree origin/dev -- src/controllers/rule.ts` after B5 | Empty (file absent) |
| 8 | Test harness removed | `git ls-tree origin/dev -- simple-rule2-test.js` after B6 | Empty (file absent) |
| 9 | Branch deleted | `git ls-remote --heads https://github.com/tazama-lf/rule-executer feat-paysys` after B9 | No matching ref |
| 10 | Template repo audit | Read `.env.sample` in `rule-studio-devtestops`, follow to `psl-copilot/rule-template` if it exists | Confirm which template repo Rule Studio actually clones; apply Track A there too if needed |

---

## Overall Impact of the Fix

| Area | Before | After |
|---|---|---|
| Rule Studio clone target | `rule-executer @ feat-paysys` (abandoned, 19 commits diverged) | `rule-executer @ dev` (maintained) |
| `feat-paysys` branch on `rule-executer` | Alive, receiving deploys, hosting stale deps | Deleted |
| Dockerfile DB/NATS defaults | Hardcoded `10.10.80.18:15432` / `10.10.80.18:14222` | Empty defaults; must be supplied by caller |
| Dockerfile `GH_TOKEN` handling | `ARG GH_TOKEN` (risk of baking into layer) | `--mount=type=secret,id=GH_TOKEN` (build-secret) |
| `frms-coe-lib` | `0.0.1-psl.0` (private psl build) | `8.2.0-rc.6` (public tazama-lf release) |
| `frms-coe-startup-lib` | `3.0.2-rc.5` | `3.1.0-rc.8` |
| `rule` dependency placeholder | Missing | Present (`npm:@tazama-lf/rule-901@4.0.0-rc.6`) |
| `execute.ts` logger noise | `[L##]`-prefixed debug logs on every request | Clean `Start`/`End` markers only |
| `execute.ts` type safety | `databaseManager as any` cast | Passed with its concrete type |
| `src/controllers/rule.ts` | Present but never imported | Deleted |
| `simple-rule2-test.js` | 124-line ad-hoc harness at repo root | Deleted |
| `.husky/pre-commit` | `npx lint - staged` (silently broken) | `npx lint-staged` (working) |
| `.npmrc @psl-copilot` scope | Only on `feat-paysys` | Present on `dev` (Option 3a) or removed everywhere (Option 3b) |

---

## Fix Summary

The root problem was that Rule Studio deployments were pulling the `rule-executer` codebase from an old private-fork branch (`feat-paysys`) that had drifted 19 commits away from what Tazama actually maintains on `dev`. Every rule generated by Rule Studio for the last several months has been built on top of a stale library, with hardcoded UAT database IPs baked into its Dockerfile, a broken pre-commit hook, orphaned dead code in its controllers, and a `GH_TOKEN` build-arg pattern that risks leaking the token into image layers. None of that was a Tazama design decision — it was drift, and it kept getting reinforced because the deployment workflows kept re-cloning from the same drifted branch.

Track A is the small, immediate fix: change two workflow files in `rule-studio-example` (both on `main`/`dev` and on that repo's own `feat-paysys`) so the clone URL targets `-b dev` instead of `-b feat-paysys`. From the moment Track A ships, every new rule generated by Rule Studio is built from the maintained codebase, and the abandoned branch stops mattering to future work.

Track B is the cleanup pass on `rule-executer` itself: reconcile the `feat-paysys` branch with `dev` — bump `frms-coe-lib` and `frms-coe-startup-lib` to their public releases, restore the build-secret pattern in the Dockerfile, empty out the hardcoded UAT DB and NATS values, remove the `[L##]` debug logs and the `databaseManager as any` cast from `execute.ts`, delete the orphaned `src/controllers/rule.ts` and the ad-hoc `simple-rule2-test.js`, fix the broken husky hook, and align whitespace in `src/config.ts`. Once that PR merges, delete the `feat-paysys` branch so no one can accidentally aim a workflow at it again.

The two tracks are independent — Track A can ship today, Track B can follow at whatever pace the team is comfortable reviewing dependency bumps and controller diffs. The only ordering rule is that the `feat-paysys` branch must not be deleted until Track A has landed and any external consumers (including a possible `psl-copilot/rule-template` fork) have been retargeted.
