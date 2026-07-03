# Impact Study — Issue #447: Rule Studio pipeline hardcoded to feat-paysys branch and Paysys artifacts

**Issue:** [#447](https://github.com/tazama-lf/rule-executer/issues/447)
**Study Date:** 2026-07-03
**Author:** Ahmad Khalid
**Repository:** tazama-lf/rule-executer

---

## Summary

The Rule Studio deploy workflows in `tazama-lf/rule-studio-example` clone `rule-executer` at a hardcoded `feat-paysys` branch and inject Paysys-specific values (`@psl-copilot` npm org, UAT IPs `10.10.80.18` / `10.10.80.37`, port `15432`, private `frms-coe-lib 0.0.1-psl.0`). The `feat-paysys` branch has diverged from `dev` (28 ahead / 35 behind at 2026-07-03) and carries a private registry dep, a broken husky hook, and orphaned code that never reaches production quality. Track A redirects the workflow clones from `feat-paysys` to `dev` and removes one dead line, unblocking tazama-lf-flavoured builds. Track B is the full re-alignment: parametrise the workflows, fix Dockerfile secret handling, remove Paysys-only artifacts, and reconcile the bootstrap-service template drift (`psl-copilot/rule-template` vs `tazama-lf/rule-studio-example`).

---

## Confirmed Root Cause

- **Hardcoded branch in `deploy.yml`:** `repos/rule-studio-example/.github/workflows/deploy.yml:39` — `git clone https://github.com/tazama-lf/rule-executer -b feat-paysys`. Confirmed.
- **Hardcoded branch in `deploy-to-uat.yml`:** same repo, `deploy-to-uat.yml:36`. Confirmed.
- **`sed` at `deploy.yml:66` is dead code, not a functional problem:** the workflow follows at lines 108-111 with `npm pkg delete dependencies.rule` + `npm install --save-exact rule@npm:@$ORG/$RULE_NAME@latest --ignore-scripts`, which always installs the rule regardless of whether the earlier `sed` matched. Correcting an earlier reading of this issue as a "silent no-op that breaks the pipeline" — it does not break the pipeline, it is simply a redundant line.
- **`deploy-to-uat.yml` is Paysys-specific by design:** line 63 hardcodes `@psl-copilot` in the sed pattern; line 104 hardcodes `@psl-copilot` in the `npm install`. Retargeting the branch to `dev` alone does not make it operator-agnostic — those org references remain.
- **Dockerfile ENV hardcoding is cosmetic within the Rule Studio pipeline:** `deploy.yml:135-168` passes `-e RAW_HISTORY_DATABASE_HOST=10.10.80.18`, `-e SERVER_URL=10.10.80.18:14222`, `-e REDIS_HOST="10.10.80.18"` at `docker run`, overriding the image defaults. The Dockerfile ENVs matter when the image is run outside the workflow (e.g. locally with `docker run <image>` and no `-e`). The larger portability issue is that the workflow itself hardcodes UAT IPs.
- **`frms-coe-startup-lib` on `feat-paysys` is `3.0.2-rc.5`, not `0.0.1-psl.0`:** correcting earlier documentation. Only `frms-coe-lib` is on the private `0.0.1-psl.0` pin.
- **`src/controllers/rule.ts` is orphaned dead code:** `execute.ts` imports `handleTransaction` from `'rule/lib'` (the installed npm package), never from local `./rule`. Confirmed by grep. Contains five `console.log("hello bhai…")` statements. Default recommendation is deletion.
- **`.husky/pre-commit` runs `npx lint - staged`** (with spaces) on `feat-paysys` — will error on every commit. `dev` correctly has `npx lint-staged`.
- **`.npmrc` on `feat-paysys` adds `@psl-copilot:registry=https://npm.pkg.github.com`** — required by `deploy-to-uat.yml:104` when installing `rule@npm:@psl-copilot/$RULE_NAME@latest`. Not present on `dev`.
- **Dockerfile secret handling on `feat-paysys` uses `ARG GH_TOKEN`,** which leaves the token in image layers. `dev` uses `RUN --mount=type=secret,id=GH_TOKEN,env=GH_TOKEN npm ci`.
- **Bootstrap service template drift — needs verification:** `repos/rule-studio-devtestops/.env.sample` sets `GITHUB_TEMPLATE_OWNER=psl-copilot` and `GITHUB_TEMPLATE_REPO=rule-template`. If this is the live configuration, the bootstrap service clones `psl-copilot/rule-template`, not `tazama-lf/rule-studio-example` — fixing rule-studio-example alone would not reach new rule repos.
- **`rule-studio-example` also has a `feat-paysys` branch,** 19 commits ahead of `dev`, with Docker Hub push logic. If any consumer clones from it, the fix must be applied there too.

---

## Track A — Redirect workflow clones to `dev`

### What Changes

Three workflow edits in `rule-studio-example`:

1. `deploy.yml:39` — `-b feat-paysys` → `-b dev`.
2. `deploy-to-uat.yml:36` — `-b feat-paysys` → `-b dev`.
3. `deploy.yml:66` — delete the dead `sed` line.

Total: three lines changed across two files. No code changes in `rule-executer`.

### Impact

| Attribute | Value |
|---|---|
| Files changed | 2 (deploy.yml, deploy-to-uat.yml) |
| Schema migration required | No |
| Frontend changes required | No |
| Downtime required | No |
| Risk of regression | Medium — `dev` branch of rule-executer has never been built through this workflow |
| Reversibility | Trivial (git revert of workflow commit) |

**What this fixes immediately:**

- New tazama-lf rule deploys via `deploy.yml` will build against a maintained branch (`dev`) with public registry deps and secret-mounted `GH_TOKEN`.
- Removes the ambient confusion of a dead `sed` line in `deploy.yml`.
- Stops silently re-cloning a diverged branch every deploy.

**What this does not fix:**

- `deploy-to-uat.yml` still hardcodes `@psl-copilot` on lines 63 and 104 — Paysys deploys remain Paysys-only.
- All UAT IPs still hardcoded in `-e` flags on both workflows.
- `feat-paysys` branch on `rule-executer` remains — its dead files, broken husky hook, and `[L##]` log spam are untouched.
- Bootstrap service may still be pointed at `psl-copilot/rule-template` (needs verification).
- `rule-studio-example` `feat-paysys` branch is untouched.

Track A is safe to ship in isolation.

---

## Track B — Full re-alignment

### What Changes

Merge or discard `feat-paysys` on `rule-executer`; parametrise all environment-specific values in both workflows; convert Dockerfile secret handling; delete dead files and fix the husky hook; reconcile the bootstrap-service template repo drift.

### Schema Impact

No database or GraphQL schema changes. This is CI/CD and container configuration work.

| Change | Migration required | Risk |
|---|---|---|
| None | N/A | N/A |

### Backend Code Impact

**Core rewrites:**

| File | Change |
|---|---|
| `repos/rule-executer/package.json` | Bump `frms-coe-lib` from `0.0.1-psl.0` → public `dev` version (currently `8.2.0-rc.6`); bump `frms-coe-startup-lib` to `3.1.0-rc.8` |
| `repos/rule-executer/.npmrc` | Remove `@psl-copilot` scope (or make opt-in via env) |
| `repos/rule-executer/Dockerfile` | `ARG GH_TOKEN` → `RUN --mount=type=secret,id=GH_TOKEN,env=GH_TOKEN npm ci`; drop hardcoded IP/port ENVs in favour of workflow-injected values |
| `repos/rule-executer/src/controllers/execute.ts` | Remove ~20 `[L##]` log prefixes; remove `databaseManager as any` cast |
| `repos/rule-studio-example/.github/workflows/deploy.yml` | Parametrise `RAW_HISTORY_DATABASE_HOST`, `SERVER_URL`, `REDIS_HOST`, `RULE_VERSION` via `inputs`/`vars` |
| `repos/rule-studio-example/.github/workflows/deploy-to-uat.yml` | Replace hardcoded `@psl-copilot` with `${{ inputs.npm_org }}`; same env parametrisation as deploy.yml |

**New files:** none.

**Files to delete:**

| File | Reason |
|---|---|
| `repos/rule-executer/src/controllers/rule.ts` (feat-paysys) | Orphaned — never imported; contains `console.log("hello bhai…")` |
| `repos/rule-executer/simple-rule2-test.js` (feat-paysys) | Ad-hoc script; not part of jest suite. Delete or convert to jest. |

**Fix-in-place:**

| File | Change |
|---|---|
| `repos/rule-executer/.husky/pre-commit` | `npx lint - staged` → `npx lint-staged` |

### Frontend Code Impact

No frontend. Rule Studio UI is not touched by this issue.

Test fixture scope: any smoke-test rule (e.g. rule-901) needs one end-to-end deploy per operator (tazama-lf and Paysys) to confirm the parametrised workflow works both ways.

---

## Side Effects and Risks

### Risks of Track A Alone

| Risk | Likelihood | Mitigation |
|---|---|---|
| `dev`-branch rule-executer fails to build under the workflow (never exercised there before) | Medium | Dry-run the workflow on a throwaway rule before merging Track A |
| Paysys operator sees `deploy-to-uat.yml` still hardcodes `@psl-copilot` and continues without Track B | High | Explicitly document Track A as "unblock tazama-lf only; @psl-copilot deferred" |
| Bootstrap service is actually pointed at `psl-copilot/rule-template`, making Track A a no-op for new rules | Unknown until verified | Confirm with ops before shipping Track A |
| Redirecting to `dev` picks up an in-flight dev change that hasn't been UAT-tested | Low-Medium | Pin the workflow clone to a tagged commit on `dev` if `dev` is volatile |

### Risks of Track B

| Risk | Likelihood | Mitigation |
|---|---|---|
| Losing Paysys-specific behaviour when discarding `feat-paysys` | Medium | Walk `feat-paysys` diff with Paysys team; capture any needed behaviour as `dev` env vars |
| Workflow parametrisation regresses existing UAT deploys | Medium | Roll out to a canary rule first; keep the Paysys defaults in `inputs.*` defaults |
| Removing `@psl-copilot` `.npmrc` scope breaks Paysys install path | High | Preserve scope but gate on env var; keep secret available in Paysys deploy environment |
| `frms-coe-lib` bump introduces API drift into `execute.ts` | Medium | Run full jest suite on `dev` after the bump; typescript compile step will catch signature drift |
| Dockerfile secret-mount conversion breaks local `docker build` for developers who used ARG | Low | Document new build command; keep buildx as required tooling |

### Cross-Issue Dependencies

- **Cross-repo:** Track A and Track B both touch `rule-studio-example` workflow files. Any other open issue against `rule-studio-example` (Docker Hub push, workflow inputs) must be sequenced against Track B parametrisation to avoid merge conflicts.
- **Bootstrap service dependency:** the fate of `rule-studio-devtestops/.env.sample` (`GITHUB_TEMPLATE_OWNER=psl-copilot`) determines whether this issue's fix reaches new rules. If ops confirm `psl-copilot/rule-template` is the live template, an equivalent fix must land there — coordinate with the Paysys team.
- **`rule-studio-example` `feat-paysys` branch:** 19 commits ahead of `dev` with Docker Hub push. Any Track B on `dev` needs a decision on whether that branch is merged forward or archived.

---

## Effort Estimate

| Track | Files | Effort |
|---|---|---|
| Track A | 2 workflow files (3 lines) + smoke test + ops verification | 3-4 h |
| Track B | ~8 files across rule-executer + rule-studio-example + rule-studio-devtestops; workflow parametrisation; branch reconciliation | 2-3 engineering days + coordination |
| **Total** | | **~3 days engineering + external verification** |

---

## Acceptance Criteria (Verification Checklist)

### Track A

- [ ] `rule-studio-example/.github/workflows/deploy.yml` line 39 reads `-b dev`.
- [ ] `rule-studio-example/.github/workflows/deploy-to-uat.yml` line 36 reads `-b dev`.
- [ ] Dead `sed` line at `deploy.yml:66` removed.
- [ ] One rule end-to-end deployment on tazama-lf infrastructure succeeds from the redirected workflow.
- [ ] Confirmation captured (in this issue) of which template repo the bootstrap service actually uses.
- [ ] `deploy-to-uat.yml`'s continued reliance on `@psl-copilot` explicitly documented as deferred to Track B.

### Track B

- [ ] `frms-coe-lib` on rule-executer `dev` resolves from a public registry (no `0.0.1-psl.0`).
- [ ] `.npmrc` has no unconditional `@psl-copilot` scope.
- [ ] `Dockerfile` uses `RUN --mount=type=secret,id=GH_TOKEN` for private registry auth; no `ARG GH_TOKEN` leaks.
- [ ] `src/controllers/rule.ts` deleted (or merged and imported).
- [ ] `simple-rule2-test.js` deleted or promoted to a jest test.
- [ ] `.husky/pre-commit` runs `npx lint-staged` successfully.
- [ ] `src/controllers/execute.ts` has no `[L##]` debug log prefixes and no `as any` casts.
- [ ] `deploy.yml` docker run values (`RAW_HISTORY_DATABASE_HOST`, `SERVER_URL`, `REDIS_HOST`, `RULE_VERSION`) sourced from workflow inputs or repo variables.
- [ ] `deploy-to-uat.yml` `@psl-copilot` references sourced from `inputs.npm_org` with a Paysys-preserving default.
- [ ] `rule-studio-example` `feat-paysys` branch either merged to `dev` or archived with a documented reason.
- [ ] Template repo drift (`psl-copilot/rule-template` vs `tazama-lf/rule-studio-example`) resolved with ops.
- [ ] Both tazama-lf and Paysys deployments succeed against the parametrised workflow.

---

## Recommended Sequencing

1. **Verify bootstrap template repo** with ops (`rule-studio-devtestops/.env.sample`). If the live config points at `psl-copilot/rule-template`, decide whether to redirect `.env.sample` at `tazama-lf/rule-studio-example` or apply Track A/B to `psl-copilot/rule-template`. This is a gate on everything else.
2. **Ship Track A** on `rule-studio-example` — redirect `deploy.yml` and `deploy-to-uat.yml` to `-b dev`, remove the dead `sed` line. Smoke-test one rule end-to-end on tazama-lf infra.
3. **Independent cleanup commits on rule-executer `dev`** in parallel: fix `.husky/pre-commit`, delete `src/controllers/rule.ts` (if it ever gets ported forward), strip `[L##]` log noise from `execute.ts` if any leaked into `dev`. These are safe stand-alone PRs.
4. **Track B workflow parametrisation** — introduce `inputs.npm_org`, `inputs.db_host`, `inputs.nats_url`, `inputs.redis_host`, `inputs.rule_version` on both `deploy.yml` and `deploy-to-uat.yml`. Coordinate with the Paysys operator on default values so Paysys deploys continue unchanged.
5. **Track B rule-executer Dockerfile** — convert `ARG GH_TOKEN` to secret mount; drop hardcoded ENV IPs. Coordinate with the workflow parametrisation PR so image and workflow ship together.
6. **Reconcile `rule-studio-example` `feat-paysys`** (19 commits ahead) — merge forward or archive with rationale.
7. **Deferred:** once Track B is running for both operators, decide whether to delete `feat-paysys` on `rule-executer` or keep it as a documented operator overlay.

---
