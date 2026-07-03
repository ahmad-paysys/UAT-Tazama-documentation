# feat-paysys Branch — Commit-by-Commit Diff Report

**Repository:** tazama-lf/rule-executer
**Branch:** `feat-paysys`
**Scope:** 28 commits unique to `feat-paysys` (not present on `dev`), oldest to newest
**Report Date:** 2026-07-03
**Derived from:** `git log origin/feat-paysys --not origin/dev --reverse`

---

## Overview

The 28 commits span from 2026-01-23 to 2026-06-12. They fall into five distinct phases of work, all performed by the Paysys team:

| Phase | Commits | What was happening |
|---|---|---|
| 1 — UAT wiring | C01–C04 | Hardcode UAT infrastructure into Dockerfile; add first transaction log probe |
| 2 — Rule placeholder | C05–C07 | Switch to `@psl-copilot` rule package; add placeholder mechanism; fix RULE_NAME |
| 3 — Library churn | C08–C14 | Rapid `frms-coe-lib` version cycling; add `simple-rule2-test.js`; fix Husky hooks; introduce `as any` cast |
| 4 — Redis / local rule experiment | C15–C22 | POC of Redis distributed cache; swap `handleTransaction` import to a local `rule.ts`; multiple lib bumps; revert Redis |
| 5 — Late lib bumps | C23–C28 | Three `frms-coe-lib` version attempts; final revert to `0.0.1-psl.0`; add `src/controllers/rule.ts` |

---

## Commit Log

---

### C01 — `e2fefc8` — 2026-01-23

**Author:** Syed Hassan Rizwan (`hassan.rizwan@paysyslabs.com`)
**Message:** `Added env conf to testing new rule`

**Files changed:** `Dockerfile` (+13 −13)

**What it does:**
Replaces all empty ENV defaults in the Dockerfile with hardcoded UAT values. Every database host/port/user/password that was blank (the correct production default, allowing runtime injection) is replaced with fixed addresses pointing at a specific Paysys UAT server. The NATS port is also changed from `4222` to `14222`.

```diff
-ENV RAW_HISTORY_DATABASE_HOST=
-ENV RAW_HISTORY_DATABASE_PORT=
-ENV RAW_HISTORY_DATABASE_USER=
-ENV RAW_HISTORY_DATABASE_PASSWORD=
+ENV RAW_HISTORY_DATABASE_HOST=10.10.80.18
+ENV RAW_HISTORY_DATABASE_PORT=15432
+ENV RAW_HISTORY_DATABASE_USER=postgres
+ENV RAW_HISTORY_DATABASE_PASSWORD=postgres
# (identical changes for CONFIGURATION and EVENT_HISTORY databases)

-ENV SERVER_URL=0.0.0.0:4222
+ENV SERVER_URL=0.0.0.0:14222
```

**Assessment:** UAT-only convenience change. Breaks portability for any non-Paysys deployment. Should not be on `dev`.

---

### C02 — `851b4bc` — 2026-01-23

**Author:** Syed Hassan Rizwan (`hassan.rizwan@paysyslabs.com`)
**Message:** `fix: Disable APM and fixed nats server URL for testing`

**Files changed:** `Dockerfile` (+2 −2)

**What it does:**
Two further Dockerfile ENV changes: disables APM entirely and finalises the NATS server URL to include the UAT IP.

```diff
-ENV APM_ACTIVE=true
+ENV APM_ACTIVE=false

-ENV SERVER_URL=0.0.0.0:14222
+ENV SERVER_URL=10.10.80.18:14222
```

**Assessment:** Disabling APM globally silences all observability for every Studio rule that inherits this Dockerfile. Not appropriate for any environment beyond local UAT testing. Should not be on `dev`.

---

### C03 — `857cdd6` — 2026-01-28

**Author:** sohaib1083-paysys (`sohaib.shamsi@paysyslabs.com`)
**Message:** `fix: logs`

**Files changed:** `src/controllers/execute.ts` (+1)

**What it does:**
Inserts a single bare `loggerService.log` call that dumps the full serialised transaction object immediately before the rule config database fetch. No prefix, no context argument — the transaction JSON is logged directly.

```diff
+    loggerService.log(JSON.stringify(request.transaction));
     ruleConfig = await databaseManager.getRuleConfig(ruleRes.id, ruleRes.cfg, request.transaction.TenantId);
```

**Assessment:** A diagnostic probe for debugging NATS message content. Useful during development; a data-exposure risk in production (full transaction PII in logs). Should not be on `dev` in this form.

---

### C04 — `919cd5a` — 2026-02-02

**Author:** Syed Hassan Rizwan (`hassan.rizwan@paysyslabs.com`)
**Message:** `fix: downgrade @tazama-lf/frms-coe-lib to 7.0.0-rc.0 and update rule dependency to @psl-copilot/rule-096@1.0.0`

**Files changed:** `package.json` (+3 −3), `package-lock.json` (large reshuffle)

