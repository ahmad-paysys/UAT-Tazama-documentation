# PR Review: TRS #132 — fix: rule-only simulation NATS subject generation

**Repo:** tazama-lf/rule-studio
**Branch:** `feat-paysys-fix-simulation` → `dev`
**Author:** MuhammadAli-Paysys (Muhammad Ali)
**Date Reviewed:** 2026-07-17
**Label:** bug
**Size:** +4 / -6 lines across 2 files
**Commits:** 2 (`ebe775e`, `d8a5ee0`)
**State:** OPEN (mergeStateStatus: BLOCKED, mergeable)
**HEAD SHA verified:** `d8a5ee030abb34a52abe5456a89814e2cf1c2c23`
**Existing approvals:** none — CodeRabbit COMMENTED ("no actionable comments"), no human approval yet

---

## Table of Contents

- [Overview](#overview)
- [What Changed (Detailed)](#what-changed-detailed)
  - [1. `frontend/src/pages/RuleEditor/Simulation/useSimulationController.tsx`](#1-frontendsrcpagesruleeditorsimulationusesimulationcontrollertsx)
  - [2. `frontend/__tests__/pages/RuleEditor/Simulation/useSimulationController.test.tsx`](#2-frontend__tests__pagesruleeditorsimulationusesimulationcontrollertesttsx)
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

The Rule Studio simulation controller has two modes: **rule-only** (read-only mode; posts a message directly to a deployed rule container via the `nats-utilities` REST bridge) and **end-to-end** (goes through DEMS). This PR is a one-hunk fix in the rule-only branch of `handleSimulation`, plus the corresponding unit-test expectation update.

The previous code split `rule_config_id` on `@` to extract the ID, then rebuilt the NATS `destination`/`consumer` as `sub-rule-<id>@<version>`. The PR body says this produced invalid subjects such as `sub-rule-rule-456@1.0.0` — meaning `rule_config_id` in production was already `rule-456@1.0.0`, and stripping the `@version` then re-templating with `sub-rule-…@<version>` doubled the `rule-` prefix. The fix removes the split/rebuild entirely and interpolates `rule_config_id` verbatim: `destination: \`sub-${rule_config_id}\``.

**Base branch:** `dev` — correct for this repo's flow.

**Failing CI checks:** none — CodeQL, DCO, dependency-review, encoding, hadolint, node-ci (build/style/tests), njsscan, and gpg-verify all pass on `d8a5ee0`.

| File | Nature of Change |
|------|-----------------|
| `frontend/src/pages/RuleEditor/Simulation/useSimulationController.tsx` | Rule-only branch of `handleSimulation` no longer splits `rule_config_id` on `@` and rebuilds; interpolates `rule_config_id` verbatim into `sub-…` / `pub-…`. |
| `frontend/__tests__/pages/RuleEditor/Simulation/useSimulationController.test.tsx` | Expected `destination`/`consumer` values updated from `sub-rule-config-456@1.0.0` to `sub-config-456@1.0.0`. |

[↑ Back to top](#pr-review-trs-132--fix-rule-only-simulation-nats-subject-generation)

---

## What Changed (Detailed)

### 1. `frontend/src/pages/RuleEditor/Simulation/useSimulationController.tsx`

```diff
         if (isReadOnly) {

             const rule_config_id = data?.rule_config_id
-            const id = rule_config_id?.toString().split('@')[0]
-            const version = data?.version
             body = {
                 functionName: '',
                 awaitReply: true,
-                destination: `sub-rule-${id}@${version}`,
-                consumer: `pub-rule-${id}@${version}`,
+                destination: `sub-${rule_config_id}`,
+                consumer: `pub-${rule_config_id}`,
                 message: parsedPayload
             };
             mutation = ruleOnly;
```

The `sub-`/`pub-` template used two inputs: the pre-`@` segment of `rule_config_id`, and `data?.version`. Removed. The new template uses `rule_config_id` verbatim — which, per test data (`rule_config_id: 'config-456@1.0.0'`), already contains an `@version` suffix.

### 2. `frontend/__tests__/pages/RuleEditor/Simulation/useSimulationController.test.tsx`

```diff
                     expect.objectContaining({
                         functionName: '',
                         awaitReply: true,
-                        destination: 'sub-rule-config-456@1.0.0',
-                        consumer: 'pub-rule-config-456@1.0.0',
+                        destination: 'sub-config-456@1.0.0',
+                        consumer: 'pub-config-456@1.0.0',
                         message: { test: 'data' },
                     })
                 );
```

Mock data (`rule_config_id: 'config-456@1.0.0'`) unchanged; only the expectation for the output subject moves.

[↑ Back to top](#pr-review-trs-132--fix-rule-only-simulation-nats-subject-generation)

---

## Code Quality Analysis

### Strengths

- **Deletes dead per-field wrangling.** Extracting `id = rule_config_id.split('@')[0]` and separately reading `data?.version` was fragile — if `rule_config_id` did not contain `@`, the `id` was the full string and the resulting subject appended `@undefined`. Removing that entire chain is a clear win when the source-of-truth already carries the version.
- **Fix and test move together.** The one behavioural change and the one assertion change are in the same PR, so a bisect will always find them together.
- **Scope discipline.** Two-file diff; no incidental cleanups or renames dragged in.

### Issues and Observations

#### Issue 1 — PR body's claimed "correct format" (`sub-rule-456@1.0.0`) contradicts the test's new expectation (`sub-config-456@1.0.0`)

**Severity: Major (Bug / Documentation)**

The PR description states:

> This change ensures rule-only simulations use the correct deployed rule configuration subject format, preventing invalid NATS subjects such as `sub-rule-rule-456@1.0.0` and ensuring messages are routed correctly using the expected `sub-rule-456@1.0.0` format.

But the updated test expects `sub-config-456@1.0.0` — no `rule-` prefix at all. Those cannot both be true. Either:

- The PR body is wrong about the target format, and the code/test are right (the true expected format is `sub-<rule_config_id>` verbatim, and `rule_config_id` in production really is `rule-456@1.0.0` or `config-456@1.0.0` depending on the tenant); or
- The code is wrong: it should strip the `rule-` prefix if present *and* re-prepend a single `rule-`, i.e. `sub-rule-<id-without-rule-prefix>`.

This matters because the *backend* side of the same repo (`/home/ak/workspace/tazama-uat/repos/rule-studio/backend/src/services/simulation-studio/ephemeral-env/ephemeral-env.service.ts:99`) writes:

```typescript
// rule-executer builds streams as sub-rule-${RULE_NAME}@${VERSION} — strip any existing 'rule-' prefix
// so RULE_NAME env stays clean and subjects don't double up (e.g. sub-rule-rule-901@rc)
const ruleBaseName = ruleName.replace(/^rule-/, '');
const natsSubject = `sub-rule-${ruleBaseName}@${version}`;
```

And `backend/src/services/simulation-studio/generation-engine/run-simulation.service.ts:402` also constructs `sub-rule-${ruleName}@${version}`. Both explicitly include the `rule-` prefix; both were written with awareness of the double-prefix trap. So the backend end of the simulation contract, the ephemeral-env spec, and the DEMS Postman collection (`sub-rule-901@rc`) *all* expect a `rule-` prefix on the wire.

The frontend rule-only branch now sends `sub-<rule_config_id>`. If `rule_config_id` is `901` (bare) → wire is `sub-901`, which does not match the container's `sub-rule-901@1.0.0` subscription. If `rule_config_id` is `rule-901@1.0.0` → wire is `sub-rule-901@1.0.0`, which does match. If `rule_config_id` is `config-456@1.0.0` (as in the test) → wire is `sub-config-456@1.0.0`, which does not match the container's `sub-rule-…` subscription unless the deployed container has been re-parameterised.

Please clarify with the author which of these three (or a fourth) is the intended production shape of `rule_config_id`, and either fix the PR body (if the code is correct) or the code (if the PR body is correct). Do not merge until the discrepancy is resolved and the test's mock data reflects the production shape.

#### Issue 2 — `rule_config_id` DTO example is `"CFG001"`, not `"<name>@<version>"` — schema drift risk

**Severity: Major (Data Integrity)**

`backend/src/services/rules/dto/rules.dto.ts:91` documents:

```typescript
@ApiPropertyOptional({ description: 'Configuration identifier', example: 'CFG001', maxLength: 10 })
@IsOptional()
@IsString()
rule_config_id?: string;
```

The Swagger example is `'CFG001'` — a 6-char opaque identifier, `maxLength: 10`. The test mock uses `'config-456@1.0.0'` — 16 chars, containing `@`. If the DTO example reflects the API contract, the interpolated subject would be `sub-CFG001` — with no `@`, and no version — clearly not what the rule-executer subscribes to. If instead the runtime value has always been `"<name>@<version>"` regardless of the Swagger example, then the DTO annotation and `maxLength: 10` constraint are wrong and will start rejecting valid inputs once form validation is wired.

Please reconcile the DTO's `example`/`maxLength` with the new frontend assumption. Either (a) update the DTO example to `'rule-456@1.0.0'` and raise `maxLength`, or (b) revisit the frontend template to build the subject from `rule_name` + `version` fields (which the same controller *already* reads via `data?.rule_name`) rather than assuming `rule_config_id` carries the version.

#### Issue 3 — Sibling code paths in the same repo now diverge on subject construction

**Severity: Major (Consistency / Bug Risk)**

Per the parallel-siblings hunt (Section 3.1), grepping for `sub-rule`/`sub-config` across the repo turns up:

| File | Line | Constructs |
|------|------|-----------|
| `frontend/src/pages/RuleEditor/Simulation/useSimulationController.tsx` | 332 (this PR) | `sub-${rule_config_id}` |
| `backend/src/services/simulation-studio/ephemeral-env/ephemeral-env.service.ts` | 99 | `sub-rule-${ruleBaseName}@${version}` |
| `backend/src/services/simulation-studio/generation-engine/run-simulation.service.ts` | 402 | `sub-rule-${ruleName}@${version}` |
| `backend/src/services/simulation-studio/ephemeral-env/Ephemeral-Env.postman_collection.json` | 159 | `sub-rule-901@rc` (example) |
| `backend/src/services/simulation-studio/ephemeral-env/dto/simulation-info.dto.ts` | 38 | `sub-rule-901@rc` (Swagger example) |
| `backend/test/simulation-studio/run-simulation.service.spec.ts` | 44, 492–507 | `sub-rule-021@rc`, `sub-rule-rule-021@rc` |

Four backend consumers still consistently include `rule-`. The frontend rule-only path no longer does. If the rule-only path targets a *real* deployed rule container built by `deploy.yml` (which sets `RULE_NAME=$RULE_ID`, e.g. `901`), the container is subscribed to `sub-rule-901@1.0.0` — the message will hit the wire on a different subject and time out on the `awaitReply`. Please walk the flow end-to-end against a deployed rule and confirm the new subject actually delivers.

The test in this PR only asserts the *payload* passed to the mocked `ruleOnly` mutation — it never exercises the wire. A green test therefore does not prove routing works. See Test Coverage below.

#### Issue 4 — Silent fallback when `rule_config_id` is undefined

**Severity: Minor (Bug)**

Line 328: `const rule_config_id = data?.rule_config_id`. If `data.rule_config_id` is missing, the interpolation yields `sub-undefined` / `pub-undefined`. Previously the code failed the same way (`sub-rule-undefined@undefined`), so this is not a regression — but the fix is a fine moment to guard:

```typescript
if (!rule_config_id) {
  toast.error('rule_config_id is missing on this rule; simulation cannot run.');
  return;
}
```

Non-blocking; the read-only mode UI probably shouldn't offer the button in that state anyway.

#### Issue 5 — Two commits, second is a test-only rewrite that should probably be squashed

**Severity: Informational (Code Quality)**

Commit history: `ebe775e` "fix: single rule simulation payload updated" then `d8a5ee0` "test: updated test cases". Fine for review; on merge, squash to a single conventional commit so the fix and its test live in one atomic change.

[↑ Back to top](#pr-review-trs-132--fix-rule-only-simulation-nats-subject-generation)

---

## Security Assessment

| Concern | Assessment |
|---------|-----------|
| Unvalidated input reaching a data / command layer | `rule_config_id` originates from the rule editor page's Redux store (populated from the backend `/rules` API). It flows into a NATS *subject string* — NATS subjects are alphanumeric plus `.`/`-`/`_`; special characters would reject at the broker. Not a command-injection concern; a validator on the server side would still be prudent. |
| Auth guards on the endpoint | The `ruleOnly` mutation POSTs to `${VITE_NATS_API_URL}/natsPublish` with a bearer token from `getAuthToken()` (`frontend/src/redux/Api/Nats/index.ts:12-17`). Unchanged by this PR. |
| Secrets / PII in logs / errors | The rule-only branch calls `addSimulationLog(body, res, 'read_only')` (line 349) with the entire payload including the routed `destination`/`consumer`. This is stored via `addLogs`. `rule_config_id` is not PII. No change in exposure. |
| URL / SSRF | `VITE_NATS_API_URL` is a build-time env var, not user-controlled. No new URL construction from untrusted input. |
| HTML rendering / XSS | Not touched. |
| CI security-scanner status | CodeQL Analyze, njsscan, and nodejsscan all pass on `d8a5ee0`. |

No new vulnerabilities are introduced by this PR.

[↑ Back to top](#pr-review-trs-132--fix-rule-only-simulation-nats-subject-generation)

---

## Test Coverage

**What is tested.** The single existing test in `useSimulationController.test.tsx` at line ~600 asserts that when `handleSimulation` runs with a mock `data.rule_config_id = 'config-456@1.0.0'` and readonly=true, the mocked `ruleOnly` mutation receives an object with `destination: 'sub-config-456@1.0.0'` and `consumer: 'pub-config-456@1.0.0'`. The PR updates that assertion to match the new code.

**What is not tested.**

1. **Routing to an actual NATS subject.** The test mocks the mutation entirely; it never checks the wire subject the container is subscribed to. If Issue 3 (sibling divergence) turns out to bite, this test will not catch it. A contract test — even a static assertion that the subject prefix matches `sub-rule-` — would lock the shape.
2. **`rule_config_id` without an embedded `@version`.** No case for `rule_config_id: 'CFG001'` (the DTO example shape). Given Issue 2, this is a live gap: if any real rule has `rule_config_id = 'CFG001'`, the new code sends `sub-CFG001` and the container never receives it.
3. **`rule_config_id` undefined / missing.** No test covers the toast-or-noop path; without a guard the payload silently becomes `sub-undefined`.

**PR checklist state (from PR body).** Marked: Locally ✓, Development Environment ✓, Husky ✓, Unit tests passing ✓, Documentation done ✓. "Not needed" left unchecked. "Frontend Jest suite: 162 test suites passed, 5,215 tests passed" reported in PR body — that number is repo-wide, not evidence that the rule-only branch was manually validated end-to-end.

**Ask the author:** capture a screen recording or NATS log showing a rule-only simulation succeeding against a real deployed rule container using the new subject, and paste the actual on-the-wire subject into the PR. That closes Issue 1/3 cleanly.

[↑ Back to top](#pr-review-trs-132--fix-rule-only-simulation-nats-subject-generation)

---

## CodeRabbit Activity

### Pass 1 — walkthrough on `d8a5ee0`

**Commit reviewed:** `d8a5ee0`
**Findings:** 0 actionable comments

CodeRabbit's summary correctly identifies the shape change: "The read-only rule-only simulation path now builds `destination` and `consumer` from `rule_config_id`, with the corresponding test expectation updated." No blocking or non-blocking items surfaced.

**Reviewer note:** CodeRabbit did not flag the contradiction between the PR description ("expected `sub-rule-456@1.0.0` format") and the test's expectation (`sub-config-456@1.0.0`), nor the sibling divergence with the backend NATS subject construction — those are Issues 1 and 3 above, caught by the parallel-siblings hunt from Section 3.1. No hunt gap in the instructions; CodeRabbit's CHILL profile just did not chase the cross-file consistency.

[↑ Back to top](#pr-review-trs-132--fix-rule-only-simulation-nats-subject-generation)

---

## Summary and Verdict

**Verdict: Changes Requested**

The mechanical diff is right for the *stated* bug (double `rule-` prefix in the subject), and the test moves with it — but there are two large gaps that need clarifying before this can safely merge. First, the PR body says the correct target format is `sub-rule-456@1.0.0`, while the updated test asserts `sub-config-456@1.0.0` — those are different subjects. Second, the backend of the same repo constructs `sub-rule-${ruleBaseName}@${version}` in two separate services and Swagger examples — meaning the rule-executer container the frontend is trying to reach is subscribed to a `sub-rule-…` subject, and the new frontend code will not hit it unless `rule_config_id` happens to already include the `rule-` prefix. Confirming what `rule_config_id` looks like in production, and reconciling the frontend, backend, and DTO shape, is the blocker.

### Blocking

1. **Reconcile the contradiction between the PR body (`sub-rule-456@1.0.0`) and the new test expectation (`sub-config-456@1.0.0`)** — decide which is authoritative and update the other; document what `rule_config_id` actually contains in production.
2. **Reconcile with backend NATS subject construction** — `ephemeral-env.service.ts:99` and `run-simulation.service.ts:402` both build `sub-rule-${ruleName}@${version}`; the new frontend template must either produce a compatible subject or the backend must be updated in lockstep. A green mocked test does not prove routing works.
3. **Reconcile with `RulesDto.rule_config_id` (`example: 'CFG001'`, `maxLength: 10`)** — the DTO example and length constraint contradict the assumption that `rule_config_id` carries `<name>@<version>`. Update the DTO or update the source of the subject.

### Non-blocking but recommended

4. **Guard against `rule_config_id === undefined`** — currently interpolates to `sub-undefined`; a toast + early-return keeps the failure loud.
5. **Squash the two commits on merge** — one fix + its test is a single logical change.

[↑ Back to top](#pr-review-trs-132--fix-rule-only-simulation-nats-subject-generation)

---

## GitHub Review Comment

````markdown
**Changes Requested**

The one-line fix and matching test look mechanically clean, but before merge please resolve a contradiction and confirm the new subject actually routes to a live rule container. A green mocked test does not prove the wire subject matches the container's subscription.

---

### Blocking

**1. PR body and test disagree on the target subject shape.**

The PR description says the correct format is `sub-rule-456@1.0.0` (with a single `rule-` prefix), but [`useSimulationController.test.tsx:603`](frontend/__tests__/pages/RuleEditor/Simulation/useSimulationController.test.tsx#L603) now asserts `sub-config-456@1.0.0` (no `rule-` prefix). Both can't be right — which one is production? Please pick one and update the other, and document what `rule_config_id` actually contains in the deployed system (`CFG001`? `rule-456@1.0.0`? `config-456@1.0.0`?).

**2. Frontend is diverging from backend NATS subject construction.**

The same repo's backend still constructs `sub-rule-${ruleName}@${version}` in two places:

- [`ephemeral-env.service.ts:99`](backend/src/services/simulation-studio/ephemeral-env/ephemeral-env.service.ts#L99)
- [`run-simulation.service.ts:402`](backend/src/services/simulation-studio/generation-engine/run-simulation.service.ts#L402)

Both include the `rule-` prefix, and the `ephemeral-env.service.ts` code even carries a comment about stripping duplicate `rule-` prefixes ("so RULE_NAME env stays clean and subjects don't double up (e.g. sub-rule-rule-901@rc)"). The DEMS Postman example and DTO Swagger example also use `sub-rule-901@rc`. If a deployed rule container built by `rule-studio-example`'s `deploy.yml` sets `RULE_NAME=901`, it subscribes to `sub-rule-901@1.0.0` — the new frontend code will send to `sub-<rule_config_id>`, and unless `rule_config_id` already starts with `rule-`, it will not match.

Please walk the flow end-to-end against a deployed rule and paste the on-the-wire subject into the PR — a screen recording or NATS server log is fine. If routing works, add a static contract check to the test (e.g. `expect(destination).toMatch(/^sub-rule-/)` or `expect(destination).toBe(<literal from server logs>)`) so a future edit can't quietly regress it.

**3. DTO example and length contradict the assumption.**

[`rules.dto.ts:88-91`](backend/src/services/rules/dto/rules.dto.ts#L88-L91) documents `rule_config_id` as `example: 'CFG001', maxLength: 10`. The test mock uses `'config-456@1.0.0'` (16 chars, contains `@`). If a rule with `rule_config_id = 'CFG001'` reaches this code, the new subject is `sub-CFG001` — no version, unlikely to match anything. Either update the DTO (`example: 'rule-456@1.0.0'`, raise `maxLength`) or build the subject from `data?.rule_name` + `data?.version` (both are already available in this same hook) rather than from `rule_config_id`.

---

### Non-blocking (please address in this PR if possible)

**4. Guard `rule_config_id === undefined`.**

If `data?.rule_config_id` is missing, [`useSimulationController.tsx:332-333`](frontend/src/pages/RuleEditor/Simulation/useSimulationController.tsx#L332-L333) interpolates to `sub-undefined` / `pub-undefined` and posts to nats-utilities. A `if (!rule_config_id) { toast.error(...); return }` at the top of the readonly branch keeps the failure loud instead of silently misrouting.
````

[↑ Back to top](#pr-review-trs-132--fix-rule-only-simulation-nats-subject-generation)
