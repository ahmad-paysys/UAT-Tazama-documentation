# PR Review: RSDTO #5 — fix: bootstrap repository creation and Docker Git runtime

**Repo:** tazama-lf/rule-studio-devtestops
**Branch:** `fix/bootstrap` → `dev`
**Author:** MuhammadAli-Paysys (Muhammad Ali; original commit by SamitSaleem)
**Date Reviewed:** 2026-07-17
**Label:** none
**Size:** +315 / -165 lines across 9 files
**Commits:** 4 (`30671bb`, `3ddef55`, `0ea107e`, `df06577`)
**State:** OPEN (mergeStateStatus: BLOCKED, mergeable)
**HEAD SHA verified:** `df065776971bd89b5e30544bc674f9dc8430b61e`
**Existing approvals:** none — CodeRabbit auto-review was rate-limited (no findings issued yet)

---

## Table of Contents

- [Overview](#overview)
- [What Changed (Detailed)](#what-changed-detailed)
  - [1. `src/services/github.logic.service.ts` — bootstrap rewrite + populate/promote hardening](#1-srcservicesgithublogicservicets--bootstrap-rewrite--populatepromote-hardening)
  - [2. `src/config.ts`, `src/interfaces/envConfig.interface.ts`, `.env.sample` — env rename](#2-srcconfigts-srcinterfacesenvconfiginterfacets-envsample--env-rename)
  - [3. `Dockerfile` — install git in runner image](#3-dockerfile--install-git-in-runner-image)
  - [4. `README.md` — env-var rename + git prereq](#4-readmemd--env-var-rename--git-prereq)
  - [5. `package.json` / `package-lock.json` — add `simple-git`](#5-packagejson--package-lockjson--add-simple-git)
  - [6. `__tests__/unit/github.logic.service.test.ts` — coverage for new paths](#6-__tests__unitgithublogicservicetestts--coverage-for-new-paths)
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
- [Second Follow-up Review (2026-07-17)](#second-follow-up-review-2026-07-17)
  - [Prior Follow-up Items — Resolution Status](#prior-follow-up-items--resolution-status)
  - [New Issues Found in Updated Commits (Second Follow-up)](#new-issues-found-in-updated-commits-second-follow-up)
  - [Updated Verdict (Second Follow-up)](#updated-verdict-second-follow-up)
  - [GitHub Review Comment (Second Follow-up)](#github-review-comment-second-follow-up)

---

## Overview

`rule-studio-devtestops` is the Fastify service that translates rule-studio UI actions ("create a rule", "populate rule code", "promote to dev") into GitHub API calls that bootstrap and manage a per-rule repository cloned from the `rule-studio-example` template. This PR is a **substantial rewrite** of the bootstrap flow:

- **Before:** `bootstrapHandler` called GitHub's `POST /repos/{owner}/{repo}/generate` (the template-repository generate API), then polled `waitForRepoReady` to check when GitHub finished materialising the copy, then optionally renamed/deleted branches to end up on `GITHUB_INIT_BRANCH`.
- **After:** `bootstrapHandler` creates an empty repo via `POST /orgs/{org}/repos`, then uses the newly-added `simple-git` library to (a) clone the template repo's `GITHUB_BRANCH` into a temp dir with a `x-access-token:` credential-embedded URL, (b) swap the origin to the new empty repo, (c) rename the branch to `initBranch` via `git branch -M`, (d) push. The polling `waitForRepoReady` and the `setDefaultBranch` / `deleteBranchIfExists` helpers are deleted.

Alongside, three cross-cutting fixes land:

1. `GITHUB_DEFAULT_BRANCH` renamed to `GITHUB_BRANCH` (env var, config, DTO, docs, tests).
2. `populateHandler` now auto-encodes rule/test source as base64 when not already base64 (`isBase64Content` + `toGitHubContent` helpers).
3. `Dockerfile` runner stage installs `git` via `apk add --no-cache git` — required because `simple-git` shells out to a `git` binary that was not in the Alpine runner. Without this the new bootstrap flow `spawn git ENOENT`s in production Jenkins deploys.
4. Several GitHub API URL constructions now `encodeURIComponent(branch)` / `encodeURIComponent(sha)` to survive branch names with special characters.

**Base branch:** `dev` — correct for this repo's flow.

**Failing CI checks:** none — CodeQL Analyze (actions and javascript-typescript) both pass on `df06577`. **CodeRabbit's auto-review was rate-limited** and did not run against this HEAD (see CodeRabbit Activity below) — no independent second opinion is available yet.

| File | Nature of Change |
|------|-----------------|
| `src/services/github.logic.service.ts` | Bootstrap rewritten around `simple-git`; base64 auto-encoding for populate; `encodeURIComponent` on branch/sha in URLs; deletes `waitForRepoReady`, `setDefaultBranch`, `deleteBranchIfExists`; separates `ensureBranchFromBase` from the template-copy path. |
| `src/config.ts` | `GITHUB_DEFAULT_BRANCH` → `GITHUB_BRANCH`. |
| `src/interfaces/envConfig.interface.ts` | Same rename. |
| `.env.sample` | Same rename. |
| `Dockerfile` | `RUN apk add --no-cache git` in the `runner` stage. |
| `README.md` | Env-var rename in tables/examples; adds "Git CLI available on `PATH`" prerequisite. |
| `package.json` / `package-lock.json` | Add `simple-git@^3.36.0` (pulls `@kwsites/file-exists`, `@kwsites/promise-deferred`, `@simple-git/args-pathspec`, `@simple-git/argv-parser`). |
| `__tests__/unit/github.logic.service.test.ts` | Extensive additions: base64 encoding coverage, tenant-branch creation, promote source-branch edge cases, repo-exists error path. Replaces old `waitForRepoReady` timeout test. |

[↑ Back to top](#pr-review-rsdto-5--fix-bootstrap-repository-creation-and-docker-git-runtime)

---

## What Changed (Detailed)

### 1. `src/services/github.logic.service.ts` — bootstrap rewrite + populate/promote hardening

**Bootstrap: template-generate → empty-repo-create + git clone/push** (lines 120–158):

```typescript
// Before
const createRes = await fetch(
  `${api}/repos/${configuration.GITHUB_TEMPLATE_OWNER}/${configuration.GITHUB_TEMPLATE_REPO}/generate`,
  { method: 'POST', headers, body: JSON.stringify({ owner: organization, name: repo, private: false, include_all_branches: false }) }
);
await waitForRepoReady(organization, repo, headers);
await ensureBranchFromBase(organization, repo, initBranch, configuration.GITHUB_DEFAULT_BRANCH, headers);
if (initBranch !== configuration.GITHUB_DEFAULT_BRANCH) {
  await setDefaultBranch(organization, repo, initBranch, headers);
  await deleteBranchIfExists(organization, repo, configuration.GITHUB_DEFAULT_BRANCH, headers);
}
await copyTemplateFiles(organization, repo, ruleVersion, initBranch, headers);

// After
const createRes = await fetch(`${api}/orgs/${organization}/repos`, {
  method: 'POST', headers, body: JSON.stringify({ name: repo, private: false }),
});
// ...
const tempDir = await fs.mkdtemp(path.join(os.tmpdir(), 'bootstrap-'));
try {
  const git = simpleGit();
  const templateRepoUrl = `https://x-access-token:${token}@github.com/${configuration.GITHUB_TEMPLATE_OWNER}/${configuration.GITHUB_TEMPLATE_REPO}.git`;
  await git.clone(templateRepoUrl, tempDir, ['--single-branch', '--branch', configuration.GITHUB_BRANCH]);
  const repoGit = simpleGit(tempDir);
  await repoGit.removeRemote('origin');
  const newRepoUrl = `https://x-access-token:${token}@github.com/${organization}/${repo}.git`;
  await repoGit.addRemote('origin', newRepoUrl);
  await repoGit.branch(['-M', initBranch]);
  await repoGit.push(['-u', 'origin', initBranch]);
  await copyTemplateFiles(organization, repo, ruleVersion, initBranch, headers);
} finally {
  await fs.rm(tempDir, { recursive: true, force: true });
}
```

The `waitForRepoReady`, `setDefaultBranch`, and `deleteBranchIfExists` helpers are entirely removed from the file. `ensureBranchFromBase` is retained (still used by `populateHandler` at line 184).

**Populate: base64 auto-encoding** (lines 35–49, 194–195, 204, 221):

```typescript
function isBase64Content(content: string): boolean {
  if (content.trim() === '') return true;
  try {
    return Buffer.from(content, 'base64').toString('base64') === content;
  } catch { return false; }
}

function toGitHubContent(content: string): string {
  return isBase64Content(content) ? content : Buffer.from(content).toString('base64');
}

// in populateHandler:
const encodedRuleCode = toGitHubContent(ruleCode);
const encodedTestCode = toGitHubContent(testCode);
```

Previously the handler passed `ruleCode`/`testCode` straight to the GitHub `PUT /contents/…` API — which requires base64 — and relied on the caller to send pre-encoded content. Now both raw TypeScript and already-encoded base64 are accepted.

**URL encoding of branch/sha in five spots** (lines 361–362, 422, 480, 561, 614): `?ref=${branch}` and `?branch=${branch}` all become `?ref=${encodeURIComponent(branch)}` / `?branch=${encodeURIComponent(branch)}`. Prevents URL parsing errors when branch names contain `/` or `#`.

**Import layout tightened** (lines 1–22): explicit `../schemas/index` / `../interfaces/index` paths; `simpleGit`, `fs`, `os`, `path` added; `setTimeout as delay` removed with the poller.

### 2. `src/config.ts`, `src/interfaces/envConfig.interface.ts`, `.env.sample` — env rename

```diff
-  { name: 'GITHUB_DEFAULT_BRANCH', type: 'string', optional: false },
+  { name: 'GITHUB_BRANCH', type: 'string', optional: false },
```

Same rename in the TypeScript interface. `.env.sample` line 18 similarly renamed. This is a **breaking config change** for any existing deployment: existing `.env`s with `GITHUB_DEFAULT_BRANCH=main` will fail the startup check (`optional: false`) because `GITHUB_BRANCH` is now missing.

### 3. `Dockerfile` — install git in runner image

```diff
 FROM node:20-alpine AS runner

 WORKDIR /app

+RUN apk add --no-cache git
+
 # Copy built files and dependencies
```

Also adds a trailing newline (previously "no newline at end of file"). `simple-git` shells out to the `git` CLI at runtime; without this, `git.clone(...)` throws `spawn git ENOENT` inside the container. Called out in the PR description.

### 4. `README.md` — env-var rename + git prereq

Rename in the Prerequisites, `.env` example, and the "API Service `.env`" table. Adds:

```markdown
- Git CLI available on `PATH`
```

to the Prerequisites list.

### 5. `package.json` / `package-lock.json` — add `simple-git`

```diff
     "fastify": "^5.6.2",
     "node-cron": "^4.2.1",
     "pino-pretty": "^13.0.0",
+    "simple-git": "^3.36.0",
     "zod": "^3.24.0"
```

Runtime dep, not dev — correct because bootstrap runs in production. Transitive deps: `@kwsites/file-exists`, `@kwsites/promise-deferred`, `@simple-git/args-pathspec`, `@simple-git/argv-parser` (all small, MIT-licensed per the lockfile).

### 6. `__tests__/unit/github.logic.service.test.ts` — coverage for new paths

Additions worth calling out:

- `simple-git` is mocked at the module level (line 53–56) so tests don't actually shell out.
- Bootstrap success test: fetch chain replaced (removes the `waitForRepoReady` mock, adds `status: 404` on `repoExists`), asserts the new success message wording.
- New `should handle repo existence check error` test (line 258–274) — GitHub `500` on repo check now propagates a clear error rather than being conflated with a 404.
- New `should encode raw TypeScript before populating files` test (line 292–313) — asserts `Buffer.from(...).toString('base64')` on both rule and test uploads.
- New "should create tenant init branch from template branch before populating files" (line 323–349): three GitHub calls (probe tenant branch → get template branch sha → create tenant ref).
- New "missing template branch" and "tenant branch creation failure" negative-path tests.
- Promote: new "already on tenant source branch" fast-path assertion and "missing source branch" negative-path.

Removes the `should handle repository initialization timeout` test (now unreachable — `waitForRepoReady` is gone).

[↑ Back to top](#pr-review-rsdto-5--fix-bootstrap-repository-creation-and-docker-git-runtime)

---

## Code Quality Analysis

### Strengths

- **Real bug fix.** Installing `git` in the runner image (`Dockerfile:24`) closes the `spawn git ENOENT` failure the PR description names. This is the change that actually made the pipeline work.
- **URL encoding of branch/sha.** Five spots switched to `encodeURIComponent`. Branch names with `/` or `+` will no longer create malformed GitHub URLs.
- **Base64 auto-detection.** `isBase64Content` + `toGitHubContent` lets callers send raw TypeScript OR pre-encoded content without a schema break — a defensible ergonomic win, and both branches have explicit tests.
- **Explicit 404 handling in `repoExists`.** New shape (line 654–672): return `true` on 200, `false` on 404, `throw` on anything else. Previously any non-OK response was treated as "repo doesn't exist" and would enter the create branch even on transient GitHub failures.
- **Try/finally around `fs.rm`.** The clone-and-push flow uses `try { ... } finally { await fs.rm(tempDir, { recursive: true, force: true }); }` — leaves no orphaned temp directories on push failure.
- **Meaningful test expansion.** 54 tests, statement coverage 98.61%, branch coverage 95.48%, function coverage 100% per PR body. Every new negative path has a matching assertion.

### Issues and Observations

#### Issue 1 — GitHub PAT interpolated into the git remote URL is captured in `.git/config` and in `simple-git`'s error output

**Severity: Major (Security)**

Lines 139 and 147:

```typescript
const templateRepoUrl = `https://x-access-token:${token}@github.com/${configuration.GITHUB_TEMPLATE_OWNER}/${configuration.GITHUB_TEMPLATE_REPO}.git`;
await git.clone(templateRepoUrl, tempDir, [...]);
// ...
const newRepoUrl = `https://x-access-token:${token}@github.com/${organization}/${repo}.git`;
await repoGit.addRemote('origin', newRepoUrl);
```

Consequences:

1. `git clone <url>` writes the URL — including the credential — into `<tempDir>/.git/config` under `[remote "origin"] url = …`. Even though the tempDir is `fs.rm`'d in the `finally`, if the process crashes between clone and rm (SIGKILL, OOM), the credential lives on disk. On a shared runner or if the OS tempdir survives container restart, that's a leaked token.
2. `simple-git` propagates command output verbatim on failure. If the `push` step 401s, the error message can echo the URL including the credential and land in `loggerService.error(message)` (line 101) — which typically pipes to STDOUT/journald. Any log shipper (Elastic, Splunk, CloudWatch) then persists the token.
3. If the process is running with `HISTFILE`-visible spawns (unlikely here since simple-git uses `spawn` not `exec` with a shell, but worth confirming), the URL could surface in process listings.

**Fix options** (in order of preference):

- Use a git credential helper or the `GIT_ASKPASS` env override so the token never reaches the URL. Example:
  ```typescript
  await git.env({ GIT_ASKPASS: 'echo', GIT_TERMINAL_PROMPT: '0' })
    .env('GITHUB_TOKEN', token)
    .clone(`https://github.com/${owner}/${repo}.git`, tempDir, ['--config', `http.extraheader=Authorization: bearer ${token}`, ...]);
  ```
  (`http.extraheader` still lands in `.git/config` — use `-c` on the command line so it does not persist.) The most robust approach is a short-lived credential helper that reads from an env var and does not persist.
- Ensure `handleError` scrubs tokens from any message before logging: `message.replace(/x-access-token:[^@]+@/g, 'x-access-token:***@')`.
- After push and before `fs.rm`, explicitly `await repoGit.remote(['set-url', 'origin', urlWithoutToken])` to overwrite the credential in `.git/config` — belt-and-braces.

At minimum, add the log-scrub in `handleError` before merge; the credential-helper migration can be a follow-up if time-boxed.

#### Issue 2 — Success message for the "created" branch says "Default branch" but no default branch is actually set

**Severity: Minor (Bug / User Communication)**

Line 163:

```typescript
message: exists
  ? `Updated version to ${ruleVersion} in ${organization}/${repo} on Default branch ${initBranch}`
  : `Created ${organization}/${repo} v${ruleVersion} on Default branch ${initBranch}`,
```

The old code path called `setDefaultBranch(organization, repo, initBranch, headers)` (deleted in this PR). The new code path only does `git branch -M initBranch && git push -u origin initBranch` — pushing the branch does NOT set it as the repository's *default branch* on GitHub. The default remains whatever GitHub assigned when `POST /orgs/{org}/repos` created the empty repo (typically `main`). So the message reads "Created … on Default branch staging" even though GitHub's UI still shows `main` as the default.

Two possible fixes:

- Restore `setDefaultBranch(organization, repo, initBranch, headers)` after the push (in the same `try` block); or
- Reword the message to `on branch ${initBranch}` (accurate, doesn't overclaim).

If setting the default was the intent (and the old code did do it), option (a) preserves behaviour. Otherwise (b).

#### Issue 3 — When the repo already exists, template code is never re-copied; the "Updated version" message is misleading

**Severity: Major (Bug)**

Lines 120–158 wrap the entire clone-and-copy chain in the `else` (repo does not exist) branch. When the repo *already exists*, only the `loggerService.log(...)` runs, and then the success message says:

```
`Updated version to ${ruleVersion} in ${organization}/${repo} on Default branch ${initBranch}`
```

Nothing was actually updated — no `copyTemplateFiles`, no `ensureBranchFromBase`, no rewrite of `package.json`. The old flow always ran `waitForRepoReady` → `ensureBranchFromBase` → optional `setDefaultBranch`/delete → `copyTemplateFiles`, both for existing and freshly-generated repos. The new code drops the whole update-when-exists path.

If the intent is that `bootstrap` is only for *new* repos and re-bootstraps are silently no-ops, the message should reflect that (`Repository already exists; no changes made`). If re-bootstraps should re-copy the template (which the old code did), move the `ensureBranchFromBase` + `copyTemplateFiles` calls outside the `else`. Per the PR description ("Refactored the GitHub bootstrap workflow to: … Update the `package.json` file on the initialized branch."), the intent seems to be that `copyTemplateFiles` DOES run — but currently it only runs in the freshly-created branch.

#### Issue 4 — `simple-git` mock in the test suite hides shell dependencies; no test covers the actual `git` binary requirement

**Severity: Minor (Test Coverage)**

Lines 53–56 of the test file:

```typescript
jest.mock('simple-git', () => ({
  __esModule: true,
  default: jest.fn(),
}));
```

Then in `beforeEach` (line 92–98):

```typescript
(simpleGit as jest.Mock).mockReturnValue({
  clone: jest.fn().mockResolvedValue(undefined),
  removeRemote: jest.fn().mockResolvedValue(undefined),
  addRemote: jest.fn().mockResolvedValue(undefined),
  branch: jest.fn().mockResolvedValue(undefined),
  push: jest.fn().mockResolvedValue(undefined),
});
```

Any of these can throw at runtime in a variety of shapes (auth failure, network failure, non-fast-forward, permission denied); none are exercised. Add at least:

- A test that `git.clone` rejects with a 401-shaped `simpleGit.GitError` → the handler returns 500 with a scrubbed message.
- A test that `git.push` rejects (non-fast-forward from a previous partial run in the same repo) → the tempDir is still cleaned up.

Combined with Issue 1, a test that asserts `handleError` does NOT log the token substring when a `simple-git` error contains the URL would be worth adding.

#### Issue 5 — `isBase64Content` false-positive: any input whose bytes happen to round-trip through base64 will be treated as pre-encoded

**Severity: Minor (Data Integrity)**

Lines 35–45:

```typescript
function isBase64Content(content: string): boolean {
  if (content.trim() === '') return true;
  try {
    return Buffer.from(content, 'base64').toString('base64') === content;
  } catch { return false; }
}
```

Any string that happens to contain only base64-alphabet characters (`A-Za-z0-9+/=`) will be treated as pre-encoded. Some short, valid TypeScript literally satisfies this — `const x=42` (13 chars, all base64-alphabet) does NOT (contains `=` in `=42`); but `"abcd"` (4 chars, all base64-alphabet, correct-length) does. So a rule template file that happens to contain only base64-alphabet characters (rare for TypeScript, but possible in a minimised or otherwise contrived case) would be posted verbatim, and GitHub would reject or store garbage. Also: an empty string is treated as "base64", which is defensible (uploads an empty file) but silently.

A stricter approach: require the caller to explicitly indicate encoding (e.g. add `encoding: 'utf8' | 'base64'` to the request body), or use a regex that also requires the length to be a multiple of 4 AND the specific mix of characters that unambiguously identifies base64 (all base64 outputs satisfy those constraints, but almost no natural TypeScript does). For now, add a test with a short TypeScript sample that happens to be base64-clean, to prove the assumption holds for realistic inputs.

#### Issue 6 — `promoteHandler`'s "already on tenant source branch" fast-path skips baseSha lookup even when the branch actually differs from the source at the tip

**Severity: Informational (Bug Risk)**

Lines 254–260:

```typescript
if (branchName === sourceBranch) {
  reply.status(200).send({ success: true, message: `Branch ${branchName} is already the tenant source branch` });
  return;
}
```

This is a semantic fast-path: "if you're promoting *to* the source branch, do nothing". Reasonable — but it doesn't verify the source branch *exists* first. In an ephemeral test setup where the source branch was deleted between init and promote, this silently returns 200 for a branch that isn't there. Low-impact; note only.

#### Issue 7 — Some existing helpers stayed URL-unencoded

**Severity: Informational (Consistency)**

Not everything moved to `encodeURIComponent`. For example, `getRepoName(ruleId)` (line 60) is `ruleId` verbatim — a `ruleId` containing `/` or `..` would land in URL paths like `${api}/repos/${organization}/${ruleId}`. `ruleId` upstream should be a small numeric string, but a validator would be a good defense-in-depth. Similarly the `sha` on line 262 (`getBranchSha` return) is embedded in `${api}/repos/${organization}/${repo}/commits/${baseSha}` without encoding — SHAs are hex, so this is safe, but the pattern is worth aligning.

[↑ Back to top](#pr-review-rsdto-5--fix-bootstrap-repository-creation-and-docker-git-runtime)

---

## Security Assessment

| Concern | Assessment |
|---------|-----------|
| Token in git remote URL (`.git/config` + error logs) | **New exposure — see Issue 1.** The old code path used the GitHub REST API with `Authorization: token …` headers, which never wrote credentials to disk. The new code path uses `simple-git` with a URL that embeds the token in the string; `git clone` persists this to `<tempDir>/.git/config`, and `simple-git` error messages can echo it into `loggerService.error`. Fix required before merge. |
| Unvalidated input in URL paths | `ruleId` and `initBranch` come from the request body/headers and land in `${api}/repos/{org}/{ruleId}` paths and (via `getRepoName`) as the target repo name. Not command-injection-exploitable via the GitHub REST client, but a repo name of `../` would be embarrassing. `initBranch` is now URL-encoded in five spots (good), but `getRepoName` and `ruleId` still are not. Add a `zod` validator (`.regex(/^[a-zA-Z0-9-_]+$/)`) on the request schemas as defense-in-depth. |
| Auth guards on new endpoints | No new endpoints. Existing handlers still rely on `tenantToken` and `organizationName` from the request (via `ITenantRequest`); those are set by whatever middleware/plugin decorates the request. Not touched by this PR. |
| CI security scanners | CodeQL Analyze (actions) SUCCESS; CodeQL Analyze (javascript-typescript) SUCCESS on `df06577`. No njsscan configured on this repo (unlike rule-studio). |
| `simple-git` supply chain | Adds `simple-git@^3.36.0` and four transitive deps (`@kwsites/file-exists`, `@kwsites/promise-deferred`, `@simple-git/args-pathspec`, `@simple-git/argv-parser`). All MIT, all published by the git-js author. Reasonable choice for a Node git library. |

Net: one Major security regression (Issue 1) added by this PR that was not present in the pre-PR REST-only flow. Everything else neutral or slightly better.

[↑ Back to top](#pr-review-rsdto-5--fix-bootstrap-repository-creation-and-docker-git-runtime)

---

## Test Coverage

**What is tested.**

- Bootstrap: success (new repo), success (repo already exists), repo creation failure, package fetch failure, package update failure, GitHub 500 on the repo-existence check.
- Populate: base64 content pass-through, raw TypeScript encoding, rule update error, template-branch existence check, tenant-branch creation, tenant-branch creation failure, missing template branch.
- Promote: source branch equal to target (fast-path), missing source branch, create new branch from default, updated success message text.
- Fetch latest test report: existing coverage retained.

Reported: 54 tests, statement coverage 98.61%, branch coverage 95.48%, function coverage 100%.

**What is not tested.**

1. **The Major security issue (Issue 1) has no matching test.** No assertion that `handleError`'s logged message does not contain the token substring when a `simple-git` failure includes the credentialed URL. Add before merge if Issue 1 stands.
2. **`simple-git` failure paths.** Every `simpleGit` method is mocked to `resolve(undefined)`. `git.clone` throwing (network, auth, missing branch), `git.push` throwing (non-fast-forward, protection rule), and the `try/finally` cleanup semantics on each failure are unexercised.
3. **`isBase64Content` boundary cases.** No test with a short valid TypeScript that happens to be base64-alphabet-clean (see Issue 5). A regression test would prove the assumption safely.
4. **Existing-repo re-bootstrap.** Test at line 165 asserts the "already exists" branch returns 200 with the "Updated version" message — but does not assert `copyTemplateFiles` was called (it isn't — see Issue 3). A negative assertion (`expect(fetch).not.toHaveBeenCalledWith(<contents URL>)`) would document the intent.
5. **Default branch is actually set.** No assertion about GitHub's default_branch after bootstrap (see Issue 2).
6. **Coverage screenshot / CI evidence.** PR body reports numbers; no CI artifact link. Acceptable if the numbers hold in CI.

**PR checklist state (from body).** Marked: Locally ✓, Development Environment ✓, Husky ✓, Unit tests passing ✓, Documentation done ✓. "Not needed" left unchecked.

[↑ Back to top](#pr-review-rsdto-5--fix-bootstrap-repository-creation-and-docker-git-runtime)

---

## CodeRabbit Activity

CodeRabbit's automatic review was **rate-limited on this PR** — the summary explicitly says:

> Review limit reached
> @MuhammadAli-Paysys, you've reached your PR review limit, so we couldn't start this review.
> Next review available in: 59 minutes

CodeRabbit therefore produced **no findings** against `df06577`. This means the review file above is the sole line-by-line reading of the diff. Two consequences:

1. Do not treat CodeRabbit's absence as approval — nothing was checked.
2. Consider re-triggering CodeRabbit with `@coderabbitai review` once the rate limit clears, so a second machine reviewer looks at the change before merge (especially useful for confirming Issue 1's token-in-URL risk).

[↑ Back to top](#pr-review-rsdto-5--fix-bootstrap-repository-creation-and-docker-git-runtime)

---

## Summary and Verdict

**Verdict: Changes Requested**

The PR delivers three real fixes — installing git in the runner image (which resolves the actual `spawn git ENOENT` production incident), auto-encoding rule content as base64, and URL-encoding branch/sha in GitHub URLs. The rewrite from GitHub's `generate` API to `simple-git`-based clone-and-push is a defensible change with a clearer failure mode than polling `waitForRepoReady` — and the test suite has grown to match. But two items really need addressing before merge: the token now travels through the git remote URL and lands both in `.git/config` on disk and in `simple-git` error output (a security regression against the pre-PR REST-only flow), and the "repo already exists" branch of the new bootstrap does no work but returns a message that claims an update happened. Both are fixable in-scope; the token exposure can be scoped down to a log-scrubber + explicit remote-url cleanup even without a full credential-helper migration.

### Blocking

1. **PAT interpolated into git remote URL persists on disk and can leak into logs** — Issue 1. Move the credential out of the URL (via `http.extraheader` on the command line, or `GIT_ASKPASS`), scrub any URL that appears in error paths before it reaches `loggerService.error`, and null out `origin` before the `try/finally` cleans up.
2. **Existing-repo branch of `bootstrapHandler` does no template copy but claims "Updated version"** — Issue 3. Either move `ensureBranchFromBase` + `copyTemplateFiles` outside the `else` (matching the old behaviour and the PR description), or change the message to "Repository already exists; no changes made".

### Non-blocking but recommended

3. **Success message says "Default branch" but no default branch is set** — Issue 2. Restore `setDefaultBranch` after push, or reword.
4. **Cover `simple-git` failure paths** — Issue 4. At least: clone auth failure and push non-fast-forward.
5. **Tighten `isBase64Content` or add an explicit `encoding` field** — Issue 5. Add a regression test that a short base64-alphabet-clean TypeScript sample still round-trips correctly.
6. **Re-run CodeRabbit once the rate limit clears** so at least one machine reviewer has scanned the diff before merge.

[↑ Back to top](#pr-review-rsdto-5--fix-bootstrap-repository-creation-and-docker-git-runtime)

---

## GitHub Review Comment

`````markdown
**Changes Requested**

Three real fixes in here — installing `git` in the runner image (which actually resolves the `spawn git ENOENT` incident), auto-encoding rule/test content as base64, and URL-encoding branch/sha in GitHub API paths. The switch from GitHub's `generate` API to `simple-git`-based clone-and-push has a cleaner failure model than the old `waitForRepoReady` poller, and the test suite grew to cover the new negative paths. Two items need addressing before merge.

---

### Blocking

**1. GitHub PAT interpolated into the git remote URL persists on disk and can leak into logs.**

In [`github.logic.service.ts:139`](src/services/github.logic.service.ts#L139) and [`:147`](src/services/github.logic.service.ts#L147):

```typescript
const templateRepoUrl = `https://x-access-token:${token}@github.com/${configuration.GITHUB_TEMPLATE_OWNER}/${configuration.GITHUB_TEMPLATE_REPO}.git`;
await git.clone(templateRepoUrl, tempDir, [...]);
// ...
const newRepoUrl = `https://x-access-token:${token}@github.com/${organization}/${repo}.git`;
await repoGit.addRemote('origin', newRepoUrl);
```

Two exposures the pre-PR REST flow didn't have:

1. `git clone` writes the URL — token and all — to `<tempDir>/.git/config` under `[remote "origin"] url = …`. The `finally` `fs.rm` cleans it up on the happy path, but if the process is SIGKILL'd (OOM, container restart) between clone and rm, the credential survives on disk.
2. `simple-git` echoes command output verbatim into thrown errors. A push failure whose error message contains `https://x-access-token:<token>@…` then lands in [`handleError`](src/services/github.logic.service.ts#L99-L103) → `loggerService.error(message)` → your log shipper.

Fix (either or both):

- Move the credential out of the URL. Simplest: pass it as an HTTP header on the command line, which does NOT persist to `.git/config`:
  ```typescript
  await git
    .env('GIT_TERMINAL_PROMPT', '0')
    .clone(`https://github.com/${owner}/${repo}.git`, tempDir, [
      '-c', `http.extraheader=Authorization: bearer ${token}`,
      '--single-branch', '--branch', configuration.GITHUB_BRANCH,
    ]);
  ```
  (Same treatment for `push`.)
- Scrub the URL from any error message before logging:
  ```typescript
  const scrub = (s: string) => s.replace(/https:\/\/x-access-token:[^@\s]+@/g, 'https://x-access-token:***@');
  const handleError = (error, reply) => {
    const message = scrub(error instanceof Error ? error.message : String(error));
    loggerService.error(message);
    reply.status(500).send({ success: false, message });
  };
  ```

Add a test that a mocked `simple-git` failure containing the credentialed URL is not echoed verbatim by `handleError`.

**2. The "repo already exists" branch of `bootstrapHandler` does no work but claims an update happened.**

Lines 120–158 gate all the new clone-and-copy work behind `else` (repo does NOT exist). When the repo already exists, only `loggerService.log(...)` runs — no `ensureBranchFromBase`, no `copyTemplateFiles`, no `package.json` update. Then line 162 says:

```typescript
message: exists
  ? `Updated version to ${ruleVersion} in ${organization}/${repo} on Default branch ${initBranch}`
  : ...
```

The old flow ran `waitForRepoReady` → `ensureBranchFromBase` → optional `setDefaultBranch`/delete → `copyTemplateFiles` for **both** cases. The PR description ("Update the `package.json` file on the initialized branch") reads like `copyTemplateFiles` should still run on re-bootstrap. Either:

- Move `ensureBranchFromBase(...)` + `copyTemplateFiles(...)` outside the `else` (matching the previous behaviour), or
- Reword to `Repository already exists; no changes made` so the message reflects reality.

Whichever you pick, add an assertion in [`__tests__/unit/github.logic.service.test.ts`](__tests__/unit/github.logic.service.test.ts) that documents the intent (`expect(fetch).toHaveBeenCalledWith(<contents URL>)` or `.not.toHaveBeenCalledWith(...)`).

---

### Non-blocking (please address in this PR if possible)

**3. "Created … on Default branch" message overclaims — no default branch is actually set.**

`setDefaultBranch` was deleted with this PR. `git push -u origin initBranch` sets upstream tracking but does NOT change GitHub's `default_branch`. Either add back a `PATCH /repos/{org}/{repo}` with `{ default_branch: initBranch }` after push, or reword the message to `on branch ${initBranch}`.

**4. Cover `simple-git` failure paths.** All five `simpleGit` methods are mocked to resolve `undefined`. Add at minimum a clone-auth-failure test and a push-non-fast-forward test that assert `fs.rm(tempDir)` still runs and no token leaks into the reply body.
`````

[↑ Back to top](#pr-review-rsdto-5--fix-bootstrap-repository-creation-and-docker-git-runtime)

---

---

---

## Follow-up Review (2026-07-17)

**Reviewed commit:** `27a6c282be493600a77daeea396c74e84b2f716f` — *"refactor: updated readme"* (2026-07-17)
**Reviewed against:** CHANGES_REQUESTED (2026-07-17T11:51Z, review id `4722373908`) by `ahmad-paysys`
**Delta reviewed:** `df06577..27a6c28` — 3 files, +138 / -25 (`README.md`, `__tests__/unit/github.logic.service.test.ts`, `src/services/github.logic.service.ts`)
**Commits in delta:** `32cc302` (*fix: token leak fixed*), `295f046` (*fix: bootstrap fixed*), `27a6c28` (*refactor: updated readme*)
**Developer response:** No issue-comment reply from the author; response is code-only across the three commits above, plus an inline reply on `github.logic.service.ts:48` — see [comment 3603622105](https://github.com/tazama-lf/rule-studio-devtestops/pull/5#discussion_r3603622105): *"saves code as base 64 so no need for this as we need to save raw content"*. CodeRabbit ran a full pass in the interval (review `4723133452`, 2026-07-17T13:35Z) and posted 2 new actionable findings — see [CodeRabbit Activity](#coderabbit-activity) additions below.
**Note on the standing approval:** an `APPROVED` review from `ahmad-paysys` was recorded at 2026-07-17T14:23Z (review id `4723508886`). The user has stated this was posted by mistake and does not reflect a genuine sign-off; this follow-up round supersedes it.

[↑ Back to top](#pr-review-rsdto-5--fix-bootstrap-repository-creation-and-docker-git-runtime)

---

### Changes Requested — Resolution Status

#### Item 1 — PAT interpolated into git remote URL persists on disk and can leak into logs

**Status: PARTIALLY RESOLVED**

Two exposures were flagged: (a) `git clone` writes the token-carrying URL to `<tempDir>/.git/config`, surviving on disk if the process is SIGKILL'd between clone and `fs.rm`; (b) `simple-git` echoes the URL verbatim into thrown errors, which `handleError` then logs.

**(b) log-leak vector — RESOLVED.** [`github.logic.service.ts:99-101`](src/services/github.logic.service.ts#L99) adds:

```typescript
const scrubToken = (s: string): string =>
  // eslint-disable-next-line require-unicode-regexp -- 'v' flag requires ES2024 target
  s.replace(/https:\/\/x-access-token:[^@\s]+@/g, 'https://x-access-token:***@');
```

Wired into `handleError` at [line 107-109](src/services/github.logic.service.ts#L107):

```typescript
const rawMessage = error instanceof Error ? error.message : String(error);
const message = scrubToken(rawMessage);
loggerService.error(message);
```

Tests at [`github.logic.service.test.ts:349-421`](__tests__/unit/github.logic.service.test.ts#L349) assert both the clone-failure and push-failure paths scrub the token from `reply.send(...)`. Good coverage.

**(a) disk-persistence vector — NOT RESOLVED.** The credential is still embedded in the URL passed to `simple-git`, via [`getAuthenticatedGitHubUrl`](src/services/github.logic.service.ts#L103):

```typescript
const getAuthenticatedGitHubUrl = (owner: string, repo: string, token: string): string =>
  `https://x-access-token:${encodeURIComponent(token)}@github.com/${owner}/${repo}.git`;
```

`git clone` writes the fetch URL — token and all — to the cloned repo's `.git/config` under `[remote "origin"] url = https://x-access-token:<token>@github.com/…`. The `finally { fs.rm(tempDir, ...) }` at [line 172](src/services/github.logic.service.ts#L172) cleans it up on the happy path, and the two new tests confirm cleanup on the sad path. But `fs.rm` is not a durable guarantee: if the container is SIGKILL'd or the node OOMs after clone but before `rm`, the token stays on disk under whatever `/tmp` is mounted from (in Docker/Jenkins, this may or may not be tmpfs).

The `encodeURIComponent(token)` addition doesn't change persistence — it only makes the URL syntactically safe when the token contains reserved characters.

The prior-round suggestion was to pass the credential as an `http.extraheader` on the command line instead of interpolating it into the URL. That variant does **not** persist to `.git/config`, closes both exposures at once, and is a one-line change per call site:

```typescript
await git
  .env('GIT_TERMINAL_PROMPT', '0')
  .clone(`https://github.com/${owner}/${repo}.git`, tempDir, [
    '-c', `http.extraheader=Authorization: bearer ${token}`,
    '--single-branch', '--branch', configuration.GITHUB_BRANCH,
  ]);
```

This is a **blocking** item in the follow-up verdict — the log-leak fix is welcome but the disk-persistence exposure was the harder half and remains.

#### Item 2 — "repo already exists" branch does no work but claims an update happened

**Status: RESOLVED**

`copyTemplateFiles(...)` is now called unconditionally, outside the `else` block, at [`github.logic.service.ts:173`](src/services/github.logic.service.ts#L173):

```typescript
    }

    await copyTemplateFiles(organization, repo, ruleVersion, initBranch, headers);

    reply.status(200).send({
      success: true,
      message: exists
        ? `Updated version to ${ruleVersion} in ${organization}/${repo} on branch ${initBranch}`
        : `Created ${organization}/${repo} v${ruleVersion} on branch ${initBranch}`,
    });
```

Both the `exists=true` and `exists=false` branches now run `copyTemplateFiles`, matching the pre-PR semantics. The `Updated version to X` message is no longer an overclaim.

The updated "should successfully update version" test at [`github.logic.service.test.ts:125-146`](__tests__/unit/github.logic.service.test.ts#L125) now asserts both the reply body (`Updated version to 1.0.0 in test-org/123 on branch main`) and the fetch call sequence (`mockPackageGetResponse` → `mockPackagePutResponse`), locking the new behaviour in.

#### Item 3 — "Created … on Default branch" message overclaims

**Status: RESOLVED**

Two remediations, either alone sufficient:

- The success message now says `on branch ${initBranch}` (not `on Default branch`) at [`github.logic.service.ts:177-178`](src/services/github.logic.service.ts#L177).
- A new `setDefaultBranch` helper (deleted in the initial round) was re-added at [`github.logic.service.ts:691-708`](src/services/github.logic.service.ts#L691) and called after push at [line 168](src/services/github.logic.service.ts#L168):

```typescript
async function setDefaultBranch(
  organization: string,
  repo: string,
  branch: string,
  headers: Record<string, string>
): Promise<void> {
  const res = await fetch(`${configuration.GITHUB_API_URL}/repos/${organization}/${repo}`, {
    method: 'PATCH',
    headers,
    body: JSON.stringify({ default_branch: branch }),
  });

  if (!res.ok) {
    throw new Error(`Failed to set default branch: ${await res.text()}`);
  }

  loggerService.log(`Set default branch to ${branch} for ${organization}/${repo}`);
}
```

So the message is now accurate *and* the default branch is genuinely being set. The existing bootstrap tests were updated to mock the additional `PATCH` call at [`github.logic.service.test.ts:177, 247, 279, 319`](__tests__/unit/github.logic.service.test.ts#L177). The README description of `GITHUB_BRANCH` was also corrected in commit `27a6c28` from "Default branch for new repos" to "Template source branch to copy from", so the docs match the code semantics.

#### Item 4 — Cover `simple-git` failure paths

**Status: RESOLVED**

Two new tests at [`github.logic.service.test.ts:348-421`](__tests__/unit/github.logic.service.test.ts#L348):

- `should clean up temp dir and scrub token on clone failure` — mocks `simpleGit().clone` to reject with an error message that embeds a fake credentialed URL, asserts `fs.rm` runs, asserts the reply status is 500, and asserts the sent message does **not** contain the token but **does** contain `***`.
- `should clean up temp dir and scrub token on push failure` — same shape but mocks `raw` (see New Issue 3 below on why `push` was replaced with `raw`) to reject with a non-fast-forward-style error.

Both cover the interaction that Item 1(b) hardened. The mocks include the new `env` chain method (`env: jest.fn().mockReturnThis()`) and `raw` method that the production code now uses. Good.

---

| # | Item | Status |
|---|------|--------|
| 1 | PAT persistence in `.git/config` and error logs | ⚠️ Partial — log-leak vector fixed; disk-persistence vector remains |
| 2 | `exists=true` branch does no work but claims update | ✅ Resolved |
| 3 | "Default branch" overclaim | ✅ Resolved |
| 4 | `simple-git` failure-path tests | ✅ Resolved |

[↑ Back to top](#pr-review-rsdto-5--fix-bootstrap-repository-creation-and-docker-git-runtime)

---

### New Issues Found in Updated Commits

#### Issue 5 — Bootstrap is not atomic: a failed initialization leaves a phantom empty repo that poisons retries

**Severity: Major (Data Integrity)**

This was also flagged by CodeRabbit in the interval — see [outside-diff comment on lines 128-174](https://github.com/tazama-lf/rule-studio-devtestops/pull/5#pullrequestreview-4723133452). I concur.

The flow at [`github.logic.service.ts:127-172`](src/services/github.logic.service.ts#L127) is:

```
1. repoExists(...)               → exists === false
2. POST /orgs/{org}/repos        → creates empty repo
3. simpleGit.clone(template)     → can fail (network, auth, template deleted)
4. addRemote / branch -M         → cheap, unlikely to fail
5. simpleGit.raw(push -u)        → can fail (non-fast-forward, network, quota)
6. setDefaultBranch (PATCH)      → can fail (permissions, race)
7. fs.rm(tempDir)                → cleanup in finally
```

If any of steps 3–6 throws, the finally-block cleans up the temp dir but the **empty repository from step 2 stays on GitHub**. On the next bootstrap for the same rule, `repoExists` returns `true`, the entire `else` block is skipped, and control falls through to `await copyTemplateFiles(organization, repo, ruleVersion, initBranch, headers)` — which tries to `GET .../contents/package.json?ref=${initBranch}` on a repo that has no `initBranch` and no `package.json`. That fetch 404s and the retry fails with an opaque "Package fetch failed"–style error, permanently, until the empty repo is deleted by hand.

Fix (either approach):

- **Compensating delete on failure.** Track a `repoCreatedByThisAttempt` flag; in a catch block wrapping steps 3–6, `DELETE /repos/{org}/{repo}` before re-throwing. Add a test that asserts the delete runs on clone failure. Handles the common case of transient network/auth issues.
- **Idempotent recovery.** When `exists === true`, probe the repo state (does `initBranch` exist? does `package.json` exist on that branch?) and re-run the clone/push if either is missing. More robust against SIGKILL'd bootstraps but strictly more code.

Either variant is fine; the current code makes the failure mode indistinguishable from a stuck deployment and requires manual GitHub-side cleanup.

#### Issue 6 — `isBase64Content` mis-classifies raw source that happens to be valid base64

**Severity: Major (Data Integrity)**

CodeRabbit flagged this at [`github.logic.service.ts:48`](https://github.com/tazama-lf/rule-studio-devtestops/pull/5#discussion_r3603542240) and the author replied that raw content is the intended input:

> saves code as base 64 so no need for this as we need to save raw content — @MuhammadAli-Paysys

That reply confirms the concern rather than defusing it. The current helper at [lines 35-48](src/services/github.logic.service.ts#L35):

```typescript
function isBase64Content(content: string): boolean {
  if (content.trim() === '') {
    return true;
  }
  try {
    return Buffer.from(content, 'base64').toString('base64') === content;
  } catch {
    return false;
  }
}

function toGitHubContent(content: string): string {
  return isBase64Content(content) ? content : Buffer.from(content).toString('base64');
}
```

Any raw source whose bytes happen to round-trip through base64 decode/encode gets treated as "already encoded" and passed through unchanged. That includes:

- `"test"` — round-trips to `"test"` and decodes to bytes `[0xB5, 0xEB, 0x2D]`. If a rule file's entire body is the literal word `test` (e.g. a stub, a scratch file), GitHub stores 3 unrelated bytes.
- `"true"`, `"false"`, `"null"`, single-word identifiers of length 4 / 8 / any multiple of 4 whose characters all fall in `[A-Za-z0-9+/=]`.
- Longer runs are more likely to include characters outside the base64 alphabet and safely fall through — but the failure mode is silent and payload-dependent.

Missed by prior-round review — not caught by any Section 3.1 hunt (it's a content-classification bug, not shared-value / type / boundary / guard drift). This is a new hunt class: **"heuristic encoding sniff on user-controlled input."**

Fix: drop the sniff and always encode, as CodeRabbit suggested and the author's reply concedes ("we need to save raw content"):

```typescript
function toGitHubContent(content: string): string {
  return Buffer.from(content, 'utf8').toString('base64');
}
```

Then delete `isBase64Content`. If any caller was intentionally passing pre-encoded content, that caller needs to decode first — but grep shows both call sites at [lines 211-212](src/services/github.logic.service.ts#L211) receive `ruleCode` / `testCode` sourced from the request body, which per the author's reply is raw source.

Add a regression test with `content: "test"` and assert the stored value is `Buffer.from("test").toString("base64")` (`"dGVzdA=="`), not `"test"`.

#### Issue 7 — Undocumented switch from `simpleGit.push()` to `simpleGit.raw(['push', ...])`

**Severity: Minor (Maintainability)**

At [`github.logic.service.ts:167`](src/services/github.logic.service.ts#L167) the code went from:

```typescript
await repoGit.push(['-u', 'origin', initBranch]);
```

to:

```typescript
await repoGit.raw(['push', '-u', 'origin', initBranch]);
```

`.raw(...)` bypasses simple-git's typed-argument handling and its structured push-response parsing, and any push failure now surfaces as raw `git` stderr rather than simple-git's structured error class. There's no commit message or code comment explaining the switch. If the reason is that `.push()` was misclassifying a specific failure or emitting a specific unwanted log line, capture that in a comment so a future reader doesn't "clean it up" back to `.push()` and reintroduce whatever was being avoided.

Non-blocking; a one-line comment above the call is sufficient.

#### Issue 8 — CodeRabbit's `Docstring Coverage` pre-merge check is failing at 0%

**Severity: Informational (Test Coverage)**

CodeRabbit's pre-merge checks report `Docstring Coverage` at 0.00% (threshold 80%). None of the new functions (`scrubToken`, `getAuthenticatedGitHubUrl`, `setDefaultBranch`) carry docstrings, and pre-existing functions in the file don't either. Whether this is a hard blocker depends on repo policy — it's not blocking CI (statusCheckRollup shows all required checks green), and no reviewer has cited it. Flagging so the author is aware and can either add JSDoc to the newly-introduced helpers or acknowledge that the threshold is not enforced.

[↑ Back to top](#pr-review-rsdto-5--fix-bootstrap-repository-creation-and-docker-git-runtime)

---

### Updated Verdict

**Verdict: Changes Requested**

The three prior-round remediation commits (`32cc302`, `295f046`, `27a6c28`) address Items 2, 3, and 4 cleanly, and half of Item 1 — the log-leak vector is closed with a good regex + tests. What's left is: (a) the credential still lives on disk in `.git/config` inside the temp dir, protected only by a `finally { fs.rm }` that doesn't survive SIGKILL; and (b) two new blocking findings — non-atomic bootstrap that poisons retries on any mid-flow failure, and the base64 detection heuristic that silently corrupts short raw source. Items (b) both landed in CodeRabbit's follow-up pass and are corroborated here.

The base branch (`dev`) and CI status are still correct. The standing `APPROVED` review from this reviewer was posted by mistake per the user and does not reflect an approval on the merits — this follow-up supersedes it.

#### Blocking

1. **Move the PAT out of the git URL (Item 1a).** Use `http.extraheader=Authorization: bearer ${token}` on the `git -c` command-line instead of interpolating the token into the clone/remote URL. Closes the `.git/config` disk-persistence vector that `fs.rm` alone can't guarantee.
2. **Make bootstrap atomic on failure (Issue 5).** After creating the empty repo, wrap steps 3–6 in try/catch that deletes the freshly-created repo on any error before re-throwing. Add a test that asserts the delete runs on clone failure.
3. **Drop `isBase64Content` (Issue 6).** Always base64-encode raw content in `toGitHubContent`. Add a regression test with `content: "test"`.

#### Non-blocking but recommended

4. **Comment the `.push()` → `.raw(['push', ...])` switch (Issue 7).** One line explaining what the switch is avoiding, so a future reader doesn't revert it.
5. **Docstring-coverage pre-merge warning (Issue 8).** Add JSDoc to `scrubToken`, `getAuthenticatedGitHubUrl`, and `setDefaultBranch` if the repo intends the 80% threshold to apply.

[↑ Back to top](#pr-review-rsdto-5--fix-bootstrap-repository-creation-and-docker-git-runtime)

---

### GitHub Review Comment (Follow-up)

`````markdown
**Changes Requested (follow-up, HEAD `27a6c28`)**

Prior-round Items 2, 3, and 4 are resolved, and Item 1's log-leak vector is closed with `scrubToken` + tests. Two problems remain from the initial round, and two new ones surfaced (both corroborated by CodeRabbit's follow-up pass). The standing `APPROVED` review from me on this PR was posted by mistake — please treat this comment as the authoritative status.

---

### Blocking

**1. PAT still persists on disk in `.git/config` (follow-up on prior Item 1).**

The `scrubToken` fix closes the log-leak half of the original concern — thanks. What's still open is that `getAuthenticatedGitHubUrl` interpolates the token into the URL passed to `git clone`, and `git clone` writes that URL to `<tempDir>/.git/config` under `[remote "origin"] url = https://x-access-token:<token>@github.com/…`. The `finally { fs.rm(tempDir, ...) }` covers the happy path and the two new failure-path tests, but if the container is SIGKILL'd or OOM'd between clone and `rm`, the credential stays on disk under whatever `/tmp` is mounted from.

Move the credential onto the command line so it never touches `.git/config`:

```typescript
await git
  .env('GIT_TERMINAL_PROMPT', '0')
  .clone(`https://github.com/${configuration.GITHUB_TEMPLATE_OWNER}/${configuration.GITHUB_TEMPLATE_REPO}.git`, tempDir, [
    '-c', `http.extraheader=Authorization: bearer ${token}`,
    '--single-branch', '--branch', configuration.GITHUB_BRANCH,
  ]);
```

Same treatment for `addRemote('origin', ...)` — set the origin to the plain HTTPS URL and pass the header via `-c` on the `raw(['push', ...])` call. Then `getAuthenticatedGitHubUrl` can be deleted.

**2. Bootstrap is not atomic — a failed step leaves a phantom empty repo that poisons retries.**

In `src/services/github.logic.service.ts` around lines 127–172, the flow is: `POST /orgs/{org}/repos` creates the empty repo, then clone → addRemote → branch -M → push → `setDefaultBranch`. Any failure in the clone/push/setDefaultBranch block leaves the empty repo behind. On retry, `repoExists` returns `true`, the entire `else` block is skipped, and control falls into `copyTemplateFiles` which tries to `GET .../contents/package.json?ref=${initBranch}` on a repo that has neither the branch nor the file. Retries then fail permanently with an opaque "Package fetch failed" error until the empty repo is deleted by hand.

Wrap the block in try/catch and delete the freshly-created repo on failure before re-throwing:

```typescript
let createdThisAttempt = false;
try {
  // POST /orgs/{org}/repos
  createdThisAttempt = true;
  // clone / addRemote / push / setDefaultBranch
} catch (err) {
  if (createdThisAttempt) {
    await fetch(`${api}/repos/${organization}/${repo}`, { method: 'DELETE', headers });
  }
  throw err;
}
```

Add a test that mocks clone failure and asserts the `DELETE` is called with the created repo's path before the error surfaces.

**3. `isBase64Content` mis-classifies short raw source as pre-encoded.**

At `src/services/github.logic.service.ts:35-48`, `isBase64Content` returns `true` for any string that round-trips through `Buffer.from(s, 'base64').toString('base64')`. That includes `"test"`, `"true"`, `"null"`, and any user-controlled rule/test body whose bytes happen to be valid base64. When it returns `true`, `toGitHubContent` passes the raw string through unchanged and GitHub stores the base64-decoded bytes — silent payload corruption.

You confirmed in the inline reply that the API takes raw source. Drop the sniff and always encode:

```typescript
function toGitHubContent(content: string): string {
  return Buffer.from(content, 'utf8').toString('base64');
}
```

Then delete `isBase64Content`. Add a regression test with `content: "test"` asserting the sent value is `"dGVzdA=="`.

---

### Non-blocking (please address in this PR if possible)

**4. Comment the `.push()` → `.raw(['push', ...])` switch (`src/services/github.logic.service.ts:167`).** One line above the call explaining what the switch is avoiding — otherwise a future reader will "clean it up" back to `.push()` and reintroduce whatever was being worked around.

**5. Docstring-coverage pre-merge warning at 0%.** CodeRabbit's pre-merge check reports `Docstring Coverage` failing (threshold 80%). If the threshold is enforced, add JSDoc to the newly-introduced helpers (`scrubToken`, `getAuthenticatedGitHubUrl`, `setDefaultBranch`). If it's advisory, feel free to ignore.
`````

[↑ Back to top](#pr-review-rsdto-5--fix-bootstrap-repository-creation-and-docker-git-runtime)

---

---

---

## Second Follow-up Review (2026-07-17)

**Reviewed commit:** `ab63e3ae33ff61f5cf1da5359ef73c73549627f4` — *"fix: removed secrets from test"* (2026-07-17)
**Reviewed against:** CHANGES_REQUESTED (2026-07-17T14:41Z, review id `4723640287`) by `ahmad-paysys`
**Delta reviewed:** `27a6c28..ab63e3a` — 2 files, +116 / -56 (`src/services/github.logic.service.ts`, `__tests__/unit/github.logic.service.test.ts`)
**Commits in delta:** `660e684` (*fix: PAT leakage fixed*), `5ddd093` (*fix: bootstrap made atomic*), `0457b56` (*refactor: switch avoiding comment added*), `821308b` (*fix: clone auth path fixed*), `eaf54ed` (*refactor: updated regex and remove unnecessary slash escape*), `ab63e3a` (*fix: removed secrets from test*)
**Developer response:** No issue-comment reply. Response is code-only across the six commits above. The inline reply on `github.logic.service.ts:48` from the prior round still stands, and CodeRabbit's counter-reply ([comment 3603624801](https://github.com/tazama-lf/rule-studio-devtestops/pull/5#discussion_r3603624801)) explicitly reiterates that Issue 6 remains open: *"the concern still applies: `isBase64Content()` can mistake raw source such as `test` or `true` for an encoded payload"*.

[↑ Back to top](#pr-review-rsdto-5--fix-bootstrap-repository-creation-and-docker-git-runtime)

---

### Prior Follow-up Items — Resolution Status

#### Item 1a (from Follow-up) — PAT persists on disk in `.git/config`

**Status: RESOLVED**

The credential no longer touches the URL. Commits `660e684` + `821308b` restructure both git operations to pass authentication via `-c http.extraheader=…` on the command line, which is scoped to the single git invocation and never persists to `.git/config`.

At [`src/services/github.logic.service.ts:108-112`](src/services/github.logic.service.ts#L108) the URL builder is replaced by a header builder:

```typescript
const getGitAuthHeader = (token: string): string =>
  `http.extraheader=Authorization: basic ${Buffer.from(`x-access-token:${token}`).toString(
    'base64'
  )}`;
```

Both invocations are rewritten to plain HTTPS URLs with the header injected via `-c`:

```typescript
// clone — line 163-172
const templateRepoUrl = `https://github.com/${configuration.GITHUB_TEMPLATE_OWNER}/${configuration.GITHUB_TEMPLATE_REPO}.git`;
await git
  .env('GIT_TERMINAL_PROMPT', '0')
  .raw([
    '-c', gitAuthHeader,
    'clone', '--single-branch', '--branch', configuration.GITHUB_BRANCH,
    templateRepoUrl, tempDir,
  ]);
// addRemote — line 174-176
const newRepoUrl = `https://github.com/${organization}/${repo}.git`;
await repoGit.addRemote('origin', newRepoUrl);
// push — line 179-180
// Use raw push so the auth header stays command-scoped instead of persisting in .git/config.
await repoGit.raw(['-c', gitAuthHeader, 'push', '-u', 'origin', initBranch]);
```

The origin URL now written to `.git/config` is `https://github.com/{org}/{repo}.git` — no credential. `getAuthenticatedGitHubUrl` is deleted. `scrubToken` is also broadened at [line 99-107](src/services/github.logic.service.ts#L99) to redact the new `http.extraheader=Authorization: basic …` shape as a defense-in-depth for logs. The bootstrap-success test at [`github.logic.service.test.ts:188-215`](__tests__/unit/github.logic.service.test.ts#L188) asserts the exact `-c` argument shape and that `addRemote` receives the plain HTTPS URL, so a regression that reintroduces the token into the URL would fail the test.

#### Issue 5 (from Follow-up New Issues) — Bootstrap not atomic; phantom empty repo poisons retries

**Status: RESOLVED**

The clone/push/setDefaultBranch block at [`src/services/github.logic.service.ts:142-192`](src/services/github.logic.service.ts#L142) is now wrapped in try/catch with a `createdThisAttempt` flag that triggers a compensating `DELETE /repos/{org}/{repo}` if any step after repo creation throws:

```typescript
let createdThisAttempt = false;
try {
  const createRes = await fetch(`${api}/orgs/${organization}/repos`, { ... });
  if (!createRes.ok) throw new Error(...);
  createdThisAttempt = true;
  // ... tempDir, clone, addRemote, branch -M, push, setDefaultBranch, fs.rm in finally
} catch (error) {
  if (createdThisAttempt) {
    await fetch(`${api}/repos/${organization}/${repo}`, { method: 'DELETE', headers });
  }
  throw error;
}
```

The clone-failure test at [`github.logic.service.test.ts:407-412`](__tests__/unit/github.logic.service.test.ts#L407) now asserts the compensating DELETE is issued as the third fetch call:

```typescript
expect(global.fetch).toHaveBeenNthCalledWith(3, 'https://api.github.com/repos/test-org/123', {
  method: 'DELETE',
  headers: expect.objectContaining({
    Authorization: 'token dummy-github-token',
  }),
});
```

Locks the compensating delete in place. One minor observation (non-blocking, see New Issue 9 below) about the compensating-delete not itself being error-handled, but the primary concern is closed.

#### Issue 6 (from Follow-up New Issues) — `isBase64Content` mis-classifies raw source

**Status: NOT RESOLVED**

The code at [`src/services/github.logic.service.ts:35-48`](src/services/github.logic.service.ts#L35) is unchanged across all six delta commits:

```typescript
function isBase64Content(content: string): boolean {
  if (content.trim() === '') return true;
  try {
    return Buffer.from(content, 'base64').toString('base64') === content;
  } catch { return false; }
}

function toGitHubContent(content: string): string {
  return isBase64Content(content) ? content : Buffer.from(content).toString('base64');
}
```

The author's inline reply on the CodeRabbit thread ([comment 3603622105](https://github.com/tazama-lf/rule-studio-devtestops/pull/5#discussion_r3603622105)) — *"saves code as base 64 so no need for this as we need to save raw content"* — confirms the API is meant to receive raw source. CodeRabbit's follow-up counter-reply ([comment 3603624801](https://github.com/tazama-lf/rule-studio-devtestops/pull/5#discussion_r3603624801)) explicitly restates the concern:

> the concern still applies: `isBase64Content()` can mistake raw source such as `test` or `true` for an encoded payload and forward it unchanged, causing GitHub to decode and store incorrect bytes.

Neither the code nor the tests were updated after CodeRabbit's reply. Per the "confirm exists" convention, silence after two independent flags (this reviewer + CodeRabbit) is not the same as declined-with-justification. This item remains **blocking**.

Fix is one line — always encode:

```typescript
function toGitHubContent(content: string): string {
  return Buffer.from(content, 'utf8').toString('base64');
}
```

Then delete `isBase64Content`. Add a regression test with `content: "test"` asserting the sent value is `"dGVzdA=="`.

#### Issue 7 (from Follow-up New Issues) — Document the `.push()` → `.raw(['push', ...])` switch

**Status: RESOLVED**

Commit `0457b56` adds an explanatory comment at [`src/services/github.logic.service.ts:179`](src/services/github.logic.service.ts#L179):

```typescript
// Use raw push so the auth header stays command-scoped instead of persisting in .git/config.
await repoGit.raw(['-c', gitAuthHeader, 'push', '-u', 'origin', initBranch]);
```

The comment accurately captures the "why" (the `-c` header must attach to the specific git invocation) so a future reader won't revert to `.push()` and reintroduce the URL-persistence exposure. Good.

#### Issue 8 (from Follow-up New Issues) — Docstring Coverage 0% pre-merge warning

**Status: NOT RESOLVED (still informational)**

No JSDoc was added to `scrubToken`, `getGitAuthHeader`, or `setDefaultBranch`. `getAuthenticatedGitHubUrl` was deleted (moot). The check remains advisory in the `statusCheckRollup` — not gating merge. Non-blocking; the author can address in a follow-up if the docstring threshold gets enforced.

---

| # | Item | Status |
|---|------|--------|
| 1a | PAT persistence in `.git/config` | ✅ Resolved — credential moved to `-c http.extraheader`, URLs are token-free, tests assert the shape |
| 5 | Bootstrap not atomic — phantom empty repo | ✅ Resolved — compensating DELETE + test |
| 6 | `isBase64Content` heuristic corrupts short raw source | ❌ Not resolved — code unchanged; CodeRabbit re-flagged after author reply |
| 7 | Document `.raw(['push', ...])` switch | ✅ Resolved — comment added |
| 8 | Docstring coverage 0% | ➖ Unchanged — informational, non-blocking |

[↑ Back to top](#pr-review-rsdto-5--fix-bootstrap-repository-creation-and-docker-git-runtime)

---

### New Issues Found in Updated Commits (Second Follow-up)

#### Issue 9 — Compensating DELETE in the atomic-bootstrap catch is itself unguarded

**Severity: Minor (Reliability)**

At [`src/services/github.logic.service.ts:186-192`](src/services/github.logic.service.ts#L186):

```typescript
} catch (error) {
  if (createdThisAttempt) {
    await fetch(`${api}/repos/${organization}/${repo}`, { method: 'DELETE', headers });
  }
  throw error;
}
```

The DELETE is fire-and-forget in two failure modes:

1. If the DELETE itself throws (network hiccup, GitHub 502, PAT lacks `delete_repo` scope), the awaited promise rejects and the original `error` is masked — the caller sees the DELETE error rather than the clone/push error that actually broke the bootstrap.
2. If the DELETE returns a non-2xx status (e.g. 403 — the default GitHub App / PAT permissions don't include `delete_repo`), the response is silently ignored: `res.ok === false` doesn't throw, and the phantom empty repo remains. The retry problem this Issue was meant to solve reappears.

Both are easy to close. Wrap the DELETE in a nested try/catch that logs (via `loggerService.warn`) but does not throw, and check `res.ok` to log the non-2xx case:

```typescript
} catch (error) {
  if (createdThisAttempt) {
    try {
      const delRes = await fetch(`${api}/repos/${organization}/${repo}`, { method: 'DELETE', headers });
      if (!delRes.ok) {
        loggerService.warn(
          `Compensating delete failed for ${organization}/${repo}: HTTP ${delRes.status}`
        );
      }
    } catch (delErr) {
      loggerService.warn(
        `Compensating delete threw for ${organization}/${repo}: ${(delErr as Error).message}`
      );
    }
  }
  throw error;
}
```

Non-blocking — the primary happy-path atomicity is in place — but worth doing before merge so a `delete_repo`-scope-mismatch doesn't silently reintroduce the phantom-repo bug.

#### Issue 10 — `scrubToken`'s `Authorization: basic|bearer` regex misses the `Basic|Bearer` casing that the `git` CLI emits on some paths

**Severity: Informational (Defense-in-Depth)**

At [`src/services/github.logic.service.ts:99-107`](src/services/github.logic.service.ts#L99), the second replacement branch uses the `i` flag on the outer pattern but the `(?:basic|bearer)` alternation is already inside an `i`-flagged regex, so it correctly matches `Basic`/`Bearer`. Verified: `gi` flags cover both. No fix needed — noted here so a future maintainer doesn't strip the `i` and reintroduce a leak. The character class `[\w+/=.-]+` covers standard base64url + base64 + a few extra chars; adequate.

Similarly, if a future auth mode uses `Authorization: token …` (as the REST client already does — see `getGitHubApiConfig` at line 55), the scrubber will not redact it. The REST client's `Authorization: token <PAT>` header does not currently reach `loggerService.error` because fetch failures throw `TypeError: fetch failed` without the request headers — but if any future error path echoes fetch options into the message, the token will land in logs. Add a third replacement clause pre-emptively:

```typescript
.replace(/token [\w+/=.-]+/gi, 'token ***')
```

Non-blocking; note only.

[↑ Back to top](#pr-review-rsdto-5--fix-bootstrap-repository-creation-and-docker-git-runtime)

---

### Updated Verdict (Second Follow-up)

**Verdict: Changes Requested**

Real progress in this round. The Item 1a disk-persistence exposure is closed cleanly by moving the credential from the URL to a command-scoped `-c http.extraheader` argument — the origin URL written to `.git/config` no longer contains a token, and tests lock the shape in place. The atomic-bootstrap fix (Issue 5) lands with a compensating-DELETE and a matching test. The `.raw(['push', ...])` comment (Issue 7) is in.

One blocker remains: `isBase64Content` (Issue 6) is untouched despite CodeRabbit's explicit follow-up reply reiterating the concern after the author's inline response. The current heuristic silently corrupts short raw source that happens to be base64-alphabet-clean; the fix is one line. Ship the drop-the-sniff change (always encode) plus a regression test with `content: "test"`, and this PR is good to merge.

Two new minor items surfaced in the delta — the compensating DELETE swallows its own failures (Issue 9) and could reintroduce the phantom-repo bug if the PAT lacks `delete_repo` scope, and one defense-in-depth suggestion for `scrubToken` (Issue 10). Neither blocks.

#### Blocking

1. **Drop `isBase64Content`; always encode (unchanged from prior round, Issue 6).** One-line fix in `toGitHubContent`, delete the sniff helper, add a regression test with `content: "test"`.

#### Non-blocking but recommended

2. **Guard the compensating DELETE (Issue 9).** Wrap in try/catch + check `res.ok`, log via `loggerService.warn`, do not re-throw — so a DELETE failure doesn't mask the original error and a `delete_repo`-scope-mismatch is at least visible in logs.
3. **Extend `scrubToken` to cover `token <PAT>` shape (Issue 10).** Pre-emptive; the REST client's `Authorization: token …` header can't reach logs today, but the extra clause is cheap.
4. **Docstring coverage (carry-over Issue 8).** Add JSDoc to `scrubToken`, `getGitAuthHeader`, `setDefaultBranch` if the threshold gets enforced.

[↑ Back to top](#pr-review-rsdto-5--fix-bootstrap-repository-creation-and-docker-git-runtime)

---

### GitHub Review Comment (Second Follow-up)

`````markdown
**Changes Requested (second follow-up, HEAD `ab63e3a`)**

Item 1a (PAT in `.git/config`) and Issue 5 (non-atomic bootstrap) are both resolved cleanly — credential is now a command-scoped `-c http.extraheader=…` and no longer touches the origin URL, and a compensating `DELETE /repos/{org}/{repo}` runs on any failure after repo creation. Issue 7's explanatory comment on `raw(['push', ...])` is in. One blocker remains from the prior round, plus one worth-fixing-in-this-PR minor.

---

### Blocking

**1. `isBase64Content` still corrupts short raw source — please drop the sniff (prior Issue 6).**

The code at [`src/services/github.logic.service.ts:35-48`](src/services/github.logic.service.ts#L35) hasn't changed since the last round. Your inline reply confirmed the API takes raw source ("we need to save raw content"), and CodeRabbit's follow-up ([comment 3603624801](https://github.com/tazama-lf/rule-studio-devtestops/pull/5#discussion_r3603624801)) restated the concern:

> `isBase64Content()` can mistake raw source such as `test` or `true` for an encoded payload and forward it unchanged, causing GitHub to decode and store incorrect bytes.

Under the current code, a rule/test body of `test`, `true`, `null`, or any string whose bytes happen to round-trip through base64 decode/encode is passed through unchanged — GitHub then stores the base64-decoded bytes instead of the source text. Silent, payload-dependent corruption.

Drop the sniff and always encode:

```typescript
function toGitHubContent(content: string): string {
  return Buffer.from(content, 'utf8').toString('base64');
}
```

Then delete `isBase64Content`. Add a regression test at [`__tests__/unit/github.logic.service.test.ts`](__tests__/unit/github.logic.service.test.ts) with `ruleCode: 'test'` and assert the outgoing `content` field is `'dGVzdA=='`, not `'test'`.

---

### Non-blocking (please address in this PR if possible)

**2. Guard the compensating DELETE in the atomic-bootstrap catch (`src/services/github.logic.service.ts:186-192`).**

The DELETE is currently fire-and-forget:

```typescript
} catch (error) {
  if (createdThisAttempt) {
    await fetch(`${api}/repos/${organization}/${repo}`, { method: 'DELETE', headers });
  }
  throw error;
}
```

Two failure modes it doesn't handle:
- If DELETE throws (network / GitHub 5xx), the DELETE's error replaces the original `error`, masking the actual bootstrap failure.
- If DELETE returns a non-2xx status (most commonly a 403 when the PAT lacks the `delete_repo` scope), the response is silently ignored and the phantom empty repo remains — the exact regression Issue 5 was meant to prevent.

Small fix:

```typescript
} catch (error) {
  if (createdThisAttempt) {
    try {
      const delRes = await fetch(`${api}/repos/${organization}/${repo}`, { method: 'DELETE', headers });
      if (!delRes.ok) {
        loggerService.warn(`Compensating delete failed for ${organization}/${repo}: HTTP ${delRes.status}`);
      }
    } catch (delErr) {
      loggerService.warn(`Compensating delete threw for ${organization}/${repo}: ${(delErr as Error).message}`);
    }
  }
  throw error;
}
```

**3. Docstring-coverage pre-merge warning still at 0%.** Same guidance as prior round — add JSDoc to `scrubToken`, `getGitAuthHeader`, and `setDefaultBranch` if the CodeRabbit 80% threshold is enforced. If advisory, ignore.
`````

[↑ Back to top](#pr-review-rsdto-5--fix-bootstrap-repository-creation-and-docker-git-runtime)