**What it does:**
Downgrades `frms-coe-lib` from `7.0.0-rc.2` (which was on the branch at that point from the shared history with `dev`) to `7.0.0-rc.0`. Replaces the `rule` dependency from the upstream tazama placeholder (`@tazama-lf/rule-901@3.0.1-rc.0`) with an actual Paysys-scoped rule package (`@psl-copilot/rule-096@1.0.0`).

```diff
-    "@tazama-lf/frms-coe-lib": "7.0.0-rc.2",
+    "@tazama-lf/frms-coe-lib": "7.0.0-rc.0",
-    "rule": "npm:@tazama-lf/rule-901@3.0.1-rc.0",
+    "rule": "npm:@psl-copilot/rule-096@1.0.0",
```

**Assessment:** First time the branch uses a real Paysys rule package. This is the root of the `@psl-copilot` registry dependency and the reason `.npmrc` needs that scope added. Appropriate for local POC; `dev` uses a placeholder to be substituted at build time.

---

### C05 — `d1552ef` — 2026-02-02

**Author:** Syed Hassan Rizwan (`hassan.rizwan@paysyslabs.com`)
**Message:** `Merge branch 'feat-paysys' of https://github.com/tazama-lf/rule-executer into feat-paysys`

**Files changed:** `src/controllers/execute.ts` (+1)

**What it does:**
A merge commit that resolves a divergence between a local and remote `feat-paysys`. The only substantive change picked up is a duplicate of the bare `loggerService.log(JSON.stringify(request.transaction))` call from C03, which re-appears here (suggesting two team members had the same edit on separate working copies).

**Assessment:** Housekeeping merge. No new logic.

---

### C06 — `453c2cf` — 2026-02-03

**Author:** Syed Hassan Rizwan (`hassan.rizwan@paysyslabs.com`)
**Message:** `fix: update rule dependency to use placeholder version and modify Dockerfile environment variable`

**Files changed:** `.npmrc` (+1), `Dockerfile` (+1 −1), `package.json` (+1 −1), `package-lock.json` (+1 −1)

**What it does:**
Three distinct changes in one commit:
1. Adds `@psl-copilot:registry=https://npm.pkg.github.com` to `.npmrc` so npm can resolve Paysys-scoped packages.
2. Reverts the `rule` dependency back to a placeholder form, but using the Paysys scope: `@psl-copilot/rule-placeholder@latest` instead of `@tazama-lf/rule-placeholder@latest`.
3. Changes `RULE_NAME` in the Dockerfile from `"rule-901"` to `"placeholder"`, presumably to match the `sed` substitution logic in the deploy workflow (though the `sed` pattern in `deploy.yml` targets `@tazama-lf`, not `@psl-copilot`, so this is still a mismatch).

```diff
+@psl-copilot:registry=https://npm.pkg.github.com

-ENV RULE_NAME="901"
+ENV RULE_NAME="placeholder"

-    "rule": "npm:@psl-copilot/rule-096@1.0.0",
+    "rule": "npm:@psl-copilot/rule-placeholder@latest",
```

**Assessment:** Attempt to reconcile with the deploy workflow's build-time substitution pattern. The `@psl-copilot` placeholder diverges from the `@tazama-lf` scope that `deploy.yml` targets, making the `sed` step still a no-op for `tazama-lf`-org builds.

---

### C07 — `780a28a` — 2026-02-03

**Author:** Syed Hassan Rizwan (`hassan.rizwan@paysyslabs.com`)
**Message:** `Reverted the rule name to rule-901`

**Files changed:** `Dockerfile` (+1 −1)

**What it does:**
Reverts the `RULE_NAME` Dockerfile ENV back from `"placeholder"` to `"rule-901"`. The `sed` step in the deploy workflow targets this ENV but uses a different pattern; this reverts a change that was apparently not needed.

```diff
-ENV RULE_NAME="placeholder"
+ENV RULE_NAME="rule-901"
```

**Assessment:** Back-and-forth on a placeholder strategy. The end state (`"rule-901"`) differs from `dev` which uses `"901"` (without the prefix). This is what later causes the `RULE_NAME` mismatch visible in the final `feat-paysys` diff.

---

### C08 — `745ce6f` — 2026-02-27

**Author:** sohaib1083-paysys (`sohaib.shamsi@paysyslabs.com`)
**Message:** `fix: updated frmscoelib version`

**Files changed:** `package.json` (+2 −3), `package-lock.json` (large reshuffle), `simple-rule2-test.js` (new file, +124)

**What it does:**
Three things:
1. Bumps `frms-coe-lib` from `7.0.0-rc.0` to `7.0.0-rc.4`.
2. **Removes the `rule` dependency entirely** from `package.json` — this is the commit that strips the placeholder and leaves `package.json` with no `rule` key at all, making the deploy workflow's `sed` step a permanent no-op.
3. Adds `simple-rule2-test.js` — a 124-line standalone Node.js test script that directly calls `handleTransaction` from `rule/lib` with a hard-coded mock request and rule config to verify amount-band logic for a hypothetical "Rule 500". Contains `console.log` output throughout.

