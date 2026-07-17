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
