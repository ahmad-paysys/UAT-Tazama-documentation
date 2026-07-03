# GitHub Issue & PR Templates — Issue #447

---

## GitHub Issue Body

### Problem

Rule executer deployments currently clone the `feat-paysys` branch of `tazama-lf/rule-executer` from the studio deployment workflows, but `feat-paysys` has diverged from `dev` (28 commits ahead, 35 commits behind) and is no longer maintained. As a result:

- New rules generated from the studio template deploy from a stale, Paysys-specific fork of the executer
- `dev` — the maintained branch — carries genuine improvements (updated `frms-coe-lib` and `frms-coe-startup-lib`, a working `rule` npm placeholder, a functioning husky pre-commit hook) that never reach production deployments
- `feat-paysys` carries local hacks that were useful during Paysys onboarding but should not be the long-term truth: hardcoded UAT IPs in the Dockerfile, disabled APM, ~20 numbered debug `loggerService.log` calls in `execute.ts`, a `databaseManager as any` cast, an orphaned `rule.ts` controller, and a 124-line standalone test script

The two branches need to be reconciled and the deploy workflows need to point at `dev`.

### Root Cause

The Paysys engagement forked a working branch (`feat-paysys`) to iterate quickly against a specific UAT environment. That branch was pinned into the studio-template deploy workflows and never rebased back onto `dev`. Meanwhile `dev` continued to evolve. Nothing enforces the two branches converge, so every new rule deployed via the studio pipeline ships the frozen fork.

### Impact

- Every rule created via `rule-studio-devtestops` bootstrap deploys from `feat-paysys` — carrying stale library versions
- Dockerfile hardcodes `10.10.80.18` and port `15432` — harmless inside the workflow (which overrides via `docker run -e`) but broken for anyone running the image outside the workflow
- APM is disabled on `feat-paysys` deployments
- `execute.ts` writes ~20 numbered debug lines (`[L01]`, `[L02]`, ...) per transaction, inflating logs and leaking flow markers into production
- `.husky/pre-commit` is broken on `feat-paysys` (`npx lint - staged` with stray whitespace) — commits skip lint entirely
- Two dead files (`src/controllers/rule.ts`, `simple-rule2-test.js`) confuse maintainers looking for the actual transaction handler

### Proposed Fix

Two independent tracks.

**Track A — Retarget deploy workflows to `dev` (rule-studio-example, possibly psl-copilot/rule-template)**

- What: change the `git clone -b feat-paysys` argument to `-b dev` in `deploy.yml` and `deploy-to-uat.yml`
- Files: `.github/workflows/deploy.yml` and `.github/workflows/deploy-to-uat.yml` on `tazama-lf/rule-studio-example` (both `main`/`dev` and — if still active — `feat-paysys`)
- Verification required: `rule-studio-devtestops`'s `.env.sample` points `GITHUB_TEMPLATE_OWNER=psl-copilot` / `GITHUB_TEMPLATE_REPO=rule-template`. If the production template is actually `psl-copilot/rule-template`, the same change must be applied there
- Migration: none
- Effort: ~30 minutes (change, review, merge) once the template ownership is confirmed

**Track B — Reconcile `feat-paysys` into `dev` in `tazama-lf/rule-executer`**

- What: bring the good parts of `feat-paysys` into `dev`, drop the bad, delete the branch
- Files:
  - `Dockerfile` — remove hardcoded `10.10.80.18` / port `15432`; set `APM_ACTIVE=true`; restore `SERVER_URL=0.0.0.0:4222`; set `RULE_NAME="901"` (not `"rule-901"`); restore BuildKit secret mount `--mount=type=secret,id=GH_TOKEN,env=GH_TOKEN`
  - `package.json` — `frms-coe-lib` `0.0.1-psl.0` → `8.2.0-rc.6`; `frms-coe-startup-lib` `3.0.2-rc.5` → `3.1.0-rc.8`; restore `"rule": "npm:@tazama-lf/rule-901@4.0.0-rc.6"`; bump version `3.0.0` → `4.0.0-rc.2`
  - `.npmrc` — decide fate of `@psl-copilot:registry=https://npm.pkg.github.com` scope (used by `deploy-to-uat.yml` which installs `@psl-copilot/$RULE_NAME`)
  - `src/controllers/execute.ts` — strip 20+ `[L##]`-prefixed `loggerService.log` calls; drop the `databaseManager as any` cast
  - `src/controllers/rule.ts` — **delete** (orphaned; nothing imports it — `execute.ts` uses `handleTransaction` from `'rule/lib'`)
  - `simple-rule2-test.js` — delete (124-line orphan script)
  - `.husky/pre-commit` — fix `npx lint - staged` → `npx lint-staged`
  - `src/config.ts` — whitespace-only realignment
- Migration: none (library bumps are `rc` version changes; behaviour compatibility verified in-repo)
- Sequencing: Track A can ship immediately. Track B lands on `dev`, then `feat-paysys` is deleted.
- Effort: ~1–2 days including review, CI, and a staging deployment smoke test

