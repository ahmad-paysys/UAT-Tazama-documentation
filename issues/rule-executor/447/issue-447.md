# Issue #447 — Rule Studio pipeline hardcoded to feat-paysys branch and Paysys artifacts

**Repository:** tazama-lf/rule-executer
**Issue:** [Rule Studio pipeline hardcoded to feat-paysys branch and Paysys artifacts](https://github.com/tazama-lf/rule-executer/issues/447)
**Author:** Justus-at-Tazama (Justus Ortlepp)
**State:** Open
**Report Date:** 2026-07-03

---

## Executive Summary

The Rule Studio deploy pipeline in `tazama-lf/rule-studio-example` clones `rule-executer` at a hardcoded `feat-paysys` branch and pins Paysys-specific artifacts (`@psl-copilot` npm org, UAT IP addresses `10.10.80.18` / `10.10.80.37`, port `15432`, private `frms-coe-lib 0.0.1-psl.0`). The `feat-paysys` branch of `rule-executer` has diverged from `dev` (28 commits ahead / 35 behind as of 2026-07-03), carries a private registry dependency the tazama-lf GitHub runners cannot resolve, and contains dead code (`src/controllers/rule.ts`) and a broken pre-commit hook. As a result, every new rule bootstrapped through Rule Studio inherits a Paysys-only runtime; the pipeline is not portable to any other operator.

Track A is a workflow-only redirect: change the clone target from `feat-paysys` to `dev` in the workflow files that pull rule-executer, and align the rule-executer image so it can be built from `dev`. Track B is the full re-alignment: fold the surviving parts of `feat-paysys` into `dev` (or discard them), parametrise all environment-specific values (npm org, DB host, NATS URL, Redis host, port, rule version), fix the Dockerfile secret handling, and delete the orphaned/dead files. Track B also requires confirming which template repository the bootstrap service actually clones from — the `.env.sample` in `rule-studio-devtestops` points at `psl-copilot/rule-template`, not `tazama-lf/rule-studio-example`, and if that is the live configuration then fixing `rule-studio-example` alone will not propagate to new rule repos.

**Open question — needs verification:** the actual production template repo (see `rule-studio-devtestops/.env.sample` — `GITHUB_TEMPLATE_OWNER=psl-copilot`, `GITHUB_TEMPLATE_REPO=rule-template`). Before Track A ships we must confirm whether `tazama-lf/rule-studio-example` is authoritative or whether the fix must also be applied to `psl-copilot/rule-template`.

---

## How It Works Today — Confirmed in Code

### 1. The workflow clones a Paysys branch by name

`repos/rule-studio-example/.github/workflows/deploy.yml` on `main`/`dev` — line 39:

```yaml
git clone https://github.com/tazama-lf/rule-executer -b feat-paysys
```

The sibling workflow `deploy-to-uat.yml` line 36 does the same:

```yaml
git clone https://github.com/tazama-lf/rule-executer -b feat-paysys
```

Both workflows are the only entry points that materialise a runnable rule-executer image for a new rule. No branch is parametric.

### 2. The workflow rewrites `package.json` — the `sed` is dead code

`deploy.yml` line 66 (regex `sed`), lines 108-111 (delete + reinstall):

```yaml
sed -E -i 's|"rule": "npm:@[^/]+/rule-placeholder@latest"|"rule": "npm:@'"$ORG"'/'"$RULE_NAME"'@latest"|g' package.json
...
npm pkg delete dependencies.rule
npm install --save-exact rule@npm:@$ORG/$RULE_NAME@latest --ignore-scripts
```

The `sed` regex matches any org, but the workflow then always deletes the `rule` dep and reinstalls it. The `sed` line has no functional effect — it is dead code, not a broken step.

`deploy-to-uat.yml` line 63 hardcodes the org in the regex, and line 104 hardcodes `@psl-copilot` in the install:

```yaml
sed -i 's|"rule": "npm:@psl-copilot/rule-placeholder@latest"|"rule": "npm:@psl-copilot/'"$RULE_NAME"'@latest"|g' package.json
...
npm install --save-exact rule@npm:@psl-copilot/$RULE_NAME@latest
```

`deploy-to-uat.yml` is Paysys-specific by design: `@psl-copilot` is baked in, so retargeting it at `dev` alone does not make it operator-agnostic.

### 3. The workflow injects UAT IP addresses at `docker run`

`deploy.yml` lines 135-168:

```yaml
docker run -d \
  -e RAW_HISTORY_DATABASE_HOST=10.10.80.18 \
  -e SERVER_URL=10.10.80.18:14222 \
  -e REDIS_HOST="10.10.80.18" \
  -e RULE_VERSION=1.0.0 \
  -e RULE_NAME=$RULE_ID \
  ...
```

`deploy-to-uat.yml` lines 128-157 uses a different set: `10.10.80.37`, port `5432`, no Redis. Two workflows, two environments, both hardcoded.

The runtime `-e` flags override the Dockerfile ENV defaults, so the Dockerfile hardcoded IPs (see below) are cosmetic within the Rule Studio pipeline. The portability problem lives in the workflow files themselves.

### 4. The rule-executer `feat-paysys` branch carries Paysys internals

`repos/rule-executer` on `feat-paysys`:

- `package.json` (HEAD): `"frms-coe-lib": "0.0.1-psl.0"` — private Paysys registry; not resolvable from tazama-lf runners.
- `package.json` (HEAD): `"frms-coe-startup-lib": "3.0.2-rc.5"` — public but pinned to an old RC.
- `package.json` (HEAD): version `3.0.0`; no `rule` dep (installed later by the workflow). `dev` package.json is version `4.0.0-rc.2` with `frms-coe-lib 8.2.0-rc.6`, `frms-coe-startup-lib 3.1.0-rc.8`, and `"rule": "npm:@tazama-lf/rule-901@4.0.0-rc.6"`.
- `.npmrc` adds `@psl-copilot:registry=https://npm.pkg.github.com` (not present on `dev`). This scope is used by `deploy-to-uat.yml` line 104.
- `Dockerfile`: hardcoded `RAW_HISTORY_DATABASE_HOST=10.10.80.18`, port `15432`, `APM_ACTIVE=false`, `SERVER_URL=10.10.80.18:14222`, `RULE_NAME="rule-901"` (dev: `"901"`).
- `Dockerfile`: uses plain `ARG GH_TOKEN` for private registry auth. `dev` uses `RUN --mount=type=secret,id=GH_TOKEN,env=GH_TOKEN npm ci`. The `feat-paysys` form leaves the token in the image layer.
- `src/controllers/execute.ts`: 20+ `[L##]`-prefixed `loggerService.log` calls, a `databaseManager as any` cast, and `import { handleTransaction } from 'rule/lib'` (i.e. from the installed `rule` npm package, not from local `./rule`).
- `src/controllers/rule.ts`: new file, defines a local `handleTransaction` with `BaseMessage` handling and five `console.log("hello bhai…")` statements. **Not imported anywhere.** Confirmed dead code on the branch.
- `.husky/pre-commit`: contains `npx lint - staged` (with spaces). Will error on every commit. `dev` has `npx lint-staged`.
- `simple-rule2-test.js`: 124-line standalone script at repo root, not wired into `jest`.
- `src/config.ts`: whitespace-only reformat vs `dev` — semantically identical.

### 5. Bootstrap service points at a different template repo

`repos/rule-studio-devtestops/.env.sample`:

```
GITHUB_TEMPLATE_OWNER=psl-copilot
GITHUB_TEMPLATE_REPO=rule-template
```

If this is the actual production configuration, the bootstrap service clones `psl-copilot/rule-template` — not `tazama-lf/rule-studio-example` — when creating new rule repos. Fixing `rule-studio-example` alone would then have no effect on newly bootstrapped rules. This requires verification with the operations team.

### 6. `rule-studio-example` also has a `feat-paysys` branch

The template repo itself has a `feat-paysys` branch, 19 commits ahead of `dev`, adding Docker Hub push logic. If any downstream consumer clones this template on `feat-paysys`, the fix must be applied on that branch too.

---

## Root Cause Analysis

**Root cause:** the deploy pipeline in `rule-studio-example` and the rule-executer image expected by that pipeline were both forked into a Paysys-specific configuration (`feat-paysys` branch, `@psl-copilot` npm org, UAT IPs, private `frms-coe-lib`) and then wired together by hardcoded string references. Nothing in the current `dev` branches of either repo is exercised by the deploy pipeline, so `dev` has silently rotted while `feat-paysys` is the de-facto production branch.

From that one root cause:

- Every new rule bootstrapped via Rule Studio ships with Paysys IPs and `@psl-copilot` scope.
- `dev`-branch improvements to rule-executer (secret-mounted `GH_TOKEN`, updated `frms-coe-lib`, `rule-901` pinning) never reach production.
- Non-Paysys operators cannot use Rule Studio without editing YAML files by hand.
- The `feat-paysys` branch accumulates unmaintained code (`rule.ts` dead file, broken husky hook, `simple-rule2-test.js`) with no path back to `dev`.

---

## Blast Radius — Full File Inventory

### Files that must change

| File | What is affected | Line numbers |
|---|---|---|
| `repos/rule-studio-example/.github/workflows/deploy.yml` | Hardcoded `-b feat-paysys`; hardcoded UAT IPs, ports, NATS/Redis hosts; dead `sed` line | 39, 66, 135-168 |
| `repos/rule-studio-example/.github/workflows/deploy-to-uat.yml` | Hardcoded `-b feat-paysys`; hardcoded `@psl-copilot` org in sed and install; hardcoded IPs | 36, 63, 104, 128-157 |
| `repos/rule-executer/package.json` (feat-paysys) | `frms-coe-lib 0.0.1-psl.0` private dep; stale `frms-coe-startup-lib`; version drift | package root |
| `repos/rule-executer/.npmrc` (feat-paysys) | `@psl-copilot` scope registry — Paysys-only | full file |
| `repos/rule-executer/Dockerfile` (feat-paysys) | Hardcoded DB host/port, NATS URL, RULE_NAME; plain `ARG GH_TOKEN` leaking secret into layer | ENV block; GH_TOKEN ARG |
| `repos/rule-executer/src/controllers/execute.ts` (feat-paysys) | `[L##]` log spam; `as any` cast; import path drift | ~20 log call sites; DB manager cast |
| `repos/rule-executer/src/controllers/rule.ts` (feat-paysys) | Orphaned dead file; `console.log("hello bhai…")` × 5 | full file |
| `repos/rule-executer/.husky/pre-commit` (feat-paysys) | `npx lint - staged` — broken command | single line |
| `repos/rule-executer/simple-rule2-test.js` (feat-paysys) | 124-line ad-hoc test script, not part of jest suite | full file |

### Files indirectly affected

| File | Why it is affected | Line numbers |
|---|---|---|
| `repos/rule-studio-devtestops/.env.sample` | Points at `psl-copilot/rule-template` — determines whether the fix propagates to new rules | `GITHUB_TEMPLATE_OWNER`, `GITHUB_TEMPLATE_REPO` |
| `repos/rule-studio-example` `feat-paysys` branch | 19 commits ahead of `dev` with Docker Hub push logic; may need parallel fix | full branch |
| `repos/rule-executer/src/config.ts` (feat-paysys) | Whitespace reformat vs `dev` — will show up in any merge diff | full file |
| Any bootstrapped rule repo already created from the current template | Inherits Paysys IPs and `@psl-copilot` scope in its own `deploy.yml` copy | copies of `deploy.yml` |

---

## Side-Effect Map

| Consequence if unfixed | Trigger |
|---|---|
| New rules cannot be deployed by non-Paysys operators | `@psl-copilot` hardcoded in `deploy-to-uat.yml:104` |
| `dev` branch of rule-executer accumulates untested changes never exercised in production | Workflow pins to `feat-paysys` |
| `GH_TOKEN` remains embedded in image layers | `Dockerfile` `ARG GH_TOKEN` on `feat-paysys` |
| Pre-commit hook silently fails on every developer clone | `.husky/pre-commit` typo |
| CI is unable to install `frms-coe-lib 0.0.1-psl.0` on a public runner | Private registry pin in `feat-paysys` package.json |
| Bootstrapped rules ship with dead file `rule.ts` and stray `console.log` | `feat-paysys` HEAD carries them |
| Any UAT IP change (`10.10.80.18`, `10.10.80.37`) requires editing YAML in every consumer | Hardcoded workflow env |
| Bootstrap template drift (`psl-copilot/rule-template` vs `tazama-lf/rule-studio-example`) leaves two forks unreconciled | `.env.sample` in devtestops points at psl-copilot |

---

## Effort Assessment

### Track A — Redirect workflow to `dev` and unblock non-Paysys builds

**What changes:** workflow files only, plus the minimum on rule-executer `dev` needed for the workflow to build a working image.

| Step | File | Effort |
|---|---|---|
| Change `-b feat-paysys` → `-b dev` | `rule-studio-example/.github/workflows/deploy.yml:39` | 5 min |
| Change `-b feat-paysys` → `-b dev` | `rule-studio-example/.github/workflows/deploy-to-uat.yml:36` | 5 min |
| Remove dead `sed` line | `deploy.yml:66` | 5 min |
| Verify rule-executer `dev` `Dockerfile` builds from the redirected workflow | `rule-executer/Dockerfile` (dev) | 30 min |
| Smoke-test one rule deployment end-to-end | pipeline | 1-2 h |
| Confirm which template repo bootstrap service actually uses | `rule-studio-devtestops/.env.sample` + ops team | 30 min (external) |

- Schema migration: **No**
- Frontend changes: **No**
- Total: ~3-4 h engineering + verification with ops.

**Note:** `deploy-to-uat.yml` retargeting alone will not remove the `@psl-copilot` hardcoding on line 104. Track A leaves that intact — retargeting to `dev` is only meaningful for `deploy.yml`. For `deploy-to-uat.yml`, Track A is "point at `dev` and accept it still installs from `@psl-copilot`" until Track B parametrises it.

### Track B — Full re-alignment

**What changes:** merge or discard `feat-paysys`, parametrise all env-specific values, fix Dockerfile secret handling, delete dead code, reconcile template repo drift.

| Step | File | Effort |
|---|---|---|
| Decide fate of `feat-paysys` (merge surviving pieces to `dev`, or archive branch) | `rule-executer` | 2-4 h review + team decision |
| Delete `src/controllers/rule.ts` (orphaned) | `rule-executer/src/controllers/rule.ts` | 15 min |
| Delete `simple-rule2-test.js` or convert to jest test | `rule-executer/simple-rule2-test.js` | 15 min - 2 h |
| Fix `.husky/pre-commit` typo | `rule-executer/.husky/pre-commit` | 5 min |
| Remove `[L##]` log spam and `as any` cast | `rule-executer/src/controllers/execute.ts` | 1 h |
| Bump `frms-coe-lib` to public `dev` version; remove `@psl-copilot` scope from `.npmrc` | `rule-executer/package.json`, `.npmrc` | 30 min + CI verify |
| Convert Dockerfile `ARG GH_TOKEN` → `RUN --mount=type=secret,id=GH_TOKEN` | `rule-executer/Dockerfile` | 30 min |
| Parametrise all `-e` values in `deploy.yml` via GitHub Actions inputs / repo variables | `rule-studio-example/.github/workflows/deploy.yml:135-168` | 2 h |
| Parametrise `@psl-copilot` → `${{ inputs.npm_org }}` in `deploy-to-uat.yml` | `rule-studio-example/.github/workflows/deploy-to-uat.yml:63,104,128-157` | 2 h |
| Reconcile `rule-studio-example` `feat-paysys` branch (merge or archive) | `rule-studio-example` | 2-4 h |
| Reconcile template repo drift with `psl-copilot/rule-template` (or point `.env.sample` at tazama-lf) | `rule-studio-devtestops/.env.sample` | 1-2 h + ops |
| Regression test one rule per operator (tazama-lf + Paysys) | pipeline | 4 h |

- Schema migration: **No** (this is CI/CD and container config, not data)
- Frontend changes: **No**
- Total: ~2-3 engineering days plus coordination with the Paysys operator to preserve their deployment path.

### Recommended Sequencing

1. Verify which template repo the bootstrap service actually clones (`rule-studio-devtestops/.env.sample`). This gates whether the fix goes into `tazama-lf/rule-studio-example`, `psl-copilot/rule-template`, or both.
2. Ship Track A on `rule-studio-example` `deploy.yml` (branch redirect + dead `sed` removal). This unblocks tazama-lf-flavoured deploys immediately.
3. In parallel, on rule-executer `dev`: delete dead files, fix husky hook, clean logging. These are safe standalone commits.
4. Ship Track B parametrisation (workflow inputs, Dockerfile secret mount, `.npmrc` cleanup) as a coordinated PR with the Paysys operator in the loop.
5. Only after Track B lands, decide whether to delete the `feat-paysys` branches on both repos or keep them as operator overlays.

---

## Acceptance Criteria

### Track A

- [ ] `rule-studio-example/.github/workflows/deploy.yml` line 39 reads `-b dev` (not `-b feat-paysys`).
- [ ] `rule-studio-example/.github/workflows/deploy-to-uat.yml` line 36 reads `-b dev`.
- [ ] Dead `sed` line at `deploy.yml:66` removed.
- [ ] One rule end-to-end deployment on tazama-lf infrastructure succeeds from the redirected workflow.
- [ ] Written confirmation captured (in this issue) of which template repo the bootstrap service actually uses.
- [ ] `deploy-to-uat.yml`'s continued reliance on `@psl-copilot` explicitly documented as deferred to Track B.

### Track B

- [ ] `frms-coe-lib` on rule-executer `dev` resolves from a public registry (no `0.0.1-psl.0`).
- [ ] `.npmrc` has no `@psl-copilot` scope (or scope is opt-in via env var).
- [ ] `Dockerfile` uses `RUN --mount=type=secret,id=GH_TOKEN` for private registry auth; no `ARG GH_TOKEN` leaking into layers.
- [ ] `src/controllers/rule.ts` deleted (or merged and imported).
- [ ] `simple-rule2-test.js` deleted or promoted to a jest test.
- [ ] `.husky/pre-commit` runs `npx lint-staged` successfully.
- [ ] `src/controllers/execute.ts` has no `[L##]` debug log prefixes and no `as any` casts.
- [ ] `deploy.yml` docker run values (`RAW_HISTORY_DATABASE_HOST`, `SERVER_URL`, `REDIS_HOST`, `RULE_VERSION`) sourced from workflow inputs or repo variables.
- [ ] `deploy-to-uat.yml` `@psl-copilot` references sourced from `inputs.npm_org` (default preserved for Paysys).
- [ ] `rule-studio-example` `feat-paysys` branch either merged to `dev` or archived with a documented reason.
- [ ] Template repo drift (`psl-copilot/rule-template` vs `tazama-lf/rule-studio-example`) resolved with ops team.
- [ ] Both tazama-lf and Paysys deployments succeed against the parametrised workflow.

---
