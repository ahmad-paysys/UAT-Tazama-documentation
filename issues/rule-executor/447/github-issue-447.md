# GitHub Issue & PR Templates — Issue #447

---

## GitHub Issue Body

### Problem

Every rule deployed through Rule Studio is built by cloning the `feat-paysys` branch of `tazama-lf/rule-executer`. That branch is a diverged, ad-hoc fork: as of 2026-07-03 it is **28 commits ahead and 35 commits behind `dev`**, and the clone target is hardcoded in both deploy workflows in `tazama-lf/rule-studio-example`. This means every Studio-generated rule ships executer code that misses upstream fixes, carries unreviewed local changes (hardcoded UAT IPs, disabled APM, debug `console.log` output), and is pinned to a pre-release Paysys-internal library version not available from the standard `tazama-lf` npm registry.

### Root Cause

Both `.github/workflows/deploy.yml:39` and `.github/workflows/deploy-to-uat.yml:36` in `tazama-lf/rule-studio-example` contain `git clone https://github.com/tazama-lf/rule-executer -b feat-paysys`. `feat-paysys` was created as a working branch for internal UAT customisation and was never reconciled with `dev` or placed on a promotion path. Because the clone target was never updated, it became the de facto release branch for the entire Rule Studio pipeline without any review gate.

### Impact

- All Studio-deployed rules miss 35 `dev` commits, including security fixes from PRs #443 (`frms-coe-lib` advisory sweep), #445 (reason field fix), and #446 (dispatch token fix).
- `feat-paysys/Dockerfile` hardcodes UAT IPs (`10.10.80.18:15432`) for all database connections, overriding runtime env vars injected by `docker run` in any non-UAT environment.
- APM is globally disabled (`APM_ACTIVE=false`) for all Studio rules, silently killing observability.
- `src/controllers/rule.ts` (present only on `feat-paysys`) contains raw `console.log("hello bhai…")` output visible in production container logs.
- `frms-coe-lib` is pinned to `0.0.1-psl.0`, a Paysys-internal pre-release unavailable from the `tazama-lf` GitHub Packages registry; standard builds relying on this will fail outside Paysys infrastructure.
- The `sed` step in `deploy.yml` that patches the `rule` placeholder in `package.json` is a silent no-op because `feat-paysys/package.json` carries no `rule` key.

### Proposed Fix

**Track A (immediate, low-risk):** Replace `-b feat-paysys` with `-b dev` in both workflow files in `tazama-lf/rule-studio-example`. Two lines changed, no application code. All future builds immediately use the maintained branch.

- Files: `.github/workflows/deploy.yml`, `.github/workflows/deploy-to-uat.yml`
- Migration required: No
- Estimated effort: 30 minutes

**Track B (full reconciliation):** Audit and reconcile the 28 `feat-paysys`-only commits into `dev` via a proper PR: remove hardcoded IPs and restore BuildKit secret handling in `Dockerfile`; align library versions; strip debug log statements from `execute.ts`; decide whether `rule.ts` `BaseMessage` handling is needed on `dev`; delete `feat-paysys`.

- Files: `Dockerfile`, `package.json`, `.npmrc`, `src/controllers/execute.ts`, `src/controllers/rule.ts` (decision), `simple-rule2-test.js` (delete)
- Migration required: No
- Estimated effort: 1–2 days

### Acceptance Criteria

- [ ] `deploy.yml:39` and `deploy-to-uat.yml:36` clone branch `dev`, not `feat-paysys`
- [ ] A triggered workflow log shows a `dev` branch SHA in the clone step
- [ ] Built container carries `frms-coe-lib@8.2.0-rc.6` and has `APM_ACTIVE=true`
- [ ] No hardcoded IPs in Docker image ENV defaults
- [ ] No `[L##]`-prefixed debug logs or `console.log("hello bhai")` in running containers
- [ ] `feat-paysys` branch deleted from remote after Track B merges

---

## PR Descriptions

### Track A PR

#### PR Title
`fix(ci): clone rule-executer from dev instead of feat-paysys`

#### PR Body

**Summary:**
- Replaces the hardcoded `-b feat-paysys` clone target in `deploy.yml` (line 39) with `-b dev`
- Replaces the same in `deploy-to-uat.yml` (line 36) with `-b dev`
- Ensures all future Rule Studio rule deployments build against the maintained upstream branch
- No application code changes; the only effect is which branch is cloned at build time

**Test Plan:**
- [ ] Trigger `deploy.yml` via `workflow_dispatch` in any Rule Studio rule repo
- [ ] Confirm workflow log step "Clone Rule Executer Repo" shows a `dev` SHA, not `feat-paysys`
- [ ] Confirm built container has `frms-coe-lib@8.2.0-rc.6` (not `0.0.1-psl.0`)
- [ ] Confirm `APM_ACTIVE=true` in the running container env
- [ ] Submit a test transaction; confirm a valid `RuleResult` is produced

**Related Issue:** https://github.com/tazama-lf/rule-executer/issues/447

---

### Track B PR

#### PR Title
`fix: reconcile feat-paysys divergence into dev and remove debug artifacts`

#### PR Body

**Summary:**
- Removes hardcoded UAT IPs (`10.10.80.18`) and ports from `Dockerfile` ENV defaults; restores empty-string defaults so runtime env vars take effect
- Restores BuildKit secret mount (`--mount=type=secret,id=GH_TOKEN`) replacing plain `ARG GH_TOKEN` in both Dockerfile build stages
- Re-enables APM (`APM_ACTIVE=true`)
- Aligns `frms-coe-lib` to `8.2.0-rc.6` and `frms-coe-startup-lib` to `3.1.0-rc.8`
- Restores `rule` placeholder dependency in `package.json` so the `deploy.yml` `sed` step is functional
- Removes all `[L##]`-prefixed debug `loggerService.log` calls from `execute.ts` and removes `databaseManager as any` cast
- Deletes `simple-rule2-test.js` (UAT-only debug file)
- Resolves decision on `src/controllers/rule.ts` (see PR description body for outcome)

**Test Plan:**
- [ ] `docker run --rm <image> node -p "require('/home/app/node_modules/@tazama-lf/frms-coe-lib/package.json').version"` → `8.2.0-rc.6`
- [ ] `docker inspect <container>` env shows `APM_ACTIVE=true`
- [ ] `docker inspect <container>` env shows `RAW_HISTORY_DATABASE_HOST=` (empty, not hardcoded)
- [ ] Container logs contain no `[L##]`-prefixed lines or `hello bhai` output during a test transaction
- [ ] End-to-end test: submit transaction → rule container → Typology Processor receives valid `RuleResult`
- [ ] `gh api repos/tazama-lf/rule-executer/branches/feat-paysys` → 404 after branch deletion

**Sequencing Note:** This PR must merge after Track A (the workflow clone-target fix). Merging Track B into `dev` before Track A means deployed rules still build from `feat-paysys` even though `dev` is corrected.

**Related Issue:** https://github.com/tazama-lf/rule-executer/issues/447

**Migration:** None. No schema changes. All changes are source code and configuration.