### Acceptance Criteria

**Track A**
- [ ] `deploy.yml` on `rule-studio-example` clones `rule-executer` from `dev`, not `feat-paysys`
- [ ] `deploy-to-uat.yml` on `rule-studio-example` clones `rule-executer` from `dev`
- [ ] If `rule-studio-example`'s `feat-paysys` branch is still active, the same change is applied there
- [ ] Ownership of the production template repo (`tazama-lf/rule-studio-example` vs `psl-copilot/rule-template`) is confirmed; the winner has both workflow files pointing at `dev`
- [ ] A fresh rule bootstrap end-to-end deploys from `dev`

**Track B**
- [ ] `Dockerfile` on `dev` has no hardcoded UAT IPs; `APM_ACTIVE=true`; `SERVER_URL=0.0.0.0:4222`; `RULE_NAME="901"`; BuildKit secret mount restored
- [ ] `package.json` on `dev` has the library versions above and version `4.0.0-rc.2`
- [ ] `.npmrc` decision recorded (kept or removed) with rationale in the PR body
- [ ] `execute.ts` on `dev` contains no `[L##]` debug log lines and no `as any` cast on `databaseManager`
- [ ] `src/controllers/rule.ts` and `simple-rule2-test.js` do not exist on `dev`
- [ ] `.husky/pre-commit` runs `npx lint-staged` successfully
- [ ] All unit tests pass; a rule built from a fresh bootstrap runs end-to-end in UAT
- [ ] `feat-paysys` branch deleted from `tazama-lf/rule-executer`

---

## PR Descriptions

### Track A PR

#### PR Title
Retarget rule-executer clone to dev in deploy workflows

#### PR Body

**Summary**
- Changes `git clone https://github.com/tazama-lf/rule-executer -b feat-paysys` to `-b dev` in `.github/workflows/deploy.yml`
- Same change in `.github/workflows/deploy-to-uat.yml`
- Applies to `main`/`dev` on `tazama-lf/rule-studio-example` (and `feat-paysys` if still active)
- If `psl-copilot/rule-template` is the actual production template used by `rule-studio-devtestops`, apply the identical change there
- Note: `deploy-to-uat.yml` remains Paysys-specific (hardcodes `@psl-copilot` at line 104) — this PR does not attempt to unify it

**Test Plan**
- [ ] Trigger a rule bootstrap through `rule-studio-devtestops` against the updated template
- [ ] Confirm the deploy log shows the clone succeeding from `dev`
- [ ] Confirm the resulting image starts and processes a transaction end-to-end in UAT
- [ ] Confirm no reference to `feat-paysys` remains in either workflow file

**Related Issue**
tazama-lf/rule-executer#447

---

### Track B PR

#### PR Title
Reconcile feat-paysys into dev; retire feat-paysys

#### PR Body

**Summary**
- Dockerfile: remove hardcoded UAT IPs / ports; set `APM_ACTIVE=true`; restore `SERVER_URL=0.0.0.0:4222`; `RULE_NAME="901"`; restore BuildKit `GH_TOKEN` secret mount
- `package.json`: `frms-coe-lib` → `8.2.0-rc.6`, `frms-coe-startup-lib` → `3.1.0-rc.8`, restore `rule` npm placeholder, bump version to `4.0.0-rc.2`
- `.npmrc`: `@psl-copilot` scope decision recorded (kept if `deploy-to-uat.yml` pipeline stays; removed if retired)
- `execute.ts`: strip ~20 numbered debug log lines, drop `databaseManager as any` cast
- Delete orphan files `src/controllers/rule.ts` and `simple-rule2-test.js`
- Fix `.husky/pre-commit` (`npx lint - staged` → `npx lint-staged`)
- Whitespace realignment in `src/config.ts`
- Delete `feat-paysys` branch after merge

**Test Plan**
- [ ] `npm ci && npm test` passes locally
- [ ] `npm run build` succeeds
- [ ] `npx lint-staged` invoked by the husky pre-commit hook completes without error
- [ ] Image built from the new Dockerfile boots without requiring the workflow-level `-e` overrides
- [ ] APM traces appear in the APM server after boot
- [ ] A rule bootstrapped from the updated template processes a transaction end-to-end in UAT
- [ ] Log output contains no `[L##]`-prefixed lines
- [ ] `grep -r "feat-paysys" .` in the repo returns nothing

**Related Issue**
tazama-lf/rule-executer#447

**Migration**
- No schema migration.
- Library version bumps are `rc` releases within the same major line — no API surface changes expected. Reviewer should still `npm ci` and confirm the build.
- After merge, delete `feat-paysys` via `git push origin --delete feat-paysys`.

**Sequencing Note**
- Track A must be merged and deployed first, so that new rule bootstraps pull from `dev`. Track B can then land on `dev` without any window where deployments still clone `feat-paysys`.
- If `psl-copilot/rule-template` is confirmed as the production template, the corresponding Track A change there must also be merged before this PR ships.