```diff
-    "rule": "npm:@psl-copilot/rule-placeholder@latest",
# (line removed, no replacement)
```

**Assessment:** The removal of the `rule` dependency here is the direct cause of the silent `sed` no-op documented in issue #447. `simple-rule2-test.js` is a developer scratchpad with no test runner integration.

---

### C09 — `349bdc9` — 2026-02-27

**Author:** sohaib1083-paysys (`sohaib.shamsi@paysyslabs.com`)
**Message:** `fix: husky pre-commit and pre-push hook syntax and TypeScript errors`

**Files changed:** `.husky/pre-commit` (+3), `.husky/pre-push` (+3)

**What it does:**
Adds the Husky v8 shell preamble to both hook files. The hooks previously contained only the command (`npx lint-staged` / `npm run test`); this commit adds the required header so they execute correctly.

```diff
+#!/usr/bin/env sh
+. "$(dirname -- "$0")/_/husky.sh"
+
 npx lint-staged
```

**Assessment:** Legitimate fix for Husky v8 compatibility. This change is the kind of thing that should be on `dev`. It is, however, partially undone in the next commit.

---

### C10 — `6d993b7` — 2026-02-27

**Author:** sohaib1083-paysys (`sohaib.shamsi@paysyslabs.com`)
**Message:** `fix: package lock and dbMnaager as any`

**Files changed:** `package.json` (+1 −1), `package-lock.json` (minor), `src/controllers/execute.ts` (+1 −1)

**What it does:**
Two changes:
1. Bumps `frms-coe-lib` from `7.0.0-rc.4` to `7.0.0-rc.5`.
2. Adds the `databaseManager as any` cast to the `handleTransaction` call in `execute.ts`. This suppresses a TypeScript type error caused by an incompatibility between the version of `frms-coe-lib` on this branch and the `DatabaseManagerInstance` type expected by the `rule/lib` `handleTransaction` signature.

```diff
-    ruleRes = await handleTransaction(request, determineOutcome, ruleRes, loggerService, ruleConfig, databaseManager);
+    ruleRes = await handleTransaction(request, determineOutcome, ruleRes, loggerService, ruleConfig, databaseManager as any);
```

**Assessment:** The `as any` cast is a code smell that masks a real incompatibility. It signals that the `frms-coe-lib` version on this branch does not satisfy the type contract expected by the rule module. The correct fix is to align library versions, not cast.

---

### C11 — `b0135d3` — 2026-03-12

**Author:** sohaib1083-paysys (`sohaib.shamsi@paysyslabs.com`)
**Message:** `fix: POC protobuff trxn type enabled`

**Files changed:** `package.json` (+1 −1), `package-lock.json` (minor), `src/controllers/execute.ts` (+20 −7)

**What it does:**
Two changes:
1. Bumps `frms-coe-lib` to `7.0.1-rc.ss1` — a Paysys-internal pre-release (`ss` suffix = Sohaib Shamsi).
2. Replaces the single existing log statement in `execute.ts` with a comprehensive set of `[L##]`-prefixed `loggerService.log` calls at every significant step of the function. Also adds the `[L32]` transaction dump with context arguments. This is the commit that introduces the full debug logging pattern visible in the final `feat-paysys` state.

```diff
-  loggerService.log('Start - Handle execute request', context, configuration.functionName);
+  loggerService.log('[L16] Function entry - Starting execute request handler', context, configuration.functionName);
+  loggerService.log('[L22] Starting request parsing and validation', context, configuration.functionName);
+    loggerService.log('[L35] Request parsing completed successfully', context, configuration.functionName);
+    loggerService.log('[L45] APM transaction started', context, configuration.functionName);
# ... (13 more similar additions throughout the function)
```

**Assessment:** This is a structured debug logging effort for tracing transaction flow through NATS. The `[L##]` labels map to approximate line numbers. Useful for one-off debugging; inappropriate as permanent production code. The lib version `7.0.1-rc.ss1` is not available from the `tazama-lf` registry.

---

### C12 — `21680c17` — 2026-03-12

**Author:** sohaib1083-paysys (`sohaib.shamsi@paysyslabs.com`)
**Message:** `fix: POC protobuff trxn type enabled twice`

**Files changed:** `.husky/pre-commit` (+1 −4), `.husky/pre-push` (−3), `package.json` (+1 −1), `package-lock.json` (minor)

**What it does:**
1. Bumps `frms-coe-lib` to `7.0.1-rc.ss2`.
2. **Breaks the Husky hooks**: removes the shell preamble added in C09 from both files and changes `npx lint-staged` to `npx lint - staged` (note the spaces, which make the command fail). Also removes `pre-push` content entirely.

```diff
-.husky/pre-commit:
-#!/usr/bin/env sh
-. "$(dirname -- "$0")/_/husky.sh"
-
-npx lint-staged
+npx lint - staged
```

