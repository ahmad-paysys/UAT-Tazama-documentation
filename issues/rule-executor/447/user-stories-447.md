# Issue #447 — User Stories: Reconcile feat-paysys with dev and Retarget Deploy Workflows

**Assigned developer:** 1 developer
**Tracks:** Track A (retarget deploy workflows) → Track B (reconcile executer branches)
**Merge order:** Track A ships first (across `rule-studio-example` and, if applicable, `psl-copilot/rule-template`). Track B lands on `rule-executer/dev` after Track A is live. `feat-paysys` is deleted last.

---

## US-447-1 — Retarget `deploy.yml` Clone Target to `dev`

**Title:** Point `deploy.yml` at `rule-executer/dev` instead of the stale `feat-paysys` branch

**Body:**
The studio-template `deploy.yml` currently clones `rule-executer` from `feat-paysys`, which is 28 commits ahead of and 35 commits behind the maintained `dev` branch. Every rule bootstrapped from the studio template therefore deploys from a frozen fork. This story flips the branch argument to `dev` so new deployments pick up the maintained code path.

**Scope (Track A):**
- Repo: `tazama-lf/rule-studio-example` (branches `main` and `dev`)
- File: `.github/workflows/deploy.yml`
- Change: on line 39 (main/dev) or line 41 (feat-paysys branch of the same repo), replace `-b feat-paysys` with `-b dev` on the `git clone https://github.com/tazama-lf/rule-executer` invocation
- No migration, no frontend changes
- Note: the `sed` operation later in the workflow (around line 66) is dead code — the workflow then runs `npm pkg delete dependencies.rule` and `npm install --save-exact rule@npm:@$ORG/$RULE_NAME@latest` (lines 108–111) which installs the rule regardless. This story does not touch that sed.

**Acceptance Criteria:**
- [ ] `deploy.yml` no longer contains the string `feat-paysys`
- [ ] The clone step succeeds against `dev`
- [ ] A fresh rule bootstrap using the updated template completes end-to-end
- [ ] The resulting image, once running, uses the maintained `dev` code path

**Testing:**
- Trigger a bootstrap via `rule-studio-devtestops` and confirm the deploy log shows a successful clone from `dev`
- Confirm the deployed rule processes a sample transaction

---

## US-447-2 — Retarget `deploy-to-uat.yml` Clone Target to `dev`

**Title:** Point `deploy-to-uat.yml` at `rule-executer/dev` instead of `feat-paysys`

**Body:**
`deploy-to-uat.yml` is the Paysys-specific UAT deploy workflow. Like `deploy.yml`, it clones `rule-executer` from `feat-paysys`. This story flips it to `dev`. The workflow remains Paysys-specific because line 104 still hardcodes the `@psl-copilot` GitHub org for the rule package — that is not in scope here.

**Scope (Track A):**
- Repo: `tazama-lf/rule-studio-example`
- File: `.github/workflows/deploy-to-uat.yml`
- Change: line 36 — replace `-b feat-paysys` with `-b dev` on the `git clone https://github.com/tazama-lf/rule-executer` invocation
- No migration, no frontend changes

**Acceptance Criteria:**
- [ ] `deploy-to-uat.yml` no longer references `feat-paysys` on its clone step
- [ ] A UAT deploy through this workflow succeeds
- [ ] The Paysys-specific `@psl-copilot` hardcoding on line 104 is untouched (out of scope)

**Testing:**
- Run the UAT deploy end-to-end against a test rule and confirm the executer image starts

---

## US-447-3 — Apply Track A Across All Template Sources and Verify Template Ownership

**Title:** Verify which template repo `rule-studio-devtestops` actually uses in production; apply Track A there too

