# PR Review: RSE #13 — feat: rule deployment workflow updated

**Repo:** tazama-lf/rule-studio-example
**Branch:** `feat-paysys-poc` → `main`
**Author:** MuhammadAli-Paysys (Muhammad Ali)
**Date Reviewed:** 2026-07-17
**Label:** none
**Size:** +250 / -146 lines across 4 files
**Commits:** 12 (`1ed2ba6`, `a263e90`, `5e6918a`, `7b09a5d`, `1420c15`, `8b33b76`, `9b93548`, `d6060d1`, `0e9f725`, `b885f51`, `7352910`, `81b43b3`)
**State:** OPEN (mergeStateStatus: BLOCKED, mergeable)
**HEAD SHA verified:** `81b43b3c5745ae6c59c8c479985f90d6685baa7d`
**Existing approvals:** none — CodeRabbit COMMENTED (2 outside-diff findings), no human approval yet

---

## Table of Contents

- [Overview](#overview)
- [What Changed (Detailed)](#what-changed-detailed)
  - [1. `.github/workflows/deploy.yml` — modernised production deploy](#1-githubworkflowsdeployyml--modernised-production-deploy)
  - [2. `.github/workflows/deploy-to-uat.yml` — partial UAT update](#2-githubworkflowsdeploy-to-uatyml--partial-uat-update)
  - [3. `package.json` — frms-coe-lib bump](#3-packagejson--frms-coe-lib-bump)
  - [4. `package-lock.json` — regenerated lockfile](#4-package-lockjson--regenerated-lockfile)
- [Code Quality Analysis](#code-quality-analysis)
  - [Strengths](#strengths)
  - [Issues and Observations](#issues-and-observations)
- [Security Assessment](#security-assessment)
- [Test Coverage](#test-coverage)
- [CodeRabbit Activity](#coderabbit-activity)
- [Summary and Verdict](#summary-and-verdict)
- [GitHub Review Comment](#github-review-comment)
- [Follow-up Review (2026-07-17)](#follow-up-review-2026-07-17)
  - [Changes Requested — Resolution Status](#changes-requested--resolution-status)
  - [New Issues Found in Updated Commits](#new-issues-found-in-updated-commits)
  - [Updated Verdict](#updated-verdict)
  - [GitHub Review Comment (Follow-up)](#github-review-comment-follow-up)

---

## Overview

This PR modernises the **production** rule-deployment workflow (`deploy.yml`) in the `rule-studio-example` template repo, which is cloned per rule to host the CI/CD pipeline. The template's four workflows drive the rule build/publish/deploy chain — this PR rewrites the deploy pipeline to (a) clone the `dev` branch of `rule-executer` instead of the abandoned `feat-paysys` branch, (b) accept `RUNNER_LABEL` and `SERVER_IP` as GitHub Actions variables so the same template can target multiple environments, (c) migrate Docker image builds from the insecure `--build-arg GH_TOKEN` (which persists the token in image layers) to the BuildKit `--mount=type=secret,id=GH_TOKEN` mount, and (d) upgrade `@tazama-lf/frms-coe-lib` from `0.0.1-psl.1` to `8.2.0-rc.6`.

The **UAT** workflow (`deploy-to-uat.yml`) was only *partially* updated — it flips the `rule-executer` branch from `feat-paysys` to `dev` and extracts the `Update Rule Dependency Name` step, but retains the pre-BuildKit `sed`-into-Dockerfile approach and the insecure `--build-arg GH_TOKEN` build. Since the `dev` branch of `rule-executer` uses BuildKit secret mounts (`RUN --mount=type=secret,id=GH_TOKEN,env=GH_TOKEN npm ci`), the UAT flow will silently fail to inject `.npmrc` credentials and `npm ci` will 401.

**Base branch:** `main` — most other PRs in this org target `dev`, and the RSE repo appears to use `main` as its default (there is no `dev` branch on origin per the fetch results). No mismatch to flag; but confirm with the maintainers that this template does not follow the dev-then-main flow used by CMS/BIAR.

**Failing CI checks:** none — all four CodeQL runs and CodeRabbit checks pass. `mergeStateStatus: BLOCKED` reflects missing human approval, not a check failure.

| File | Nature of Change |
|------|-----------------|
| `.github/workflows/deploy.yml` | Rewrite: dynamic runner/server IP, BuildKit secret mounts, split Dockerfile patch + rule-dep update, safer bash (`set -euo pipefail`). |
| `.github/workflows/deploy-to-uat.yml` | Partial update: branch swap + rule-dep step extracted, but Dockerfile patching still assumes pre-BuildKit `rule-executer` layout, and build still uses `--build-arg GH_TOKEN`. |
| `package.json` | Bump `@tazama-lf/frms-coe-lib` `0.0.1-psl.1` → `8.2.0-rc.6`. |
| `package-lock.json` | Regenerated for the above. |

[↑ Back to top](#pr-review-rse-13--feat-rule-deployment-workflow-updated)

---

## What Changed (Detailed)

### 1. `.github/workflows/deploy.yml` — modernised production deploy

**Runner and server IP become configurable variables** (lines 18–23):

```diff
     runs-on:
-    - server-18
+    - ${{ vars.RUNNER_LABEL || 'server-35' }}
     env:
       RULE_NAME: ${{ github.event.repository.name }}
       ORG: ${{ github.repository_owner }}
+      SERVER_IP: ${{ vars.SERVER_IP || '10.10.80.35' }}
```

The default runner also changes from `server-18` to `server-35`. Setups previously deploying to `server-18` without an override will now deploy to `server-35`; the maintainer needs to set `vars.RUNNER_LABEL=server-18` in every affected rule repo (or its org-level variable) to preserve the previous target.

**Rule-executer clone source** (line 40):

```diff
-          git clone https://github.com/tazama-lf/rule-executer -b feat-paysys
+          git clone https://github.com/tazama-lf/rule-executer -b dev
```

**Dockerfile patching rewritten** (lines 63–90) — the old workflow injected `.npmrc` via `sed` inside a `RUN` line and added `ARG GH_TOKEN`. The new step assumes the `dev`-branch Dockerfile already uses BuildKit secret mounts (`RUN --mount=type=secret,id=GH_TOKEN,env=GH_TOKEN npm ci --ignore-scripts`) and rewrites them to the stricter form (`--mount=type=secret,id=GH_TOKEN,required=true GH_TOKEN="$(cat /run/secrets/GH_TOKEN)" npm ci --ignore-scripts`). It also prepends `# syntax=docker/dockerfile:1.4` and applies the same patch to both the builder and production install lines.

**Rule dependency update split into its own step** (lines 92–98):

```yaml
- name: Update Rule Dependency Name
  run: |
    cd rule-executer-$RULE_NAME
    npm pkg delete dependencies.rule
    npm pkg set "dependencies.rule=npm:@$ORG/$RULE_NAME@latest"
```

This is safer than the previous `sed` on `package.json` because it uses npm's own JSON writer instead of a regex that assumed a specific quoted-string layout.

**`.npmrc` generation hardened** (lines 107–121) — bash set with `set -euo pipefail`, heredoc replaces echo-append chain, and the `.npmrc` is also copied into `rule-executer-$RULE_NAME/.npmrc` so it lives in the Docker build context.

**Docker build migrated to BuildKit secret** (lines 148–158):

```diff
       - name: Build Docker Image and Deploy Container Locally
+        shell: bash
         run: |
+          set -euo pipefail
+
           echo "Building Docker image locally on self-hosted runner..."
-          docker build --build-arg GH_TOKEN=${GH_TOKEN} -t $ORG/$RULE_NAME:latest rule-executer-$RULE_NAME
+          docker build \
+            --secret id=GH_TOKEN,env=GH_TOKEN \
+            -t "$ORG/$RULE_NAME:latest" \
+            "rule-executer-$RULE_NAME"
```

`--build-arg` embeds the token in image history (recoverable via `docker history`); `--secret` mounts it at `/run/secrets/GH_TOKEN` only during that specific `RUN` and never lands in a layer. This is a genuine security fix.

**`docker run` env vars use `$SERVER_IP`** (lines 165–198) — every previously-hardcoded `10.10.80.18` becomes `$SERVER_IP`. When `vars.SERVER_IP` is unset, this resolves to `10.10.80.35` — a different host than the old `10.10.80.18`. Same migration note as the runner label.

### 2. `.github/workflows/deploy-to-uat.yml` — partial UAT update

Only two changes land in UAT:

```diff
-          git clone https://github.com/tazama-lf/rule-executer -b feat-paysys
+          git clone https://github.com/tazama-lf/rule-executer -b dev
```

and extraction of the rule-dep update into its own step (lines 71–77). Everything else (`Modify Rule Executer Files`, `Configure npm for GitHub Packages`, `Build Docker Image`, and the `docker run` env block) retains the pre-BuildKit assumptions and the hardcoded `10.10.80.37` server IP.

The critical break: the old `sed -i '/^RUN npm ci/i\RUN echo ...'` command inserts an npmrc-writing line **immediately before** the first line matching `^RUN npm ci`. In the new `dev` branch, `rule-executer/Dockerfile` no longer has any bare `RUN npm ci`; every one is `RUN --mount=type=secret,id=GH_TOKEN,env=GH_TOKEN npm ci ...`. The `sed` therefore matches nothing, no npmrc is injected, and the BuildKit-secret `RUN` will fail to authenticate to GitHub Packages because the workflow builds with `--build-arg GH_TOKEN` (not `--secret`), meaning `/run/secrets/GH_TOKEN` does not exist inside the build.

### 3. `package.json` — frms-coe-lib bump

```diff
   "dependencies": {
-    "@tazama-lf/frms-coe-lib": "0.0.1-psl.1"
+    "@tazama-lf/frms-coe-lib": "8.2.0-rc.6"
   },
```

Huge jump — from a per-tenant fork tag (`0.0.1-psl.1`) to the upstream mainline release candidate (`8.2.0-rc.6`). The lockfile diff shows the new lib pulls in `cloudevents@^10`, `zod@^4.4.3`, `protobufjs@^7.5.5`, and a set of `is-*`/`for-each` de-`dev`ified deps. This is a *runtime* dependency that gets consumed by the built rule package, so any API break between `0.0.1-psl.1` and `8.2.0-rc.6` surfaces the moment a rule imports `@tazama-lf/frms-coe-lib`. The `src/rule.ts` template in this repo does not import it directly, so the risk is downstream — rules using the older typing/logic APIs must be revalidated.

### 4. `package-lock.json` — regenerated lockfile

Mechanical consequence of #3. Notable: the CodeRabbit walkthrough flagged that some previously `dev`/`peer` packages (`available-typed-arrays`, `for-each`, `has-tostringtag`, `is-callable`, `is-generator-function`, `is-regex`, `is-typed-array`, `safe-regex-test`, `possible-typed-array-names`, `which-typed-array`) are now runtime deps. Expected fallout of pulling `cloudevents`/`util` into the runtime tree.

[↑ Back to top](#pr-review-rse-13--feat-rule-deployment-workflow-updated)

---

## Code Quality Analysis

### Strengths

- **Real security improvement.** Moving from `--build-arg GH_TOKEN` to `--mount=type=secret,id=GH_TOKEN` in `deploy.yml` closes a genuine credential-leakage hole. The old approach persists the token in Docker image history — recoverable by any user who can pull the image or read its layers.
- **Bash hygiene.** New/rewritten steps use `shell: bash` and `set -euo pipefail`, which turns the previously silent-fail `sed`/`npm` chains into loud, early failures. This is exactly the improvement the workflow needed given how many previous rewrites went unnoticed until deploy time.
- **Configurable environment targets.** `RUNNER_LABEL` and `SERVER_IP` variables let the same template serve multiple environments (dev/staging/UAT) without a workflow fork per environment.
- **Replaced `sed` on package.json with `npm pkg`.** The old `sed 's|"rule": "npm:@psl-copilot/rule-placeholder@latest"|...|'` broke silently if the template's JSON layout ever changed. `npm pkg delete dependencies.rule && npm pkg set …` is layout-independent.
- **`.npmrc` copied into the build context.** Copying the generated `.npmrc` into `rule-executer-$RULE_NAME/.npmrc` future-proofs the build for Docker daemons that don't honour the runner's global `$HOME/.npmrc`.

### Issues and Observations

#### Issue 1 — `deploy-to-uat.yml` will silently fail to authenticate `npm ci` after the `-b dev` swap

**Severity: Major (Bug / Security)**

`deploy-to-uat.yml` lines 33–40 now clone `rule-executer` from `-b dev`. The `dev`-branch `Dockerfile` uses `RUN --mount=type=secret,id=GH_TOKEN,env=GH_TOKEN npm ci ...`, so the UAT patch step at lines 59–69:

```yaml
sed -i '/^RUN npm ci/i\RUN echo "@tazama-lf:registry=https://npm.pkg.github.com" > /root/.npmrc && \\...' rule-executer-$RULE_NAME/Dockerfile
```

matches nothing (no line begins `RUN npm ci`). No `.npmrc` is written into the image. Then the build at line 126:

```yaml
docker build --build-arg GH_TOKEN=${GH_TOKEN} -t psl-copilot/$RULE_NAME:latest rule-executer-$RULE_NAME
```

passes the token as a build-arg but the Dockerfile no longer declares `ARG GH_TOKEN`, so `$GH_TOKEN` is empty inside the build. Meanwhile the `--mount=type=secret,id=GH_TOKEN,env=GH_TOKEN` line in the Dockerfile requires a matching `--secret id=GH_TOKEN,env=GH_TOKEN` on the build command — which is not present. Result: `npm ci` fails with 401 Unauthorized against `npm.pkg.github.com`.

Corroborated by CodeRabbit (Critical, lines 59–70). Fix: apply the same Dockerfile patch + `.npmrc` copy + `docker build --secret` treatment as `deploy.yml`. Preferably factor the shared logic into a composite action or reusable workflow so `deploy.yml` and `deploy-to-uat.yml` cannot drift again.

#### Issue 2 — `deploy-to-uat.yml` still passes the token as `--build-arg` (Docker history leak)

**Severity: Major (Security)**

Same file, line 126:

```yaml
docker build --build-arg GH_TOKEN=${GH_TOKEN} -t psl-copilot/$RULE_NAME:latest rule-executer-$RULE_NAME
```

`GH_TOKEN` (a live GitHub PAT with `write:packages`) is captured in the image's layer history and is recoverable via `docker history --no-trunc`. The whole point of the `deploy.yml` migration was to close this hole; leaving UAT on the old path means production and UAT diverge on the security posture. Fix: mirror `deploy.yml` lines 148–158 (`docker build --secret id=GH_TOKEN,env=GH_TOKEN ...`).

#### Issue 3 — Default `RUNNER_LABEL` and `SERVER_IP` changed silently (breaking change for existing installs)

**Severity: Major (Breaking Change Risk)**

`deploy.yml` line 19 defaults `runs-on` to `server-35` (was previously `server-18`); line 23 defaults `SERVER_IP` to `10.10.80.35` (was `10.10.80.18`). Any rule repo that was successfully deploying under the old workflow — and has NOT explicitly set `vars.RUNNER_LABEL` and `vars.SERVER_IP` — will start hitting the new targets on the next merge. If those hosts are not the same box, containers will land on the wrong server.

Fix: pick one of (a) leave the defaults as `server-18` / `10.10.80.18` (the pre-PR values) and document the migration, (b) if the new hosts are intentionally the new default, add a short note to the PR description that every existing rule repo needs its variables set, or (c) fail the workflow early if `vars.RUNNER_LABEL` / `vars.SERVER_IP` are unset (`if: vars.RUNNER_LABEL == ''` → `exit 1`) to force explicit configuration rather than silent redirection.

#### Issue 4 — `frms-coe-lib` upgrade `0.0.1-psl.1` → `8.2.0-rc.6` is a pre-release; no test evidence in the PR body

**Severity: Minor (Test Coverage / Maintainability)**

The PR body says "aligning the rule package with the latest `frms-coe-lib` release required for the Paysys POC" and marks "Unit tests passing" — but the RSE template has no unit tests that exercise `frms-coe-lib` directly (the template only carries a rule shell). The risk is with downstream rule repos generated from this template; existing rules built against `0.0.1-psl.1` may break on API drift. Note: `rc.6` implies further RCs may follow before an `8.2.0` GA. Pinning to a release candidate in a template that gets cloned into every new rule repo means every future rule inherits an in-flight version.

Fix: either (a) pin to a GA release when one lands, or (b) document in the template README that consumers should upgrade to the GA when it exists. Non-blocking because it is a template pin, not runtime code, but worth acknowledging.

#### Issue 5 — `Update Rule Dependency Name` step in `deploy-to-uat.yml` hardcodes `@psl-copilot`

**Severity: Minor (Maintainability)**

`deploy-to-uat.yml` line 76: `npm pkg set "dependencies.rule=npm:@psl-copilot/$RULE_NAME@latest"`. The equivalent step in `deploy.yml` (line 97) uses `@$ORG/$RULE_NAME@latest` where `ORG` = `${{ github.repository_owner }}`. So the same template will deploy to `@<repository_owner>` on prod but always to `@psl-copilot` on UAT. This is fine when the UAT is exclusively for `psl-copilot` rules but breaks the template's reusability the moment another tenant forks it.

Fix: use `@$ORG/$RULE_NAME@latest` here too (matching `deploy.yml`), or set `ORG` explicitly in the UAT `env:` block and use it.

#### Issue 6 — `deploy.yml` uses `sed -i "s/placeholder/$RULE_ID/g"` with `/g`, previous UAT used no `/g`

**Severity: Informational (Consistency)**

`deploy.yml` line 77 uses `/g`; `deploy-to-uat.yml` line 64 uses the non-`/g` form. If the `rule-executer` Dockerfile ever contains the string `placeholder` more than once (currently it does not), the two workflows would produce different output. Cosmetic today; worth aligning.

[↑ Back to top](#pr-review-rse-13--feat-rule-deployment-workflow-updated)

---

## Security Assessment

| Concern | Assessment |
|---------|-----------|
| Token in Docker image layers (`--build-arg`) | **Fixed for prod, still present in UAT.** `deploy.yml` migrates to `--secret id=GH_TOKEN,env=GH_TOKEN` (line 155). `deploy-to-uat.yml` line 126 still uses `--build-arg GH_TOKEN=${GH_TOKEN}` — see Issue 2. |
| `.npmrc` on disk containing a token | Written to `$HOME/.npmrc` and copied into the build context. Ephemeral on GitHub-hosted or self-hosted runner workspaces, but persists until the runner cleans up. Not a new exposure — same as before. |
| `TAZAMA_TOKEN` scope | Not visible in the diff. The token is used for git clone of `rule-executer`, npm auth to `npm.pkg.github.com`, and Docker build. Requires `repo` + `write:packages` at minimum. Verify the org's PAT is scoped as narrowly as feasible (fine-grained token limited to the two repos it touches). |
| `docker run` command injection via `$RULE_NAME` / `$RULE_ID` | `RULE_NAME` = `github.event.repository.name`, `RULE_ID` = `${RULE_NAME#rule-}`. Repo names are already sanitised by GitHub, so injection is not exploitable in practice. Informational only. |
| SSRF / open redirect | Not applicable — no HTTP-response-based redirection. |
| CI security scanners | CodeQL Analyze (actions) and CodeQL Analyze (javascript-typescript) both pass on `81b43b3`. |

No new user-controlled input reaches a data or command layer. The security net-effect of the PR is **positive for prod** (token no longer leaks to image history) and **neutral / slightly worse for UAT** (still leaks + broken auth).

[↑ Back to top](#pr-review-rse-13--feat-rule-deployment-workflow-updated)

---

## Test Coverage

There is no automated test coverage for GitHub Actions workflows in this repo — the workflows are consumed as templates and only exercised end-to-end when a rule repo cut from this template is actually pushed. The PR body claims "Locally" and "Development Environment" validation; per the description "No local automated tests were run" — so the only signal is:

- The four CodeQL runs pass on `81b43b3`.
- CodeRabbit's Pre-merge checks pass (title, description, docstring, linked-issues, out-of-scope) — the CodeRabbit review nonetheless raises two Major/Critical findings inside the diff.
- No dry-run of `deploy-to-uat.yml` is documented.

**Gaps to raise with the author:**

1. Run the modernised `deploy.yml` end-to-end on at least one throwaway rule repo before merging, and record the resulting container in the PR body. The refactor is broad enough (Dockerfile patch, npm config, docker build) that a smoke test is warranted.
2. **Every Major in Issues above lacks a matching test.** For workflows this means at minimum a scripted validation ("clone rule-executer dev, run the patched Dockerfile through `docker build --secret` locally") captured as a manual test log in the PR body.
3. If the intent is to leave `deploy-to-uat.yml` untouched, the PR description should say so explicitly rather than leaving a partial change. As it stands, the UAT edits look accidental.

[↑ Back to top](#pr-review-rse-13--feat-rule-deployment-workflow-updated)

---

## CodeRabbit Activity

### Pass 1 — walkthrough + 2 outside-diff comments on `deploy-to-uat.yml`

**Commit reviewed:** `81b43b3`
**Findings:** 2 actionable comments (both on `deploy-to-uat.yml`, both outside-diff)

| Finding | Severity | Status |
|---------|----------|--------|
| `deploy-to-uat.yml` 59–70: UAT `sed` on `^RUN npm ci` no longer matches the dev-branch Dockerfile; align with `deploy.yml` BuildKit patch + `--secret` build | 🔴 Critical | ❌ Not resolved — corroborates Issue 1 |
| `deploy-to-uat.yml` 86–97: `.npmrc` not copied into the UAT build context; align with `deploy.yml` | 🟠 Major | ❌ Not resolved — corroborates Issue 1 (same root cause) |

Both findings are corroborated independently under Issues 1 and 2. The 3.1 hunt that caught them is **parallel-siblings / guard-scope asymmetry** — `deploy.yml` and `deploy-to-uat.yml` are the sibling deploy pipelines, and the fix landed in one but not the other. CodeRabbit's framing is correct; the author's response is not yet in the PR thread.

[↑ Back to top](#pr-review-rse-13--feat-rule-deployment-workflow-updated)

---

## Summary and Verdict

**Verdict: Changes Requested**

The `deploy.yml` rewrite is a real improvement — it closes a token-in-image-history leak, hardens the bash, and makes runner/server targets configurable. But the same refactor was applied to `deploy-to-uat.yml` only *halfway*: the `rule-executer` branch was flipped to `dev` (which uses BuildKit secret mounts) while the UAT patch script and `docker build` command still assume the pre-BuildKit layout. That will silently 401 on `npm ci` at build time, and it leaves the token-in-history hole open specifically on UAT. In addition, the silent default changes for `RUNNER_LABEL` (`server-18` → `server-35`) and `SERVER_IP` (`10.10.80.18` → `10.10.80.35`) will misroute every rule repo that has not yet set these variables — a compatibility trap that needs either a rollback of the defaults or an explicit migration note (and ideally a fail-fast check in the workflow).

### Blocking

1. **`deploy-to-uat.yml` will fail `npm ci` with 401** — the UAT clone now pulls the `dev` `rule-executer`, whose Dockerfile uses `RUN --mount=type=secret,id=GH_TOKEN,...`; the UAT `sed` no longer matches, no `.npmrc` is injected, and the build uses `--build-arg` (not `--secret`). Apply the same patch + `--secret` build treatment as `deploy.yml`, or split the two workflows onto different rule-executer branches, or (best) extract the shared logic into a reusable workflow so the two cannot drift again.
2. **`deploy-to-uat.yml` still passes `GH_TOKEN` as `--build-arg`** — leaks the token into Docker image history. Migrate to `--secret id=GH_TOKEN,env=GH_TOKEN` mirroring `deploy.yml` line 155.
3. **Silent default change for `RUNNER_LABEL` and `SERVER_IP`** — every rule repo that had been working against `server-18` / `10.10.80.18` without explicit vars will silently retarget to `server-35` / `10.10.80.35`. Either roll defaults back to the pre-PR values, fail the workflow when the vars are unset, or explicitly document and coordinate with every affected rule repo before merging.

### Non-blocking but recommended

4. **`deploy-to-uat.yml` `Update Rule Dependency Name` hardcodes `@psl-copilot`** — should use `@$ORG/$RULE_NAME@latest` for template reusability (line 76).
5. **Pin to a stable `frms-coe-lib` release** — `8.2.0-rc.6` is an RC; when GA lands, bump the template so new rule repos inherit a stable version.

[↑ Back to top](#pr-review-rse-13--feat-rule-deployment-workflow-updated)

---

## GitHub Review Comment

````markdown
**Changes Requested**

The `deploy.yml` modernisation is solid — moving to BuildKit `--secret` mounts closes a real token-in-image-history leak, and the configurable `RUNNER_LABEL`/`SERVER_IP` variables are a good move. But `deploy-to-uat.yml` was only *partially* updated, and two of the changes here are silent breaking changes for any rule repo that has not yet been reconfigured.

---

### Blocking

**1. `deploy-to-uat.yml` will fail `npm ci` with 401 on the new `dev`-branch `rule-executer` Dockerfile.**

The clone at [`deploy-to-uat.yml:36`](.github/workflows/deploy-to-uat.yml#L36) now pulls `rule-executer -b dev`. The dev-branch `Dockerfile` uses `RUN --mount=type=secret,id=GH_TOKEN,env=GH_TOKEN npm ci ...` — no bare `RUN npm ci` line remains, so the `sed` at [`deploy-to-uat.yml:68`](.github/workflows/deploy-to-uat.yml#L68) (`/^RUN npm ci/i\RUN echo ...`) matches nothing and no `.npmrc` is injected. Then the build at [`deploy-to-uat.yml:126`](.github/workflows/deploy-to-uat.yml#L126) uses `--build-arg GH_TOKEN=…` but the Dockerfile no longer declares `ARG GH_TOKEN`, so `$GH_TOKEN` is empty inside the build. Meanwhile the BuildKit `--mount=type=secret,id=GH_TOKEN` line requires a matching `--secret id=GH_TOKEN,env=GH_TOKEN` on the build command, which isn't there. Result: `npm ci` 401s against `npm.pkg.github.com`.

Fix: mirror `deploy.yml`'s treatment — patch the Dockerfile in place with `set -euo pipefail`, copy `$HOME/.npmrc` into `rule-executer-$RULE_NAME/.npmrc`, and switch the build to:

```yaml
docker build \
  --secret id=GH_TOKEN,env=GH_TOKEN \
  -t "psl-copilot/$RULE_NAME:latest" \
  "rule-executer-$RULE_NAME"
```

Better still, factor `deploy.yml`'s "clone + patch + build" chain into a composite action (or a reusable workflow) and have both `deploy.yml` and `deploy-to-uat.yml` call it — that way the two cannot drift on the next rule-executer refactor.

**2. `deploy-to-uat.yml` still passes `GH_TOKEN` as `--build-arg`, leaking it into image history.**

Line 126: `docker build --build-arg GH_TOKEN=${GH_TOKEN} ...`. `docker history --no-trunc` on the built image will expose the token. This is exactly the hole `deploy.yml` line 155 closes with `--secret`; leaving UAT on the old path means prod and UAT have diverged on security posture. Same fix as #1.

**3. Default `RUNNER_LABEL` and `SERVER_IP` silently changed.**

[`deploy.yml:19`](.github/workflows/deploy.yml#L19) now defaults `runs-on` to `server-35` (was `server-18`); [`deploy.yml:23`](.github/workflows/deploy.yml#L23) defaults `SERVER_IP` to `10.10.80.35` (was `10.10.80.18`). Any rule repo cut from this template that has NOT set `vars.RUNNER_LABEL` and `vars.SERVER_IP` will silently start deploying to the new hosts on the next merge. Pick one:

- Roll the defaults back to the pre-PR values (`server-18` / `10.10.80.18`) and only override via `vars.*`.
- Or fail the workflow when the vars are unset, forcing explicit configuration:
  ```yaml
  - name: Require deployment vars
    if: vars.RUNNER_LABEL == '' || vars.SERVER_IP == ''
    run: |
      echo "::error::RUNNER_LABEL and SERVER_IP repo variables must be set"
      exit 1
  ```
- Or document the migration in the PR body and confirm every downstream rule repo has been updated *before* merging.

---

### Non-blocking (please address in this PR if possible)

**4. `deploy-to-uat.yml` line 76 hardcodes `@psl-copilot`.**

`deploy.yml` line 97 uses `@$ORG/$RULE_NAME@latest` where `ORG = ${{ github.repository_owner }}`. UAT should match; otherwise the template only works for the `psl-copilot` tenant on UAT.

**5. `frms-coe-lib` is pinned to `8.2.0-rc.6` — a release candidate.**

Fine for the POC, but every new rule repo cloned from this template inherits an in-flight version. Follow up with a bump to GA once `8.2.0` lands.
````

[↑ Back to top](#pr-review-rse-13--feat-rule-deployment-workflow-updated)

---

---

---

## Follow-up Review (2026-07-17)

**Reviewed commit:** `a5aedc0` — *"refactor: common variables targetted to 18 machine on default"* (2026-07-17)
**Reviewed against:** CHANGES_REQUESTED on commit `81b43b3` by `ahmad-paysys` ([review 4722253769](https://github.com/tazama-lf/rule-studio-example/pull/13#pullrequestreview-4722253769), 2026-07-17)
**Delta reviewed:** `81b43b3..a5aedc0` — 2 files changed, +57 / -28 lines (`deploy-to-uat.yml` +55/-27, `deploy.yml` +2/-2)
**HEAD SHA verified:** `a5aedc04009e6515a3381f759f33540bc3c2de09`
**New commits:** 2 (`09e15c9` "refactor: deploy to yat workflow made same", `a5aedc0` "refactor: common variables targetted to 18 machine on default")
**Developer response:** No inline reply to the CHANGES_REQUESTED review itself; the two follow-up commits speak for the author. A separate reply on the new CodeRabbit nitpick pass says `"this is not applicable"` ([comment #5003810516](https://github.com/tazama-lf/rule-studio-example/pull/13#issuecomment-5003810516)) — that response applies to the Pass-2 CodeRabbit nitpicks, not to the prior blocking items.

The delta is exactly the mirror-into-UAT that the initial review's Blocking (1) asked for, plus the defaults roll-back that Blocking (3) asked for. No new features, no scope expansion.

[↑ Back to top](#pr-review-rse-13--feat-rule-deployment-workflow-updated)

---

### Changes Requested — Resolution Status

#### Item 1 — `deploy-to-uat.yml` will fail `npm ci` with 401

**Status: RESOLVED** (commit `09e15c9`)

The `Modify Rule Executer Files` step is replaced by `Modify Rule Executer Dockerfile` ([`deploy-to-uat.yml:61-88`](repos/rule-studio-example/.github/workflows/deploy-to-uat.yml#L61-L88)) which now:

- Runs under `shell: bash` with `set -euo pipefail`.
- Prepends `# syntax=docker/dockerfile:1.4` (idempotently — deletes any prior syntax directive first).
- Rewrites both the builder `RUN --mount=type=secret,id=GH_TOKEN,env=GH_TOKEN npm ci --ignore-scripts` and the production `RUN --mount=type=secret,id=GH_TOKEN,env=GH_TOKEN npm ci --omit=dev --ignore-scripts` lines to the stricter `required=true GH_TOKEN="$(cat /run/secrets/GH_TOKEN)"` form.
- Ends with a `grep -n -A2 -B2 'npm ci' "$DOCKERFILE"` sanity print.

Complementary changes:
- `Configure npm for GitHub Packages` ([lines 105-119](repos/rule-studio-example/.github/workflows/deploy-to-uat.yml#L105-L119)) now uses a heredoc, `set -euo pipefail`, and `cp "$HOME/.npmrc" "rule-executer-$RULE_NAME/.npmrc"` — matching `deploy.yml`.
- `Build Docker Image` ([lines 146-156](repos/rule-studio-example/.github/workflows/deploy-to-uat.yml#L146-L156)) switches to `docker build --secret id=GH_TOKEN,env=GH_TOKEN -t "$ORG/$RULE_NAME:latest" "rule-executer-$RULE_NAME"`.

UAT is now structurally identical to prod for the clone + patch + npm + build chain. `npm ci` will authenticate correctly against `npm.pkg.github.com`.

#### Item 2 — `deploy-to-uat.yml` still passes `GH_TOKEN` as `--build-arg`

**Status: RESOLVED** (commit `09e15c9`)

Same fix as Item 1. Line 152-155 now uses `docker build --secret id=GH_TOKEN,env=GH_TOKEN ...`. `GH_TOKEN` is no longer captured in Docker image history. Prod and UAT security posture are aligned.

#### Item 3 — Silent default change for `RUNNER_LABEL` and `SERVER_IP`

**Status: RESOLVED** (commit `a5aedc0`)

`deploy.yml` line 19 now defaults to `server-18` (was `server-35` in the reviewed commit); line 23 defaults `SERVER_IP` to `10.10.80.18` (was `10.10.80.35`):

```diff
-    - ${{ vars.RUNNER_LABEL || 'server-35' }}
+    - ${{ vars.RUNNER_LABEL || 'server-18' }}
...
-      SERVER_IP: ${{ vars.SERVER_IP || '10.10.80.35' }}
+      SERVER_IP: ${{ vars.SERVER_IP || '10.10.80.18' }}
```

This restores the pre-PR runtime target for every rule repo that has not set `vars.RUNNER_LABEL` / `vars.SERVER_IP`. The variables remain overridable, so environments already opted-in to `server-35` / `10.10.80.35` (or any other host) can continue via repo/org vars. No silent breakage on merge.

Note: `deploy-to-uat.yml` line 21 still defaults `SERVER_IP` to `10.10.80.37` — this matches the pre-PR hardcoded UAT host, so no behavioural change for UAT consumers.

#### Item 4 — `deploy-to-uat.yml` `Update Rule Dependency Name` hardcodes `@psl-copilot`

**Status: RESOLVED** (commit `09e15c9`)

Every previously-hardcoded `@psl-copilot` in UAT is now `@$ORG` (with `ORG: ${{ github.repository_owner }}` added to the workflow env at line 20):

- Line 92 (log): `@$ORG/$RULE_NAME@latest`.
- Line 95 (`npm pkg set`): `dependencies.rule=npm:@$ORG/$RULE_NAME@latest`.
- Line 133 (`npm install --save-exact`): `rule@npm:@$ORG/$RULE_NAME@latest`.
- Line 154 (docker build tag): `"$ORG/$RULE_NAME:latest"`.
- Line 192 (docker run image): `$ORG/$RULE_NAME:latest`.

Template is now tenant-neutral for UAT as well as prod.

#### Item 5 — `frms-coe-lib` pinned to `8.2.0-rc.6`

**Status: ➖ Declined by author** (deferred to Justus / template maintainer)

Per the initial review, this was flagged as non-blocking with an implicit "raise with maintainer" note. The author's initial GitHub Review Comment version of this item ([review 4722253769](https://github.com/tazama-lf/rule-studio-example/pull/13#pullrequestreview-4722253769)) rephrased it as *"This note is directly for Justus. The `frms-coe-lib` version management for rules needs a closer look here."* — i.e. the author agrees the RC pin is worth revisiting but treats it as out of scope for this PR. No code change. Acceptable — carry forward as a follow-up ticket rather than a merge blocker.

#### Item 6 — `sed` `/g` vs no `/g` mismatch between `deploy.yml` and `deploy-to-uat.yml`

**Status: RESOLVED** (commit `09e15c9`)

UAT line 75 is now `sed -i "s/placeholder/$RULE_ID/g" "$DOCKERFILE"` — matches `deploy.yml`. The two workflows now use identical patch semantics.

#### Summary table

| # | Item | Status |
|---|------|--------|
| 1 | `deploy-to-uat.yml` `npm ci` 401 (BuildKit patch + `.npmrc` copy + `--secret` build) | ✅ Resolved (`09e15c9`) |
| 2 | `deploy-to-uat.yml` `--build-arg GH_TOKEN` leak to image history | ✅ Resolved (`09e15c9`) |
| 3 | Silent default change `RUNNER_LABEL` / `SERVER_IP` in `deploy.yml` | ✅ Resolved (`a5aedc0`) |
| 4 | UAT hardcodes `@psl-copilot` (should use `@$ORG`) | ✅ Resolved (`09e15c9`) |
| 5 | `frms-coe-lib` pinned to `8.2.0-rc.6` (RC) | ➖ Declined — follow-up ticket for maintainer |
| 6 | `sed` `/g` mismatch between prod and UAT | ✅ Resolved (`09e15c9`) |

[↑ Back to top](#pr-review-rse-13--feat-rule-deployment-workflow-updated)

---

### New Issues Found in Updated Commits

I ran the four Section 3.1 hunts (parallel-siblings, type/prop drift, label/boundary drift, guard/scope asymmetry) against the `81b43b3..a5aedc0` delta.

- **Parallel-siblings.** The delta is *itself* the sibling alignment — every step in `deploy-to-uat.yml` that had drifted from `deploy.yml` is now in lockstep (Dockerfile patch, `.npmrc` generation, docker build, `$ORG` substitution, `/g` on `sed`). The two workflows are now materially identical for the clone + patch + build chain. No new asymmetry introduced.
- **Type/prop drift.** N/A — pure workflow YAML, no shared types.
- **Label/boundary drift.** The default `SERVER_IP` in `deploy.yml` changed from `10.10.80.35` back to `10.10.80.18`; the PR body ("Replaced hardcoded deployment server IPs with the configurable `SERVER_IP` variable") is still accurate for both values. No stale user-facing string to update.
- **Guard/scope asymmetry.** New env var `ORG` and new `SERVER_IP` var are wired identically at every consumer in the UAT workflow. Nothing missed.

**CodeRabbit Pass 2** (on `a5aedc0`, [review 4722514944](https://github.com/tazama-lf/rule-studio-example/pull/13#pullrequestreview-4722514944)) produced three nitpicks, all on `deploy-to-uat.yml`, all rated 🔵 Trivial:

| Finding | Severity (CodeRabbit) | My assessment |
|---------|----------------------|---------------|
| Quote `$SERVER_IP`, `$RULE_NAME`, `$ORG` in the `docker run` block (lines 163-192) | 🔵 Trivial | Valid observation but low-value: `RULE_NAME` = `github.event.repository.name` (GitHub sanitises to `[A-Za-z0-9._-]`), `ORG` = `github.repository_owner` (same sanitisation), `SERVER_IP` comes from a workflow variable set by an operator. None can legitimately contain whitespace. **Informational — not new**; the same unquoted pattern exists in `deploy.yml` and was not flagged in the prior round. Author declined ("this is not applicable"). Agree with the author. |
| Lowercase `ORG` for Docker/npm compatibility (`ORG=${ORG,,}`) | 🔵 Trivial | Real concern in theory — Docker image tags and npm scopes require lowercase. However, `github.repository_owner` for the actual consumer orgs (`tazama-lf`, `psl-copilot`) is already lowercase, so no runtime failure. **Informational — not new**; identical pattern in `deploy.yml` line 21 with the same lowercase-org assumption, and was not flagged in the prior round. Acceptable to defer. |
| Consolidate `Update Rule Dependency Name` + `Generate package-lock.json` + `Install Latest Rule Version` into one `npm install --save-exact` step | 🔵 Trivial | The three-step sequence is redundant (the final `Install Latest Rule Version` step deletes + re-installs what the first step just wrote), but the wasted work is a few seconds and the split is arguably easier to reason about in the Actions log. **Informational — pre-existing**; the pattern was already present at HEAD `81b43b3` and not flagged in the prior round. Non-blocking. |

None of the three rises to a new blocking or non-blocking item on top of what was already carried into round 1. Recording them here for the record; no action required for merge.

[↑ Back to top](#pr-review-rse-13--feat-rule-deployment-workflow-updated)

---

### Updated Verdict

**Verdict: Approved**

All three round-1 blocking items are resolved by commits `09e15c9` and `a5aedc0`:

- The UAT workflow is now a faithful mirror of `deploy.yml` for the Dockerfile-patch + `.npmrc` + BuildKit-secret build chain, so `npm ci` will authenticate correctly against the `dev`-branch `rule-executer` (Blocking 1), and `GH_TOKEN` no longer leaks to image history on UAT (Blocking 2).
- `deploy.yml` defaults roll back to the pre-PR `server-18` / `10.10.80.18` targets, preserving behaviour for every rule repo that hasn't opted-in to a new host via `vars.RUNNER_LABEL` / `vars.SERVER_IP` (Blocking 3).

Round-1 Non-blocking item 4 (UAT hardcodes `@psl-copilot`) is also fully resolved. Item 5 (`frms-coe-lib` RC pin) remains declined-and-deferred to the maintainer, which was already acceptable in round 1.

Round-2 CodeRabbit nitpicks (unquoted vars, uppercase-`ORG` risk, redundant npm steps) are Informational, all present in `deploy.yml` since round 1, and the author has stated they are not applicable for this PR. Agree — none block merge.

Recommend merge once a human reviewer approves.

### Blocking

_None._

### Non-blocking follow-ups (post-merge)

1. **`frms-coe-lib` GA bump.** When `8.2.0` GA lands, bump the template so new rule repos inherit a stable version rather than `-rc.6`. Track separately.
2. **Reusable workflow.** `deploy.yml` and `deploy-to-uat.yml` are now materially identical for the clone + patch + npm + build chain. Extracting the shared logic into a composite action or reusable workflow would prevent future drift. Not urgent — track separately.

[↑ Back to top](#pr-review-rse-13--feat-rule-deployment-workflow-updated)

---

### GitHub Review Comment (Follow-up)

````markdown
**Approved (follow-up, HEAD `a5aedc0`)**

All three blocking items from the prior round are resolved:

1. **UAT `npm ci` 401 — fixed** (commit `09e15c9`). `deploy-to-uat.yml` now mirrors `deploy.yml`: Dockerfile patched under `set -euo pipefail` to rewrite the `dev`-branch `RUN --mount=type=secret,id=GH_TOKEN,env=GH_TOKEN npm ci ...` lines into the `required=true GH_TOKEN="$(cat /run/secrets/GH_TOKEN)"` form; `.npmrc` copied into the build context; and the build switched to `docker build --secret id=GH_TOKEN,env=GH_TOKEN -t "$ORG/$RULE_NAME:latest" "rule-executer-$RULE_NAME"`.
2. **UAT `--build-arg GH_TOKEN` leak — fixed** (commit `09e15c9`). Same treatment as (1); token no longer lands in image history.
3. **Silent default host change — fixed** (commit `a5aedc0`). `deploy.yml` defaults are back to `server-18` / `10.10.80.18`, so rule repos without explicit `vars.RUNNER_LABEL` / `vars.SERVER_IP` retain their pre-PR targets. Env variables remain overridable for opting in to other hosts.

Prior non-blocking (4) `@psl-copilot` hardcodes in UAT are also all replaced with `@$ORG`. Prior non-blocking (5) `frms-coe-lib` RC pin remains — carry forward as a follow-up when `8.2.0` GA lands, per the author's note to Justus.

CodeRabbit's Pass-2 nitpicks (unquoted docker-run vars, uppercase-`ORG` risk, redundant npm steps) are Informational and the same patterns exist in `deploy.yml`; the author's *"this is not applicable"* reply is fine for this PR.

### Post-merge follow-ups (non-blocking, track separately)

- Bump `@tazama-lf/frms-coe-lib` to `8.2.0` GA when it lands and update the template pin.
- Extract the now-identical clone + patch + npm + build chain from `deploy.yml` and `deploy-to-uat.yml` into a composite action / reusable workflow to prevent future drift.

Recommend merge once a human reviewer signs off.
````

[↑ Back to top](#pr-review-rse-13--feat-rule-deployment-workflow-updated)