**Assessment:** The Husky fix from C09 is accidentally undone here. The broken hook syntax means pre-commit linting no longer runs on this branch. A development velocity regression.

---

### C13 — `997c793` — 2026-03-12

**Author:** sohaib1083-paysys (`sohaib.shamsi@paysyslabs.com`)
**Message:** `fix: POC protobuff`

**Files changed:** `package.json` (+1 −1), `package-lock.json` (minor)

**What it does:**
Bumps `frms-coe-lib` from `7.0.1-rc.ss2` to `7.0.1-rc.ss3`. No other changes.

**Assessment:** Iterating on a private pre-release build. Three rapid same-day bumps (C11–C13) suggest active development of `frms-coe-lib` in parallel.

---

### C14 — `724a1d1` — 2026-03-12

**Author:** sohaib1083-paysys (`sohaib.shamsi@paysyslabs.com`)
**Message:** `fix: frms version`

**Files changed:** `package.json` (+1 −1), `package-lock.json` (minor)

**What it does:**
Bumps `frms-coe-lib` from `7.0.1-rc.ss3` to `7.0.1-rc.ss4`. No other changes.

**Assessment:** Continued same-day library iteration. Fourth bump across C11–C14 on a single day.

---

### C15 — `70c0212` — 2026-03-12

**Author:** sohaib1083-paysys (`sohaib.shamsi@paysyslabs.com`)
**Message:** `fix: version`

**Files changed:** `package.json` (+1 −1), `package-lock.json` (minor)

**What it does:**
Bumps `frms-coe-lib` from `7.0.1-rc.ss4` to `7.0.1-rc.ss6` (skips `ss5`). No other changes.

**Assessment:** Fifth library bump in a single day. Skipped `ss5` implies a build was discarded before being committed.

---

### C16 — `6f77db4` — 2026-03-26

**Author:** Ahmad Khalid (`ahmad.khalid@paysyslabs.com`)
**Message:** `feat: updating frms-coe and frms-coe-startup to poc for 1.1 testing`

**Files changed:** `package.json` (+2 −2), `package-lock.json` (large reshuffle)

**What it does:**
Replaces the Sohaib-authored `ss`-suffixed lib versions with Ahmad Khalid's own personal pre-release builds, both under the `ak` suffix:

```diff
-    "@tazama-lf/frms-coe-lib": "7.0.1-rc.ss6",
-    "@tazama-lf/frms-coe-startup-lib": "3.0.0",
+    "@tazama-lf/frms-coe-lib": "7.0.1-rc.ak.14",
+    "@tazama-lf/frms-coe-startup-lib": "3.0.0-rc.ak.12",
```

**Assessment:** Handover of the branch to Ahmad Khalid for 1.1 POC testing. Both `ak`-suffixed versions are personal pre-releases, not published to the `tazama-lf` registry.

---

### C17 — `50c800b` — 2026-03-27

**Author:** Ahmad Khalid (`ahmad.khalid@paysyslabs.com`)
**Message:** `feat: updated Rule Executor to add Redis memcache access for the new safeObject implementation in frms-coe-lib`

**Files changed:** `package.json` (+2 −2), `package-lock.json` (minor), `src/config.ts` (+3 −1), `src/index.ts` (+1 −1)

**What it does:**
This is the most significant source-code change on the branch. It wires up Redis distributed cache alongside the existing local cache:

1. `src/config.ts`: Adds `'redisConfig'` to the `RuleExecutorConfig` type alias, making Redis configuration a required field.
2. `src/index.ts`: Adds `Cache.DISTRIBUTED` to the `CreateStorageManager` call, so the executor initialises both local and distributed (Redis) cache on startup.
3. `package.json`: Bumps both libs to the next `ak` pre-release (`ak.16` / `ak.13`).

```diff
# src/config.ts
-  Pick<ManagerConfig, 'rawHistory' | 'eventHistory' | 'configuration' | 'localCacheConfig'>
+  Pick<ManagerConfig, 'rawHistory' | 'eventHistory' | 'configuration' | 'localCacheConfig' | 'redisConfig'>

# src/index.ts
-    [Database.CONFIGURATION, Database.EVENT_HISTORY, Database.RAW_HISTORY, Cache.LOCAL],
+    [Database.CONFIGURATION, Database.EVENT_HISTORY, Database.RAW_HISTORY, Cache.DISTRIBUTED, Cache.LOCAL],
```

**Assessment:** This is a real feature addition — adding Redis distributed cache support. It is a POC for the `safeObject` implementation in `frms-coe-lib`. It was reverted in C18 (same day), suggesting it did not work as expected or caused test failures.

---

### C18 — `c7fd5ea` — 2026-03-27

**Author:** ReebaPaysys (`reeba.siddiqui@paysyslabs.com`)
**Message:** `fix: revert last changes`

**Files changed:** `src/config.ts` (+1 −3), `src/index.ts` (+1 −1)

