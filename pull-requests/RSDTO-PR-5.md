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