**Body:**
`rule-studio-devtestops`'s `.env.sample` points at `GITHUB_TEMPLATE_OWNER=psl-copilot` / `GITHUB_TEMPLATE_REPO=rule-template`. If production uses `psl-copilot/rule-template` rather than `tazama-lf/rule-studio-example`, then the Track A change to `deploy.yml` and `deploy-to-uat.yml` must be applied there too — otherwise Track A ships nothing meaningful. This story is the verification-and-apply step. It also covers the `feat-paysys` branch of `rule-studio-example` if that branch is still in use.

**Scope (Track A):**
- Confirm the production value of `GITHUB_TEMPLATE_OWNER` / `GITHUB_TEMPLATE_REPO` used by the running `rule-studio-devtestops` instance (not just `.env.sample`)
- If it resolves to `psl-copilot/rule-template`, open the equivalent PR against `psl-copilot/rule-template` with the same `-b feat-paysys` → `-b dev` change in both workflow files
- If `rule-studio-example`'s `feat-paysys` branch is still active for anyone, apply the same change on that branch as well (or explicitly document that the branch is abandoned)
- No code changes to `rule-executer` in this story

**Acceptance Criteria:**
- [ ] The production template repo used by `rule-studio-devtestops` is confirmed and named in the PR body
- [ ] Both workflow files on the production template point to `-b dev`
- [ ] `rule-studio-example`'s `feat-paysys` branch either receives the same fix or is documented as abandoned
- [ ] A rule bootstrap from the actual production template deploys from `dev`

**Testing:**
- Read the deployed `rule-studio-devtestops` `.env` (not `.env.sample`) or its runtime config to confirm the real template
- Bootstrap a new rule through the confirmed production path and inspect the clone step in the workflow log

---

## US-447-4 — Dockerfile Cleanup: Remove Hardcoded IPs, Restore APM, Restore BuildKit Secret

**Title:** Restore the `dev` Dockerfile behaviour in `rule-executer` — remove hardcoded UAT values

**Body:**
`feat-paysys`'s `Dockerfile` hardcodes `10.10.80.18` and port `15432` for the Paysys UAT environment, disables APM, changes `SERVER_URL` from `0.0.0.0:4222`, sets `RULE_NAME="rule-901"` instead of `"901"`, and drops the BuildKit `GH_TOKEN` secret mount. The hardcoded IPs are overridden by `docker run -e` from the deploy workflow, so this is not a runtime breakage in the pipeline — but any operator running the image outside the workflow gets a broken container, and APM being off blinds observability regardless. This story restores the maintainable `dev` shape.

**Scope (Track B):**
- Repo: `tazama-lf/rule-executer` (target branch `dev`)
- File: `Dockerfile`
- Remove hardcoded `10.10.80.18` and port `15432`
- Set `APM_ACTIVE=true`
- Restore `SERVER_URL=0.0.0.0:4222`
- Set `RULE_NAME="901"` (matches the `@tazama-lf/rule-901` package convention)
- Restore `--mount=type=secret,id=GH_TOKEN,env=GH_TOKEN` on the `npm install` layer
- No schema migration, no frontend changes

**Acceptance Criteria:**
- [ ] `grep -E "10\.10\.80\.18|15432" Dockerfile` returns nothing
- [ ] `APM_ACTIVE=true` present
- [ ] `SERVER_URL=0.0.0.0:4222` present
- [ ] `RULE_NAME="901"` present
- [ ] BuildKit `GH_TOKEN` secret mount restored on `npm install`
- [ ] `docker build --secret id=GH_TOKEN,env=GH_TOKEN .` builds cleanly
- [ ] Image built from the new Dockerfile starts without requiring workflow-level `-e` overrides

**Testing:**
- Build the image locally with a real `GH_TOKEN`
- Boot the container with only the standard env vars (no IP overrides) against a dev-mode NATS / DB and confirm it comes up
- Check APM traces show up in the APM server

---

## US-447-5 — `package.json`: Align Library Versions, Restore Rule Placeholder, Bump Version

**Title:** Bring `package.json` back onto the mainline library versions