**What it does:**
Reverts the Redis changes from C17: removes `'redisConfig'` from the type alias and removes `Cache.DISTRIBUTED` from the `CreateStorageManager` call. Restores both files to their pre-C17 state. `package.json` library versions are not reverted (remain at `ak.16` / `ak.13`).

**Assessment:** The Redis POC was abandoned the same day it was added, by a different author (Reeba). This suggests the feature was either broken, not ready, or the team decided it was out of scope for this branch at that time.

---

### C19 — `805fede` — 2026-03-29

**Author:** Ahmad Khalid (`ahmad.khalid@paysyslabs.com`)
**Message:** `Feat: version updated for frms-* libs. Added a logger statement in execute.ts temporarily to keep track of transaction data passing through NATS.`

**Files changed:** `package.json` (+3 −2), `package-lock.json` (massive reshuffle −2850/+326), `src/config.ts` (+2 −3), `src/controllers/execute.ts` (+1 −1), `src/index.ts` (+1 −1)

**What it does:**
Several things bundled together:
1. Bumps libs to `ak.17` for both.
2. Changes the `cleanup` script from `rm -rf build template …` to `npx rimraf lib node_modules coverage package-lock.json` — a different output directory and tool.
3. Re-reverts `src/config.ts` and `src/index.ts` back to removing `redisConfig`/`Cache.DISTRIBUTED` (they appear to have drifted again).
4. Changes the `handleTransaction` import in `execute.ts` from `'rule/lib'` (the npm package) to `'../rule/rule'` — a **local file import** pointing to a file at `src/rule/rule.ts` which does not appear to exist on the branch at this point.
5. Adds a `[L32]` transaction log entry (redundant with C11, suggesting the `execute.ts` was reset at some point).

```diff
# execute.ts
-import { handleTransaction } from 'rule/lib';
+import { handleTransaction } from '../rule/rule';

+    loggerService.log(`[L32] Request transaction data: ${JSON.stringify(message.transaction)}`, context, configuration.functionName);
```

**Assessment:** The import change to `'../rule/rule'` is a broken state — `src/rule/rule.ts` does not exist on the branch at this commit. This would have caused a build failure. It is corrected in C21. The large `package-lock.json` diff is due to the lib version bumps.

---

### C20 — `151942` — 2026-03-29

**Author:** Ahmad Khalid (`ahmad.khalid@paysyslabs.com`)
**Message:** `Merge branch 'feat-paysys' of https://github.com/tazama-lf/rule-executer into feat-paysys`

**Files changed:** None (empty merge commit)

**What it does:**
Syncs the local and remote `feat-paysys`. No file changes in the merge itself.

**Assessment:** Housekeeping.

---

### C21 — `03e621a` — 2026-03-29

**Author:** ReebaPaysys (`reeba.siddiqui@paysyslabs.com`)
**Message:** `feat: update frm-coe-lib version`

**Files changed:** `package.json` (+2 −2), `package-lock.json` (large reshuffle)

**What it does:**
Bumps both libs from `ak.16`/`ak.13` to `ak.17`/`ak.17`. This is the same version Ahmad bumped to in C19, applied by Reeba — suggesting concurrent development on separate local copies.

**Assessment:** Another parallel bump collision. The two authors were working on the same branch simultaneously without coordinating.

---

### C22 — `696d906` — 2026-03-30

**Author:** Ahmad Khalid (`ahmad.khalid@paysyslabs.com`)
**Message:** `Fix: rule/lib typo in execute.ts import`

**Files changed:** `src/controllers/execute.ts` (+1 −1)

**What it does:**
Fixes the broken import introduced in C19. Changes the import path back from the non-existent local file reference to the npm package.

```diff
-import { handleTransaction } from '../rule/rule';
+import { handleTransaction } from 'rule/lib';
```

**Assessment:** Corrects the build-breaking change from C19. The description calls it a "typo" but it was a deliberate (if broken) attempt to use a local rule implementation. At this point the import is correct again.

---

### C23 — `55ce621` — 2026-03-30

**Author:** ReebaPaysys (`reeba.siddiqui@paysyslabs.com`)
**Message:** *(merge commit — no distinct message body)*
`Merge branch 'feat-paysys' of https://github.com/tazama-lf/rule-executer into feat-paysys`

**Files changed:** `package.json` (+1 −1), `src/controllers/execute.ts` (+2)

**What it does:**
Merge commit that integrates Reeba's C21 bump with Ahmad's C22 fix. The `execute.ts` gains some of the `[L##]` log lines again (suggesting local state divergence), and the `package.json` picks up the `ak.17`/`ak.17` versions.

**Assessment:** Another sync merge. The recurrence of log lines suggests each author had a slightly different `execute.ts` on their local branch.

---

### C24 — `cd820bc` — 2026-04-01

**Author:** ReebaPaysys (`reeba.siddiqui@paysyslabs.com`)
**Message:** `feat: update lib version`

**Files changed:** `package.json` (+2 −2), `package-lock.json` (minor)

