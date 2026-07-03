# User Stories — Issue #447

---

## US-447-1 — Fix hardcoded clone target in Rule Studio deploy workflows

**Body:**
The Rule Studio deploy workflows in `tazama-lf/rule-studio-example` clone `rule-executer` from the `feat-paysys` branch, which is a diverged, unreviewed fork. This causes every Studio-generated rule to be built against executer code that misses 35 upstream commits and carries debug artifacts. Changing the clone target to `dev` ensures all future deployments build from the maintained branch.

**Scope:**
- Track: A
- Files: `.github/workflows/deploy.yml` (line 39), `.github/workflows/deploy-to-uat.yml` (line 36) in `tazama-lf/rule-studio-example`
- Change: replace `-b feat-paysys` with `-b dev` in both files
- Migration required: No
- Frontend changes: No

**Acceptance Criteria:**
- [ ] `deploy.yml:39` reads `git clone https://github.com/tazama-lf/rule-executer -b dev`
- [ ] `deploy-to-uat.yml:36` reads `git clone https://github.com/tazama-lf/rule-executer -b dev`
- [ ] A triggered Rule Studio rule deploy shows a `dev` branch SHA in the clone step log
- [ ] No change made to `feat-paysys` or any rule repo

**Testing:**
- Trigger `deploy.yml` via `workflow_dispatch` in any Rule Studio–generated rule repo
- Inspect the "Clone Rule Executer Repo" step log for the `dev` branch SHA
- Confirm no `feat-paysys` reference appears in the log

---

## US-447-2 — Remove hardcoded UAT infrastructure from feat-paysys Dockerfile

**Body:**
`feat-paysys/Dockerfile` bakes UAT IP addresses and port numbers into ENV defaults (`RAW_HISTORY_DATABASE_HOST=10.10.80.18`, etc.) and disables APM globally (`APM_ACTIVE=false`). These values override runtime env vars injected by `docker run`, breaking rule containers in any non-UAT environment and silencing observability. Restoring empty defaults and re-enabling APM makes the image portable.

**Scope:**
- Track: B
- Files: `Dockerfile` in `tazama-lf/rule-executer`
- Change: restore empty `ENV` defaults for all database host/port/user/password fields; set `APM_ACTIVE=true`; restore BuildKit secret mount (`--mount=type=secret,id=GH_TOKEN`) in both build stages
- Migration required: No
- Frontend changes: No

**Acceptance Criteria:**
- [ ] `Dockerfile` contains no hardcoded IP addresses in ENV defaults
- [ ] `APM_ACTIVE` defaults to `true`
- [ ] `RAW_HISTORY_DATABASE_HOST`, `CONFIGURATION_DATABASE_HOST`, and `EVENT_HISTORY_DATABASE_HOST` are all empty strings in the built image defaults
- [ ] BuildKit secret mount used in Stages 1 and 2 (not plain `ARG GH_TOKEN`)
- [ ] A container started with correct runtime env vars connects to the correct database

**Testing:**
- `docker inspect <container>` → env → confirm `APM_ACTIVE=true` and all DB hosts are empty
- Override with `-e RAW_HISTORY_DATABASE_HOST=<test-host>` at `docker run`; confirm the app picks it up

---

## US-447-3 — Align package.json library versions and restore rule placeholder

**Body:**
`feat-paysys/package.json` pins `frms-coe-lib` at `0.0.1-psl.0` (a Paysys-internal pre-release unavailable from the `tazama-lf` npm registry) and removes the `rule` placeholder dependency. The missing placeholder silently breaks the `sed` patching step in `deploy.yml`, which substitutes the rule package name at build time. Aligning to the `dev` versions and restoring the placeholder fixes both issues.

**Scope:**
- Track: B
- Files: `package.json` in `tazama-lf/rule-executer`
- Change: set `frms-coe-lib` to `8.2.0-rc.6`, `frms-coe-startup-lib` to `3.1.0-rc.8`; add `"rule": "npm:@tazama-lf/rule-901@4.0.0-rc.6"` as the placeholder
- Migration required: No
- Frontend changes: No