**Body:**
`feat-paysys` pinned `frms-coe-lib` at `0.0.1-psl.0` and `frms-coe-startup-lib` at `3.0.2-rc.5`, dropped the `rule` npm placeholder (used by the workflow's `npm pkg delete dependencies.rule` + reinstall dance), and left the package version at `3.0.0`. This story realigns to the maintained versions.

**Scope (Track B):**
- Repo: `tazama-lf/rule-executer` (target branch `dev`)
- File: `package.json`
- `@tazama-lf/frms-coe-lib`: `0.0.1-psl.0` → `8.2.0-rc.6`
- `@tazama-lf/frms-coe-startup-lib`: `3.0.2-rc.5` → `3.1.0-rc.8`
- Restore `"rule": "npm:@tazama-lf/rule-901@4.0.0-rc.6"` in `dependencies`
- Bump `"version"`: `3.0.0` → `4.0.0-rc.2`
- Regenerate `package-lock.json`
- No schema migration, no frontend changes

**Acceptance Criteria:**
- [ ] `npm ci` succeeds with the new versions
- [ ] `npm run build` succeeds
- [ ] `npm test` passes
- [ ] The `rule` placeholder is present so that `npm pkg delete dependencies.rule` in the deploy workflow has something to delete
- [ ] `package.json` version reads `4.0.0-rc.2`

**Testing:**
- Fresh clone → `npm ci` → `npm run build` → `npm test`
- Simulate the workflow's rule-swap: `npm pkg delete dependencies.rule && npm install --save-exact rule@npm:@tazama-lf/rule-901@4.0.0-rc.6`

---

## US-447-6 — Resolve `.npmrc` `@psl-copilot` Scope Decision

**Title:** Decide the fate of the `@psl-copilot` npm scope entry in `.npmrc`

**Body:**
`.npmrc` on `feat-paysys` carries `@psl-copilot:registry=https://npm.pkg.github.com`, which is actually used by `deploy-to-uat.yml` when it installs `@psl-copilot/$RULE_NAME`. This is not simply dead config. Two options exist:
- (a) Remove the scope from `dev` if the Paysys-specific `deploy-to-uat.yml` pipeline is being retired
- (b) Keep it on `dev` if that pipeline stays in service

This story is the decision-and-apply.

**Scope (Track B):**
- Repo: `tazama-lf/rule-executer` (target branch `dev`)
- File: `.npmrc`
- Confirm whether `deploy-to-uat.yml` (Paysys UAT) is being retired or kept
- If retired: remove the `@psl-copilot` scope line
- If kept: keep the line; document the reason in the PR body
- Record the decision and rationale explicitly in the PR body

**Acceptance Criteria:**
- [ ] Decision recorded in the Track B PR body with justification
- [ ] `.npmrc` on `dev` matches the decision
- [ ] If kept, a UAT deploy through `deploy-to-uat.yml` still resolves `@psl-copilot/$RULE_NAME` correctly
- [ ] If removed, `deploy-to-uat.yml` is either updated in the same window or documented as retired

**Testing:**
- If kept: run a UAT deploy and confirm the `@psl-copilot/*` install resolves
- If removed: confirm no build path on `dev` needs the scope

---

## US-447-7 — Strip Debug Logs and Remove `as any` Cast from `execute.ts`

**Title:** Clean up `src/controllers/execute.ts` — remove numbered debug logs and unsafe cast

**Body:**
`execute.ts` on `feat-paysys` contains 20+ `loggerService.log("[L##] ...")` calls added during Paysys debugging, plus a `databaseManager as any` cast that suppresses a type error rather than fixing it. Every deployed rule emits these lines on every transaction, and the cast hides a real type mismatch.

**Scope (Track B):**
- Repo: `tazama-lf/rule-executer` (target branch `dev`)
- File: `src/controllers/execute.ts`
- Delete every `loggerService.log` call whose message begins with a bracketed line marker (`[L01]`, `[L02]`, ... — all 20+)
- Remove the `databaseManager as any` cast; if the resulting type error is genuine, fix it properly (correct the type or the call signature) rather than reintroducing the cast
- Preserve all functional logic and the `handleTransaction` import from `'rule/lib'`
- No schema migration, no frontend changes

**Acceptance Criteria:**
- [ ] `grep -nE "\\[L[0-9]+\\]" src/controllers/execute.ts` returns nothing
- [ ] `grep -n "as any" src/controllers/execute.ts` returns nothing
- [ ] `npm run build` succeeds with strict typechecking
- [ ] `npm test` passes
- [ ] A transaction processed end-to-end produces log output free of `[L##]` markers

**Testing:**
- Run a transaction through the executer and inspect logs
- Confirm no new `TypeScript` errors surface where the `as any` used to be

---

## US-447-8 — Delete Orphan Files (`src/controllers/rule.ts`, `simple-rule2-test.js`)

**Title:** Delete `src/controllers/rule.ts` and `simple-rule2-test.js` — both are dead code

**Body:**
`src/controllers/rule.ts` on `feat-paysys` looks like a rule controller, but `execute.ts` imports `handleTransaction` from `'rule/lib'` — not from this file. Nothing in the codebase references `src/controllers/rule.ts`. It is a leftover artifact. `simple-rule2-test.js` is a 124-line standalone script that is not part of the test suite and is not referenced by any tooling. Both should be deleted, not "cleaned up and merged."

**Scope (Track B):**
- Repo: `tazama-lf/rule-executer` (target branch `dev`)
- Files:
  - `src/controllers/rule.ts` — **delete**
  - `simple-rule2-test.js` — **delete**
- No schema migration, no frontend changes

**Acceptance Criteria:**
- [ ] `src/controllers/rule.ts` does not exist on `dev`
- [ ] `simple-rule2-test.js` does not exist on `dev`
- [ ] `grep -r "controllers/rule" .` returns no imports (confirming the file was truly orphaned)
- [ ] `npm run build` and `npm test` still pass

**Testing:**
- `npm ci && npm run build && npm test`
- Confirm no import errors related to `controllers/rule`

---

## US-447-9 — Fix Broken Husky Pre-Commit; Delete `feat-paysys` Branch

**Title:** Repair `.husky/pre-commit`, realign `src/config.ts` whitespace, and delete the `feat-paysys` branch

**Body:**
`.husky/pre-commit` on `feat-paysys` reads `npx lint - staged` (with stray spaces) — a broken command that silently no-ops, so commits skip lint entirely. Fix it to `npx lint-staged`. `src/config.ts` also has drifted whitespace on `feat-paysys` — restore alignment to match `dev`'s style. Finally, after the Track B PR merges to `dev`, delete `feat-paysys` from `tazama-lf/rule-executer` so it cannot be pinned again by accident.

**Scope (Track B):**
- Repo: `tazama-lf/rule-executer` (target branch `dev`)
- Files:
  - `.husky/pre-commit` — `npx lint - staged` → `npx lint-staged`
  - `src/config.ts` — whitespace realignment only, no logic changes
- Post-merge action: `git push origin --delete feat-paysys` on `tazama-lf/rule-executer`
- No schema migration, no frontend changes

**Acceptance Criteria:**
- [ ] `.husky/pre-commit` reads exactly `npx lint-staged` (no stray spaces)
- [ ] A commit with a lint-violating change fails at the pre-commit hook (proving the hook now runs)
- [ ] `src/config.ts` matches `dev` formatting; no behaviour change
- [ ] After Track B merges, `feat-paysys` no longer exists on `origin`
- [ ] Track A workflows on `rule-studio-example` (and, if applicable, `psl-copilot/rule-template`) still resolve because they now point at `dev`, not `feat-paysys`

**Testing:**
- Introduce a deliberate lint violation, attempt to commit, confirm the hook blocks it
- `git ls-remote origin feat-paysys` returns empty after the branch is deleted
- Trigger one bootstrap through the studio template to confirm no downstream job is still expecting `feat-paysys`