**What it does:**
Replaces all `ak`-suffixed pre-release versions with the Paysys-internal consolidated pre-release `0.0.1-psl.0` for both `frms-coe-lib` and `frms-coe-startup-lib`. This is a significant version jump — `0.0.1-psl.0` is a private, non-semver build that consolidates the team's changes but is not published to the `tazama-lf` GitHub Packages registry.

```diff
-    "@tazama-lf/frms-coe-lib": "7.0.1-rc.ak.17",
-    "@tazama-lf/frms-coe-startup-lib": "3.0.0-rc.ak.17",
+    "@tazama-lf/frms-coe-lib": "0.0.1-psl.0",
+    "@tazama-lf/frms-coe-startup-lib": "0.0.1-psl.0",
```

**Assessment:** This is the commit that lands the version that remains in the final `feat-paysys` state (after the oscillation in C25–C28). `0.0.1-psl.0` cannot be resolved by any runner that does not have access to the Paysys private npm registry. This is a blocking issue for `tazama-lf`-org builds.

---

### C25 — `5a905cd` — 2026-06-11

**Author:** abdul-rahim-psl (`abdul.rahim@paysyslabs.com`)
**Message:** `feat: bumped @tazama-lf/frms-coe-lib version to 8.1.0-rc.2`

**Files changed:** `package.json` (+1 −1), `package-lock.json` (moderate reshuffle)

**What it does:**
A new contributor (Abdul Rahim) bumps `frms-coe-lib` from `0.0.1-psl.0` to `8.1.0-rc.2` — a publicly available `tazama-lf` release candidate. This is a major version jump and would have resolved the registry inaccessibility problem. `frms-coe-startup-lib` is not changed.

**Assessment:** Attempt to modernise the branch by switching to a public release candidate. This would have restored build compatibility for `tazama-lf`-org runners.

---

### C26 — `173d37d` — 2026-06-11

**Author:** abdul-rahim-psl (`abdul.rahim@paysyslabs.com`)
**Message:** `feat: re-bumped @tazama-lf/frms-coe-lib version to 0.0.1-psl.0`

**Files changed:** `package.json` (+1 −1), `package-lock.json` (moderate)

**What it does:**
Immediately reverts the `8.1.0-rc.2` bump from C25, returning `frms-coe-lib` to `0.0.1-psl.0`. This was the same day as C25, suggesting the public RC caused a build failure (likely a breaking API change between `7.x` and `8.x` that was not yet resolved on this branch).

```diff
-    "@tazama-lf/frms-coe-lib": "8.1.0-rc.2",
+    "@tazama-lf/frms-coe-lib": "0.0.1-psl.0",
```

**Assessment:** The public RC was tried and immediately rejected. The branch is back to the private, inaccessible version.

---

### C27 — `d583d5d` — 2026-06-11

**Author:** abdul-rahim-psl (`abdul.rahim@paysyslabs.com`)
**Message:** `feat: bumped @tazama-lf/frms-coe-lib version to 8.0.0-rc.5`

**Files changed:** `package.json` (+2 −2), `package-lock.json` (massive reshuffle −928/+55)

**What it does:**
A third attempt in the same day: switches `frms-coe-lib` to `8.0.0-rc.5` and `frms-coe-startup-lib` to `3.0.2-rc.5` — both publicly available `tazama-lf` release candidates. This is the same version range that `dev` uses.

```diff
-    "@tazama-lf/frms-coe-lib": "0.0.1-psl.0",
-    "@tazama-lf/frms-coe-startup-lib": "0.0.1-psl.0",
+    "@tazama-lf/frms-coe-lib": "8.0.0-rc.5",
+    "@tazama-lf/frms-coe-startup-lib": "3.0.2-rc.5",
```

**Assessment:** This appears to be the version combination that builds successfully. Note `dev` is at `8.2.0-rc.6`/`3.1.0-rc.8` — still behind, but at least on public versions.

---

### C28 — `80babb6` — 2026-06-12

**Author:** sohaib1083-paysys (`sohaib.shamsi@paysyslabs.com`)
**Message:** `fix: revert`

**Files changed:** `package.json` (+1 −1), `package-lock.json` (large reshuffle), `src/controllers/rule.ts` (new file, +49)

**What it does:**
Two things in the final commit:

1. **Reverts `frms-coe-lib` back to `0.0.1-psl.0`** — undoing Abdul's C27 fix. This is the version that remains in the current `feat-paysys` HEAD.

```diff
-    "@tazama-lf/frms-coe-lib": "8.0.0-rc.5",
+    "@tazama-lf/frms-coe-lib": "0.0.1-psl.0",
```

2. **Adds `src/controllers/rule.ts`** — a new file providing a local `handleTransaction` function. This function handles the `BaseMessage` transaction type by extracting a numeric `amount` from `Payload` and passing it through `determineOutcome`. It also validates that `bands`, `exitConditions`, and `parameters.tolerance` are present in the rule config. The file contains 5 raw `console.log("hello bhai…")` debug statements.