**Acceptance Criteria:**
- [ ] `package.json` `frms-coe-lib` version is `8.2.0-rc.6`
- [ ] `package.json` `frms-coe-startup-lib` version is `3.1.0-rc.8`
- [ ] `package.json` contains a `rule` dependency matching the pattern `"rule": "npm:@…/rule-placeholder@latest"` (or the current `dev` equivalent)
- [ ] The `sed` step in `deploy.yml` line 66 produces a visible substitution in the workflow log
- [ ] `npm ci` in the builder stage resolves all packages from the `tazama-lf` GitHub Packages registry without error

**Testing:**
- Run `deploy.yml` workflow; confirm "Modify Rule Executer Files" step log shows the `rule` key was updated
- Confirm `docker run` produces no `MODULE_NOT_FOUND` error for `frms-coe-lib`

---

## US-447-4 — Strip debug log statements from execute.ts and remove any cast

**Body:**
`feat-paysys/src/controllers/execute.ts` was modified to add 20+ verbose `loggerService.log('[L##]…')` trace messages and a `databaseManager as any` type cast to suppress a type error. These changes pollute production container logs and mask a type mismatch that should be resolved properly. Stripping the debug calls and removing the cast aligns the file with `dev`.

**Scope:**
- Track: B
- Files: `src/controllers/execute.ts` in `tazama-lf/rule-executer`
- Change: remove all `[L##]`-prefixed `loggerService.log` calls; remove `databaseManager as any` cast
- Migration required: No
- Frontend changes: No

**Acceptance Criteria:**
- [ ] No log message prefixed with `[L` exists in `execute.ts`
- [ ] `databaseManager` is passed without `as any` to `handleTransaction`
- [ ] TypeScript compilation passes without `noImplicitAny` suppression
- [ ] Container logs for a normal transaction contain only the log calls present on `dev`

**Testing:**
- `tsc --noEmit` passes on the corrected file
- Submit a test transaction; grep container logs for `\[L` — no matches

---

## US-447-5 — Resolve rule.ts BaseMessage handler and delete feat-paysys branch

**Body:**
`feat-paysys` introduces `src/controllers/rule.ts`, a new file providing a `handleTransaction` function for `BaseMessage` transactions. The file contains raw `console.log("hello bhai…")` debug output and is not imported by `execute.ts` (which still imports from `rule/lib`). A decision must be made: if `BaseMessage` handling is required by the pipeline, clean the file and merge it into `dev` via a reviewed PR; otherwise, confirm it is not needed and discard it. Once resolved, delete the `feat-paysys` branch.

**Scope:**
- Track: B
- Files: `src/controllers/rule.ts` in `tazama-lf/rule-executer` (new file, decision-dependent); `feat-paysys` remote branch
- Change: merge cleaned `rule.ts` into `dev` OR confirm absent; delete `feat-paysys`
- Migration required: No
- Frontend changes: No

**Acceptance Criteria:**
- [ ] A documented decision exists (PR description or issue comment) on whether `BaseMessage` handling is required
- [ ] If merged: `src/controllers/rule.ts` on `dev` contains no `console.log` statements and compiles cleanly
- [ ] If discarded: `src/controllers/rule.ts` does not exist on `dev`
- [ ] `feat-paysys` branch is deleted: `gh api repos/tazama-lf/rule-executer/branches/feat-paysys` returns 404
- [ ] `simple-rule2-test.js` is deleted

**Testing:**
- After Track B PR merges: `git ls-remote --heads origin feat-paysys` → no output
- If `rule.ts` merged: end-to-end test with a `BaseMessage` transaction produces a valid `RuleResult`
- If `rule.ts` discarded: end-to-end test with a standard transaction produces a valid `RuleResult` from `rule/lib`
