# Impact Study — Issue #447: Rule Studio pipeline builds against diverged, hardcoded feat-paysys branch

**Issue:** [#447](https://github.com/tazama-lf/rule-executer/issues/447)
**Study Date:** 2026-07-03
**Author:** Ahmad Khalid
**Repository:** tazama-lf/rule-executer

---

## Summary

Every rule deployed through Rule Studio clones the `feat-paysys` branch of `tazama-lf/rule-executer` because two workflow files in the `tazama-lf/rule-studio-example` template repo hardcode `-b feat-paysys`. The branch is 28 commits ahead and 35 commits behind `dev`, carrying hardcoded UAT IPs, disabled APM, pre-release Paysys-internal library versions, and raw `console.log` debug output — none of which should reach a production pipeline. Track A is a two-line fix in the two workflow files. Track B reconciles the `feat-paysys` divergence into `dev` and deletes the branch.

---

## Confirmed Root Cause

- **`deploy.yml:39`** (`tazama-lf/rule-studio-example`): `git clone https://github.com/tazama-lf/rule-executer -b feat-paysys` — confirmed by reading the file.
- **`deploy-to-uat.yml:36`** (`tazama-lf/rule-studio-example`): identical hardcoded clone — confirmed by reading the file.
- `feat-paysys` vs `dev` diff (confirmed via `git diff origin/dev...origin/feat-paysys`):
  - `Dockerfile`: hardcoded `RAW_HISTORY_DATABASE_HOST=10.10.80.18`, `PORT=15432`, `APM_ACTIVE=false`, `SERVER_URL=10.10.80.18:14222`; plain `ARG GH_TOKEN` instead of BuildKit secret mount.
  - `package.json`: `frms-coe-lib` pinned at `0.0.1-psl.0` (vs `8.2.0-rc.6` on `dev`); no `rule` placeholder dependency.
  - `src/controllers/execute.ts`: 20+ `[L##]`-prefixed debug log calls; `databaseManager as any` cast.
  - `src/controllers/rule.ts`: new file present only on `feat-paysys`; contains `console.log("hello bhai…")` debug output.
  - `simple-rule2-test.js`: present only on `feat-paysys`.
- The `sed` step at `deploy.yml:66` targets `"rule": "npm:@[^/]+/rule-placeholder@latest"` but `feat-paysys/package.json` has no `rule` key — the substitution is a silent no-op (confirmed by reading both files).
- `git log --left-right origin/dev...origin/feat-paysys`: 35 commits ahead of `feat-paysys` in `dev`; 28 commits unique to `feat-paysys`.

---

## Track A — Switch clone target

### What Changes
Two workflow files in `tazama-lf/rule-studio-example`; one line each. No application code changes.

### Impact

| Area | Value |
|---|---|
| Files changed | 2 |
| Schema migration required | No |
| Frontend changes required | No |
| Downtime required | No |
| Risk of regression | Low — only changes which branch the pipeline clones |
| Reversibility | Fully reversible (revert the two lines) |

**What this fixes immediately:**
- All future Rule Studio deploys clone `dev` instead of `feat-paysys`
- New rule containers get all 35 upstream `dev` commits (security fixes, bug fixes)
- Debug logging, hardcoded IPs, and disabled APM are no longer baked into new builds

**What this does not fix:**
- Containers already deployed from `feat-paysys` remain running on diverged code until rebuilt
- `feat-paysys` branch still exists and still diverges
- The 28 `feat-paysys`-only commits (including `rule.ts`) remain unreviewed
- The dead `sed` pattern in `deploy.yml` is not corrected by this track
- No documentation of the build-time rule-injection contract

Track A is safe to ship in isolation.

---

## Track B — Reconcile feat-paysys with dev

### What Changes
Full audit and reconciliation of the 28 commits unique to `feat-paysys` into `dev`, cleanup of debug artifacts, and branch deletion.

### Schema Impact
None. No database schema changes involved.

### Backend Code Impact

| File | Repository | Change type |
|---|---|---|
| `Dockerfile` | `tazama-lf/rule-executer` | Remove hardcoded IPs/ports; restore empty ENV defaults; restore BuildKit secret mount |
| `package.json` | `tazama-lf/rule-executer` | Align library versions with `dev`; restore `rule` placeholder dependency |
| `.npmrc` | `tazama-lf/rule-executer` | Decide on `@psl-copilot` scope (add to `dev` or remove from `feat-paysys`) |
| `src/controllers/execute.ts` | `tazama-lf/rule-executer` | Remove all `[L##]` debug log statements; remove `as any` cast |
| `src/controllers/rule.ts` | `tazama-lf/rule-executer` | Evaluate `BaseMessage` handler: clean and merge into `dev`, or confirm obsolete |
| `simple-rule2-test.js` | `tazama-lf/rule-executer` | Delete (debug file, absent on `dev`) |
| `.husky/pre-commit` | `tazama-lf/rule-executer` | Align with `dev` hook |

### Frontend Code Impact
None. Rule Studio frontend is not involved.

---

## Side Effects and Risks

### Risks of Track A Alone

| Risk | Likelihood | Mitigation |
|---|---|---|
| `feat-paysys`-unique `rule.ts` provides `BaseMessage` handling required by Studio rules | Medium | Audit before shipping Track B; document whether `dev`'s `handleTransaction` (imported from `rule/lib`) supports `BaseMessage` |
| `dev` has a breaking change for `psl-copilot`-namespaced packages | Low | Verify registry scopes in `dev/.npmrc` before switching |
| Already-deployed containers not rebuilt; remain on `feat-paysys` code | Certain | Schedule a rebuild cycle for all live Studio-deployed rule containers after Track A merges |

### Risks of Track B

| Risk | Likelihood | Mitigation |
|---|---|---|
| `rule.ts` `BaseMessage` logic is required and its removal breaks rules | Medium | Test against a Rule Studio–generated rule end-to-end in UAT before deleting branch |
| Cleaning `execute.ts` log statements removes a trace needed for debugging | Low | Confirm with team; consider replacing `[L##]` labels with proper structured log levels instead of removing entirely |
| `@psl-copilot` registry scope in `.npmrc` is needed by some deployed rules | Medium | Check if any rule package is published under `@psl-copilot`; if so, add scope to `dev/.npmrc` |

### Cross-Issue Dependencies
None identified. No other open issue directory in `issues/rule-executor/` was found targeting the same files.

---

## Effort Estimate

| Track | Files changed | Estimated effort |
|---|---|---|
| Track A | 2 | 30 minutes |
| Track B | 7 | 1–2 days |
| **Total** | **9** | **1–2 days** |

---

## Acceptance Criteria (Verification Checklist)

### Track A
- [ ] `deploy.yml:39` clones branch `dev`
- [ ] `deploy-to-uat.yml:36` clones branch `dev`
- [ ] Triggered workflow log shows `dev` branch SHA — not `feat-paysys`
- [ ] Built container runs with correct upstream library versions

### Track B
- [ ] `Dockerfile` has no hardcoded IPs or ports in ENV defaults
- [ ] `APM_ACTIVE` defaults to `true`
- [ ] BuildKit secret mount (`--mount=type=secret,id=GH_TOKEN`) used in `Dockerfile`
- [ ] `package.json` has a `rule` placeholder dependency matching the `sed` pattern
- [ ] `src/controllers/execute.ts` has no `[L##]`-prefixed log calls and no `as any` cast
- [ ] `src/controllers/rule.ts` decision resolved: either cleanly on `dev` (no `console.log`) or confirmed absent
- [ ] `simple-rule2-test.js` deleted
- [ ] `feat-paysys` branch deleted from remote
- [ ] End-to-end rule execution passes in UAT from the corrected branch

---

## Recommended Sequencing

1. Ship Track A immediately as a standalone PR into `tazama-lf/rule-studio-example`.
2. Audit the 28 `feat-paysys`-only commits in parallel — specifically determine if `rule.ts` `BaseMessage` handling is required by the Studio pipeline or is covered by `dev`'s `rule/lib` import.
3. Open Track B PR into `dev` on `tazama-lf/rule-executer` once the audit is complete.
4. Rebuild and re-deploy all Rule Studio-generated rule containers from `dev` after Track A + Track B are both merged.
5. Delete `feat-paysys` after all containers are confirmed healthy.