```typescript
export async function handleTransaction(...): Promise<RuleResult> {
  // config validation...
  console.log("hello bhai the req ", req)
  const transaction = req.transaction;
  console.log("hello bhai the trxn ", transaction)
  if (isBaseMessageTransaction(transaction)) {
    console.log('hi, this is trx', JSON.stringify(transaction))
    const payload = transaction.Payload as Record<string, unknown>;
    const amountRaw = payload.amount;
    if (typeof amountRaw !== 'number') {
      console.log('hello bhai')
      throw new Error('BaseMessage payload missing numeric storyamount.amount');
    }
    console.log('hi, this is trx', JSON.stringify(amountRaw))
    return determineOutcome(amountRaw, ruleConfig, ruleRes);
  }
  throw new Error('Unsupported transaction type: expected Pacs002 or BaseMessage');
}
```

**Assessment:** This is the most consequential single commit on the branch. It reverts the only attempt to align `frms-coe-lib` with a public version, and introduces a local `handleTransaction` override with raw debug output. The mismatch between the `execute.ts` import (`from 'rule/lib'`) and this new file (which is never imported) means `rule.ts` is a dead file at `feat-paysys` HEAD — it was written but never wired up. The revert to `0.0.1-psl.0` is what leaves the branch in its current broken state from a registry perspective.

---

## Summary Table

| # | SHA | Date | Author | Message (abbreviated) | Files | Net effect |
|---|---|---|---|---|---|---|
| C01 | `e2fefc8` | 2026-01-23 | Syed Hassan Rizwan | Added env conf to testing new rule | `Dockerfile` | Hardcodes all DB IPs/ports; changes NATS port to 14222 |
| C02 | `851b4bc` | 2026-01-23 | Syed Hassan Rizwan | fix: Disable APM and fixed nats server URL | `Dockerfile` | Disables APM; finalises NATS URL to `10.10.80.18:14222` |
| C03 | `857cdd6` | 2026-01-28 | sohaib1083-paysys | fix: logs | `execute.ts` | Adds bare `loggerService.log(JSON.stringify(request.transaction))` |
| C04 | `919cd5a` | 2026-02-02 | Syed Hassan Rizwan | fix: downgrade frms-coe-lib… update rule dep | `package.json` | Downgrades lib to `7.0.0-rc.0`; switches rule to `@psl-copilot/rule-096@1.0.0` |
| C05 | `d1552ef` | 2026-02-02 | Syed Hassan Rizwan | Merge feat-paysys | `execute.ts` | Merge; duplicate log line absorbed |
| C06 | `453c2cf` | 2026-02-03 | Syed Hassan Rizwan | fix: update rule dep to placeholder; modify Dockerfile | `.npmrc`, `Dockerfile`, `package.json` | Adds `@psl-copilot` registry scope; introduces placeholder rule dep; changes `RULE_NAME` |
| C07 | `780a28a` | 2026-02-03 | Syed Hassan Rizwan | Reverted the rule name to rule-901 | `Dockerfile` | Changes `RULE_NAME` back from `"placeholder"` to `"rule-901"` |
| C08 | `745ce6f` | 2026-02-27 | sohaib1083-paysys | fix: updated frmscoelib version | `package.json`, `simple-rule2-test.js` | Bumps lib to `rc.4`; **removes `rule` dep entirely**; adds `simple-rule2-test.js` |
| C09 | `349bdc9` | 2026-02-27 | sohaib1083-paysys | fix: husky pre-commit and pre-push hook syntax | `.husky/pre-commit`, `.husky/pre-push` | Adds Husky v8 shell preamble to hooks |
| C10 | `6d993b7` | 2026-02-27 | sohaib1083-paysys | fix: package lock and dbMnaager as any | `package.json`, `execute.ts` | Bumps lib to `rc.5`; adds `databaseManager as any` cast |
| C11 | `b0135d3` | 2026-03-12 | sohaib1083-paysys | fix: POC protobuff trxn type enabled | `package.json`, `execute.ts` | Bumps lib to `rc.ss1`; adds all `[L##]` debug log statements |
| C12 | `21680c17` | 2026-03-12 | sohaib1083-paysys | fix: POC protobuff trxn type enabled twice | `.husky/`, `package.json` | Bumps lib to `rc.ss2`; **breaks Husky hooks** (removes preamble, garbles command) |
| C13 | `997c793` | 2026-03-12 | sohaib1083-paysys | fix: POC protobuff | `package.json` | Bumps lib to `rc.ss3` only |
| C14 | `724a1d1` | 2026-03-12 | sohaib1083-paysys | fix: frms version | `package.json` | Bumps lib to `rc.ss4` only |
| C15 | `70c0212` | 2026-03-12 | sohaib1083-paysys | fix: version | `package.json` | Bumps lib to `rc.ss6` (skips ss5) |
| C16 | `6f77db4` | 2026-03-26 | Ahmad Khalid | feat: updating frms-coe… poc for 1.1 testing | `package.json` | Replaces `ss` lib versions with `ak.14`/`ak.12` (Ahmad Khalid's builds) |
| C17 | `50c800b` | 2026-03-27 | Ahmad Khalid | feat: updated Rule Executor to add Redis memcache | `package.json`, `config.ts`, `index.ts` | Adds `Cache.DISTRIBUTED` + `redisConfig` to type; bumps libs to `ak.16`/`ak.13` |
| C18 | `c7fd5ea` | 2026-03-27 | ReebaPaysys | fix: revert last changes | `config.ts`, `index.ts` | Reverts Redis additions from C17; lib versions unchanged |
| C19 | `805fede` | 2026-03-29 | Ahmad Khalid | Feat: version updated… Added logger statement | `package.json`, `config.ts`, `execute.ts`, `index.ts` | Bumps libs to `ak.17`; **changes import to broken `'../rule/rule'`**; re-reverts Redis |
| C20 | `151942` | 2026-03-29 | Ahmad Khalid | Merge feat-paysys | — | Empty merge |
| C21 | `03e621a` | 2026-03-29 | ReebaPaysys | feat: update frm-coe-lib version | `package.json` | Bumps libs to `ak.17`/`ak.17` (parallel with C19) |
| C22 | `696d906` | 2026-03-30 | Ahmad Khalid | Fix: rule/lib typo in execute.ts import | `execute.ts` | Fixes broken import `'../rule/rule'` back to `'rule/lib'` |
| C23 | `55ce621` | 2026-03-30 | ReebaPaysys | Merge feat-paysys | `package.json`, `execute.ts` | Merge; absorbs C21 and C22 |
| C24 | `cd820bc` | 2026-04-01 | ReebaPaysys | feat: update lib version | `package.json` | Switches to `0.0.1-psl.0` for both libs (private, inaccessible externally) |
| C25 | `5a905cd` | 2026-06-11 | abdul-rahim-psl | feat: bumped frms-coe-lib to 8.1.0-rc.2 | `package.json` | Attempts switch to public RC; `startup-lib` unchanged |
| C26 | `173d37d` | 2026-06-11 | abdul-rahim-psl | feat: re-bumped frms-coe-lib to 0.0.1-psl.0 | `package.json` | Reverts C25 immediately (same day); returns to private version |
| C27 | `d583d5d` | 2026-06-11 | abdul-rahim-psl | feat: bumped frms-coe-lib to 8.0.0-rc.5 | `package.json` | Switches to `8.0.0-rc.5`/`3.0.2-rc.5` — closest to `dev` versions |
| C28 | `80babb6` | 2026-06-12 | sohaib1083-paysys | fix: revert | `package.json`, `rule.ts` | **Reverts lib to `0.0.1-psl.0`; adds `src/controllers/rule.ts` with debug output** |

---

## Authors

| Author | Email | Commits |
|---|---|---|
| Syed Hassan Rizwan | hassan.rizwan@paysyslabs.com | C01, C02, C04, C05, C06, C07 |
| sohaib1083-paysys | sohaib.shamsi@paysyslabs.com | C03, C08, C09, C10, C11, C12, C13, C14, C15, C28 |
| Ahmad Khalid | ahmad.khalid@paysyslabs.com | C16, C17, C19, C20, C22 |
| ReebaPaysys | reeba.siddiqui@paysyslabs.com | C18, C21, C23, C24 |
| abdul-rahim-psl | abdul.rahim@paysyslabs.com | C25, C26, C27 |

---

## What feat-paysys HEAD Contains vs dev

After all 28 commits, the branch HEAD differs from `dev` as follows:

| File | Difference from dev |
|---|---|
| `Dockerfile` | Hardcoded `10.10.80.18:15432` for all DB hosts; `APM_ACTIVE=false`; `SERVER_URL=10.10.80.18:14222`; `RULE_NAME="rule-901"` (dev: `"901"`); plain `ARG GH_TOKEN` instead of BuildKit secret mount |
| `package.json` | `frms-coe-lib 0.0.1-psl.0` (dev: `8.2.0-rc.6`); `frms-coe-startup-lib 0.0.1-psl.0` (dev: `3.1.0-rc.8`); no `rule` dependency (dev has placeholder) |
| `.npmrc` | Extra line: `@psl-copilot:registry=https://npm.pkg.github.com` |
| `src/controllers/execute.ts` | 20+ `[L##]`-prefixed debug log calls; `databaseManager as any` cast |
| `src/controllers/rule.ts` | New file: local `handleTransaction` for `BaseMessage`; 5 `console.log` debug statements; never imported |
| `src/config.ts` | Whitespace-only reformatting of the `RuleExecutorConfig` type alias |
| `.husky/pre-commit` | Broken hook syntax from C12 (`npx lint - staged`) |
| `simple-rule2-test.js` | New file: standalone 124-line test script; not part of any test suite |
