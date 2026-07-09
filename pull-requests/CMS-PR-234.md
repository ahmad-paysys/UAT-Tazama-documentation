# PR Review: CMS #234 — feat: Implemented SLA functionality

**Repo:** tazama-lf/case-management-system
**Branch:** `paysys/SLA-task` → `dev`
**Author:** ibadkhan088 (Ibad Ahmed Khan)
**Date Reviewed:** 2026-07-09
**Label:** enhancement
**Size:** +2,927 / −0 (net) — 100 files touched (backend + frontend + tests + migrations)
**Commits:** 30 (all GPG-verified, all DCO-signed) — head `fc52c649`
**State:** OPEN — `mergeable: true`, `mergeable_state: blocked` (branch-protection gates: needs an approving human review; `reviewDecision: REVIEW_REQUIRED`)
**CI:** all 14 checks SUCCESS — including `dco-check / dco`, `gpg-verify / gpg-verify`, `conventional-commits / validate-pr-title`, `CodeQL`, `njsscan`, `nodejsscan`, `dependency-review`, `encoding-check`, `dockerfile-linter`, `node-ci` (build, tests, style)
**Existing approvals:** none — one human review from `volkkov` (COMMENTED, non-blocking suggestion about a `finally` block); the rest are CodeRabbit passes

---

## Table of Contents

- [Overview](#overview)
- [DCO and Commit Verification Analysis](#dco-and-commit-verification-analysis)
  - [What DCO and GPG Verification Are](#what-dco-and-gpg-verification-are)
  - [Current State on This PR](#current-state-on-this-pr)
  - [The `fix: fixed the DCO issue` Commit](#the-fix-fixed-the-dco-issue-commit)
  - [Cross-Author Commits and Signoffs](#cross-author-commits-and-signoffs)
  - [How This Passes vs. How It Could Have Failed](#how-this-passes-vs-how-it-could-have-failed)
- [What Changed (Detailed)](#what-changed-detailed)
  - [1. Prisma migrations — 9 new migration files](#1-prisma-migrations--9-new-migration-files)
  - [2. `backend/prisma/schema.prisma` — new models and enum rewrite](#2-backendprismaschemaprisma--new-models-and-enum-rewrite)
  - [3. `sla-state.util.ts` — pure derivation of SLA state](#3-sla-stateutilts--pure-derivation-of-sla-state)
  - [4. `sla-policy.util.ts` — tenant-configurable policy resolution](#4-sla-policyutilts--tenant-configurable-policy-resolution)
  - [5. `alert-priority.service.ts` — cron rewrite from alert-recalc to SLA escalation](#5-alert-priorityservicets--cron-rewrite-from-alert-recalc-to-sla-escalation)
  - [6. `case-priority.service.ts` — new supervisor priority-change service](#6-case-priorityservicets--new-supervisor-priority-change-service)
  - [7. `case-priority.util.ts` — DB-backed thresholds replace env vars](#7-case-priorityutilts--db-backed-thresholds-replace-env-vars)
  - [8. `case.controller.ts` — two new endpoints](#8-casecontrollerts--two-new-endpoints)
  - [9. `case-query.service.ts` — derived `sla_state` in list responses](#9-case-queryservicets--derived-sla_state-in-list-responses)
  - [10. `notification.constants.ts` — three new email templates](#10-notificationconstantsts--three-new-email-templates)
  - [11. `alert-priority.task.ts` — deleted](#11-alert-prioritytaskts--deleted)
  - [12. Enum-rename fan-out (DTOs, BPMN, examples)](#12-enum-rename-fan-out-dtos-bpmn-examples)
  - [13. Frontend changes](#13-frontend-changes)
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

This PR replaces the legacy time-based `Priority` enum (`NEW` / `URGENT` / `CRITICAL` / `BREACH`) with a severity-based one (`LOW` / `MEDIUM` / `HIGH`) and adds a full SLA-tracking domain on top of `Case`:

- Every case is stamped with an absolute `sla_due_at` at the moment the SLA clock starts (i.e. when it reaches `STATUS_02_READY_FOR_ASSIGNMENT` — RFA), and `sla_started_at` is persisted as the anchor for that clock.
- `sla_state` (`ON_TRACK` / `AT_RISK` / `DUE_SOON` / `BREACHED`) is **derived** from `(sla_started_at, sla_due_at, now, tenant-ratios)` at read time — never persisted on `Case`.
- Per-tenant configurability is added for both the priority score cutoffs (`case_priority_thresholds`) and the escalation ratios (`sla_escalation_thresholds`), plus per-`(tenant, priority)` SLA target durations (`sla_policies.target_seconds`, converted from hours to seconds mid-PR).
- The old `AlertPriorityTask` cron (which recalculated alert priority via elapsed time) is deleted and replaced by an hourly SLA escalation cron on `AlertPriorityService` that fans out `CASE_CLAIM_CHASE` / `CASE_SUPPORT_CHASE` / `CASE_SLA_BREACHED` notifications, guarded by a `sla_escalation_records` ledger for idempotency.
- A new supervisor-only `PATCH /cases/:caseId/priority` endpoint records a `PRIORITY_CHANGED` audit event and re-anchors `sla_due_at` off `sla_started_at`.
- A new `GET /cases/priority-thresholds` endpoint returns tenant-specific cutoffs so the frontend can preview `LOW`/`MEDIUM`/`HIGH` live in `CreateCaseModal` and `ManualTriageModal`.

The PR description ticks "Locally tested", "Husky run", and "Unit tests passing". CI is fully green (including DCO, GPG verify, and CodeQL). One human review (`volkkov`) commented with a non-blocking suggestion. All 30 commits are signed and DCO-attributed.

| File group | Nature of change |
|------|-----------------|
| `backend/prisma/migrations/2026070*` (9 files) | Enum rename (2-phase), `sla_due_at`, `sla_escalation_records`, `sla_escalation_thresholds`, `case_priority_thresholds`, `target_seconds` unit conversion, `sla_started_at` |
| `backend/prisma/schema.prisma` | New models `SlaPolicy`, `SlaEscalationThreshold`, `CasePriorityThreshold`, `SlaEscalationRecord`; new `SlaState` enum; `Priority` rewrite |
| `backend/src/modules/alert-priority/*` | New `case-priority.service.ts`, new `sla-state.util.ts`; `alert-priority.service.ts` fully rewritten; `alert-priority.task.ts` deleted |
| `backend/src/modules/shared/utils/{case-priority,sla-policy}.util.ts` | DB-backed tenant-scoped threshold/policy resolution with `DEFAULT` tenant fallback and hardcoded final fallback |
| `backend/src/modules/case/{case.controller,case.service,services/*}.ts` | Two new endpoints, SLA fields in query responses, priority-change flow, RFA-triggered SLA clock start |
| `backend/src/constants/notification.constants.ts` | 3 new email templates: `caseClaimChase`, `caseSupportChase`, `caseSlaBreached` |
| `backend/test/*` | 5 new spec files, 8 modified specs — thorough coverage of the new domain |
| `frontend/src/features/{alerts,cases}/*` | Priority enum swap in filters/badges, `usePriorityThresholds` hook, SLA state badge, priority-change modal wiring |
| `backend/.env.example`, `README.md`, `env.validation.ts` | Removed `ALERT_PRIORITY_CRON_SCHEDULE` / `PRIORITY_FIRST_HALF` / etc. |

[↑ Back to top](#pr-review-cms-234--feat-implemented-sla-functionality)

---

## DCO and Commit Verification Analysis

This is the part you asked to see in detail. Two separate mechanisms are in play on every commit in this PR, and both are enforced by required CI checks.

### What DCO and GPG Verification Are

**DCO (Developer Certificate of Origin)** is a text-level attestation. When you commit with `git commit -s`, git appends a trailer to the commit message:

```
Signed-off-by: Ibad Khan <ibad.ahmed@paysyslabs.com>
```

That trailer is the developer certifying, under the DCO terms (roughly: "I have the right to submit this under the project's licence"), that the change is theirs to contribute. It's a **contract**, not a cryptographic proof — anyone who can edit a commit message can add or remove a `Signed-off-by:` line. The `dco-check / dco` action in this repo enforces two things automatically on every push:

1. **Every non-merge commit has a `Signed-off-by:` trailer.**
2. **The email on that trailer matches the commit's `Author:` email** (i.e. you can't sign off on someone else's behalf without also including their own signoff — see below).

**GPG verification** is orthogonal and cryptographic. When you commit with `git commit -S`, git signs the commit object itself with your GPG (or SSH) key. GitHub's `gpg-verify / gpg-verify` action calls the REST API's `/commits/{sha}` and reads `.commit.verification`. A commit is `verified: true, reason: "valid"` only when:

- The signature checks out against a public key on the author's GitHub account.
- The signing identity matches the commit author.
- The key hasn't been revoked or expired.

Together they answer two different questions:

| Mechanism | Question it answers | Enforced by |
|---|---|---|
| DCO | "Did the author consciously agree that they have the right to contribute this?" | `dco-check / dco` |
| GPG verify | "Was this commit actually produced by that author's identity, and untampered?" | `gpg-verify / gpg-verify` |

### Current State on This PR

I pulled `/repos/tazama-lf/case-management-system/pulls/234/commits` and grouped by `.commit.verification.reason`:

```json
[{"reason":"valid","count":30}]
```

**All 30 commits verify.** Every commit also carries at least one `Signed-off-by:` trailer whose email matches the author. Both required checks are green:

- `dco-check / dco` — SUCCESS
- `gpg-verify / gpg-verify` — SUCCESS

### The `fix: fixed the DCO issue` Commit

Commit `502d0028` (2026-07-01) is literally titled `fix: fixed the DCO issue`. It's a real event in this branch's history — an earlier state of the branch failed the DCO gate, and this commit landed the correction. The most common failure modes that a "fix DCO" commit exists to solve are:

1. **Missing `-s` on one or more commits.** `dco-check` fails the whole PR the moment any single commit in the range lacks a `Signed-off-by:` trailer. The fix is either an interactive rebase to re-sign every offender, or squash + re-sign.
2. **Author email ≠ signoff email.** Someone commits as `ibad@laptop.local` but signs off as `ibad.ahmed@paysyslabs.com` — `dco-check` treats those as different identities and fails. The fix is `git commit --amend --author=` and re-sign, or rebase + reset the author email.
3. **A cherry-picked or merged commit from another author retained *their* signoff but not this author's.** DCO requires **every author in the chain** to be represented, not just the topmost committer. See the next section — this PR contains examples where the fix is to add a second signoff line, not overwrite.

Because git preserves history, I can't tell without an event stream (or a force-push log) *which* of the three was the actual trigger for `502d0028`. What I can say is that from `502d002` forward, every subsequent commit is DCO-clean and GPG-valid, and the head is green.

### Cross-Author Commits and Signoffs

Three commits in this PR were authored by someone other than Ibad but committed under Ibad's key. These are the most interesting from a DCO perspective because they demonstrate that the check is correctly enforcing the "chain of consent" rule:

| SHA | `Author` (git) | GH login | `Committer` (git) | Signoff trailers |
|---|---|---|---|---|
| `81c8def1` | Kyle Vorster `<kyle.vorster69@gmail.com>` | KyleVorster7 | Ibad Khan | **Two** — `Kyle Vorster <kyle.vorster69@gmail.com>` **and** `Ibad Khan <ibad.ahmed@paysyslabs.com>` |
| `5eeb4adb` | Muhammad Ali `<ali.shoaib@paysyslabs.com>` | MuhammadAli-Paysys | Ibad Khan | **Two** — `Muhammad Ali <ali.shoaib@paysyslabs.com>` **and** `Ibad Khan <ibad.ahmed@paysyslabs.com>` |
| `dd7b31d0` | KyleVorster7 (via `108583489+KyleVorster7@users.noreply.github.com`) | KyleVorster7 | Ibad Khan | **One** — `Ibad Khan <ibad.ahmed@paysyslabs.com>` only |

The first two are the correct pattern: when Ibad brought in Kyle's / Ali's work (via `git cherry-pick -s -S` or merge + re-sign), both authors' signoffs are preserved, so both DCO conditions are satisfied and `gpg-verify` sees a chained-of-trust commit that Ibad re-signed with his own key while preserving Kyle/Ali as the git-level author.

The third (`dd7b31d0`) is the merge of PR #235 into this branch. It's marked `verified: true` and passes DCO in this repo because the `dco-check` action typically skips merge commits (they carry no semantic diff of their own), which is why the missing Kyle-signoff there does not fail the gate. If you tighten `dco-check` to include merges in future, this exact commit pattern would be the one that starts failing.

### How This Passes vs. How It Could Have Failed

For this PR specifically, here's why each check is green:

- **`gpg-verify`** — every commit's `verification.verified` is `true` with `reason: "valid"`. Ibad committed everything (as `committer`) with a key registered on his GitHub account and set as the *verifying* key. Even the commits authored by Kyle and Ali verify, because *Ibad* is the committer and it's Ibad's key that GitHub cross-references; git's data model separates the author (who wrote the change) from the committer (who applied it), and `verify` looks at the committer.
- **`dco-check`** — every non-merge commit has a `Signed-off-by:` whose email exactly matches the `Author:` email. The two co-authored commits from Kyle and Ali include a second signoff for Ibad, which is the required pattern for the "committer ≠ author" case.
- **`conventional-commits / validate-pr-title`** — the PR title `feat: Implemented SLA functionality` matches the conventional-commit regex.

**What would break a future PR:**

- A single `git commit` (no `-s`) — DCO fails immediately.
- A commit authored on a machine where `user.email` differs from your GitHub-verified email — DCO fails on the trailer mismatch.
- Cherry-picking someone else's commit without their signoff — DCO fails; you must `git cherry-pick -s` **and** ensure their original `Signed-off-by:` line stays in the message.
- Committing without `-S` (or without `commit.gpgsign=true`) — `gpg-verify` fails.
- Rebasing a signed commit onto a new base with a signing config that has been removed — the rebased commit is unsigned, `gpg-verify` fails.

[↑ Back to top](#pr-review-cms-234--feat-implemented-sla-functionality)

---

## What Changed (Detailed)

### 1. Prisma migrations — 9 new migration files

The enum rename is a **two-phase** migration because PostgreSQL doesn't let you rename two enum values to the same target in one `ALTER TYPE`, and it has no `DROP VALUE` at all. The migration comments are unusually good — they explain *why* the dance is necessary.

**Phase 1 — `20260702100000_priority_severity_rename/migration.sql`:**

```sql
-- Step 1: rename the 1:1 mappings
ALTER TYPE "Priority" RENAME VALUE 'NEW' TO 'LOW';
ALTER TYPE "Priority" RENAME VALUE 'URGENT' TO 'MEDIUM';
ALTER TYPE "Priority" RENAME VALUE 'CRITICAL' TO 'HIGH';

-- Step 2: column defaults referenced the literal 'NEW' — reset them
ALTER TABLE "cases" ALTER COLUMN "priority" SET DEFAULT 'LOW';
ALTER TABLE "alerts" ALTER COLUMN "priority" SET DEFAULT 'LOW';

-- Step 3: backfill BREACH rows onto HIGH
UPDATE "cases" SET "priority" = 'HIGH' WHERE "priority" = 'BREACH';
UPDATE "alerts" SET "priority" = 'HIGH' WHERE "priority" = 'BREACH';
```

**Phase 2 — `20260702100100_priority_severity_drop_breach/migration.sql`** rebuilds the enum type without `BREACH`:

```sql
CREATE TYPE "Priority_new" AS ENUM ('LOW', 'MEDIUM', 'HIGH');
ALTER TABLE "cases" ALTER COLUMN "priority" DROP DEFAULT;
ALTER TABLE "cases" ALTER COLUMN "priority" TYPE "Priority_new" USING ("priority"::text::"Priority_new");
ALTER TABLE "cases" ALTER COLUMN "priority" SET DEFAULT 'LOW';
-- ...same for "alerts"...
DROP TYPE "Priority";
ALTER TYPE "Priority_new" RENAME TO "Priority";
```

Phase 2 **must** be a separate migration because phase 1's renames need to commit before the cast in phase 2 can reference the new labels. Prisma runs each migration in its own transaction, so this is automatic if they're separate files.

The other seven migrations create `sla_policies`, `sla_escalation_records`, `sla_escalation_thresholds`, `case_priority_thresholds`, add `sla_due_at` (with a 72h backfill for open cases), convert `target_hours → target_seconds` (multiplied by 3600), and finally add `sla_started_at` (no backfill — existing open cases only get it on their next RFA entry).

### 2. `backend/prisma/schema.prisma` — new models and enum rewrite

New: `SlaPolicy`, `SlaEscalationThreshold`, `CasePriorityThreshold`, `SlaEscalationRecord`, `enum SlaState`. `Priority` shrinks to `LOW / MEDIUM / HIGH`. `Case` gains `sla_started_at` and `sla_due_at` (both nullable) plus a back-reference to `SlaEscalationRecord[]`.

### 3. `sla-state.util.ts` — pure derivation of SLA state

`determineSlaState(slaStartedAt, slaDueAt, now, ratios)` is a pure function that returns `BREACHED` when `now >= slaDueAt`, otherwise computes `remainingMs / totalBudgetMs` and maps against tenant-configured `dueSoonRatio` / `atRiskRatio`. Callers pre-resolve the ratios per tenant so this stays synchronous and safe to call per-row inside a `.map()`.

Notable design decisions the code comments call out explicitly:

- `totalBudget` is `slaDueAt − slaStartedAt`, **not** `slaDueAt − createdAt`. Cases sitting in `DRAFT` or `PENDING_CASE_CREATION_APPROVAL` don't dilute the ratio.
- The ratios are priority-agnostic — one pair per tenant applies uniformly across `LOW/MEDIUM/HIGH` because the ratios compare fractions of the case's own budget, not absolute times.
- `sla_state` is never persisted; the value is always freshly derived.

### 4. `sla-policy.util.ts` — tenant-configurable policy resolution

Three lookup methods, each with a **three-level fallback chain**:

1. `getTargetSeconds(tenantId, priority)` — tenant-specific `sla_policies` → `DEFAULT` tenant `sla_policies` → hardcoded `FALLBACK_TARGET_SECONDS` (24h/72h/168h).
2. `getEscalationRatios(tenantId)` — tenant-specific `sla_escalation_thresholds` → `DEFAULT` → hardcoded `0.2`/`0.5`.
3. `calculateSlaDueAt(createdAt, tenantId, priority)` — composed on top of `getTargetSeconds`.

`DEFAULT_TENANT_KEY = 'DEFAULT'` is shared across this file and `case-priority.util.ts`. CodeRabbit flagged the duplicate query when `tenantId === DEFAULT_TENANT_KEY` (both lookups query the same row) as a "trivial" perf nit — see `## CodeRabbit Activity`.

### 5. `alert-priority.service.ts` — cron rewrite from alert-recalc to SLA escalation

The old service (155 lines) recalculated alert priority by elapsed time. It's gone. The new service (166 lines) runs `@Cron(CronExpression.EVERY_HOUR)` and:

1. Loads open cases with an SLA via `caseRepository.findOpenCasesForSlaCheck()`.
2. Deduplicates tenants and resolves each tenant's escalation ratios **once** (an intentional guard against N+1 across the batch).
3. `Promise.allSettled` fans out to `checkCase` per row.
4. Logs the completion summary and per-case failures individually.
5. Re-throws the first failure so the tick surfaces as failed to monitoring.

`checkCase()` derives the SLA state, then routes based on claim status:
- `BREACHED` overrides everything → notify `SUPERVISORS` group + emit `case.sla-breached` domain event.
- `AT_RISK` + unclaimed → notify `INVESTIGATIONS` group (claim chase).
- `DUE_SOON` + claimed → notify the case owner (support chase).

`escalate()` guards idempotency two ways: a pre-check against `slaEscalationRecord` for the `(case_id, sla_state)` pair, and a Prisma unique-constraint `P2002` catch after the notification succeeds. Notification failures return early without persisting the ledger row, so the next tick retries.

This is the second major back-and-forth with CodeRabbit — the initial version used `Promise.all` (which short-circuits on the first rejection), and the author explicitly switched to `Promise.allSettled` + per-failure log + re-throw of the first rejection. There is one leftover eslint issue CodeRabbit flagged and I'll cover it in `## Issues and Observations`.

### 6. `case-priority.service.ts` — new supervisor priority-change service

54 LOC. `changePriority(caseId, newPriority, actorId, tenantId, reason?)`:

1. Loads the existing case.
2. Calls `caseRepository.updateCase(caseId, { priority })` — which re-anchors `sla_due_at` off `sla_started_at` inside the repository (see below).
3. Emits `CasePriorityChangedEvent`.
4. Returns the old/new/actor/reason tuple for the controller to serialise.

The comment `// Supervisor-only access is enforced by @RequireSupervisorRole() on the controller route — no role check here, so there's a single, guard-level source of truth.` is exactly the right note — no defensive double-check.

### 7. `case-priority.util.ts` — DB-backed thresholds replace env vars

The old code read three env vars (`PRIORITY_FIRST_HALF`, `PRIORITY_SECOND_HALF`, `PRIORITY_THIRD_HALF`) and mapped scores to `NEW/URGENT/CRITICAL/BREACH`. The new code:

```ts
async determinePriority(priorityScore: number, tenantId: string): Promise<Priority> {
  const { highThreshold, mediumThreshold } = await this.getThresholds(tenantId);
  if (priorityScore >= highThreshold) return Priority.HIGH;
  if (priorityScore >= mediumThreshold) return Priority.MEDIUM;
  return Priority.LOW;
}
```

`getThresholds()` uses the same three-level fallback chain as `SlaPolicyUtil`. The method signature changed from sync → async and from `(score)` → `(score, tenantId)`; every call site was updated. `FALLBACK_HIGH_THRESHOLD = 0.7` and `FALLBACK_MEDIUM_THRESHOLD = 0.4` are exported for use by the tests.

### 8. `case.controller.ts` — two new endpoints

```ts
@Get('priority-thresholds')
@RequireAuthenticated()
async getPriorityThresholds(@Req() req): Promise<CasePriorityThresholds> {
  const { tenantId } = extractUserData(req);
  return this.casePriorityUtil.getThresholds(tenantId);
}

@Patch(':caseId/priority')
@RequireSupervisorRole()
@Audit()
@HttpCode(HttpStatus.OK)
async changeCasePriority(@Param('caseId') caseId, @Body() dto, @Req() req): Promise<PriorityChangeResult> {
  const { userId, tenantId } = extractUserData(req);
  return this.casePriorityService.changePriority(caseId, dto.newPriority, userId, tenantId, dto.reason);
}
```

`@Audit()` is wired up in the interceptor with the new `changeCasePriority` entry:

```ts
changeCasePriority: {
  description: 'Changed case priority',
  eventType: 'PRIORITY_CHANGED',
},
```

**Route ordering matters:** `GET priority-thresholds` is registered **before** `GET :caseId`. If it were the other way around, `priority-thresholds` would be matched as `:caseId="priority-thresholds"` and 404 (or worse, be misinterpreted). The current order is correct.

### 9. `case-query.service.ts` — derived `sla_state` in list responses

`getUserCases` and `getAllCases` now:

1. Fetch the case list (as before).
2. Call `resolveRatiosByTenant([...tenantIds])` — one lookup per distinct tenant in the page.
3. In the `.map()`, compute `sla_state` via `computeCaseSlaState(caseItem, ratiosByTenant.get(caseItem.tenant_id)!)`.

The response DTO gains `sla_due_at: Date | null` and `sla_state: SlaState | null`. `null` means the case has no meaningful SLA state (never reached RFA, or legacy case pre-`sla_started_at`).

### 10. `notification.constants.ts` — three new email templates

`caseClaimChase`, `caseSupportChase`, `caseSlaBreached`. Each is a `Record<string, any> → string` HTML template. All three consistently use `${data.assignee ?? 'Unassigned'}` after CodeRabbit flagged the missing fallback on `caseSupportChase` — the fix is applied in the diff at head.

### 11. `alert-priority.task.ts` — deleted

31 lines gone. This was the standalone cron shell that called `AlertPriorityService.runRecalculation()`. Its scheduling logic now lives inside the service via `@Cron`.

### 12. Enum-rename fan-out (DTOs, BPMN, examples)

Roughly a dozen files touch only Swagger `example` strings (`'URGENT'` → `'MEDIUM'`), `enum: ['NEW','URGENT','CRITICAL','BREACH']` → `enum: ['LOW','MEDIUM','HIGH']`, or `Priority.URGENT` literals. The BPMN form property `casePriority` is updated too:

```diff
- <flowable:formProperty id="casePriority" name="Case Priority" type="enum" default="NEW">
-   <flowable:value id="NEW" name="New"/>
-   <flowable:value id="URGENT" name="Urgent"/>
-   <flowable:value id="CRITICAL" name="Critical"/>
-   <flowable:value id="BREACH" name="Breach"/>
+ <flowable:formProperty id="casePriority" name="Case Priority" type="enum" default="LOW">
+   <flowable:value id="LOW" name="Low"/>
+   <flowable:value id="MEDIUM" name="Medium"/>
+   <flowable:value id="HIGH" name="High"/>
```

`env.validation.ts` drops `ALERT_PRIORITY_CRON_SCHEDULE` (no longer read now that scheduling is a `@Cron` decorator), and `README.md` / `.env.example` are aligned.

### 13. Frontend changes

- `AlertDetails.tsx` (419 LOC) and its test file are **deleted** — this component was legacy and its detail view is now handled elsewhere.
- `usePriorityThresholds.ts` (new) fetches from `/cases/priority-thresholds` and returns `{ thresholds, calculatePriority }`. CodeRabbit flagged an exhaustive-deps footgun on `calculatePriority` (recreated every render); this appears not to be memoized in head.
- `ManualTriageModal.tsx` and `CreateCaseModal.tsx` both consume the hook. CodeRabbit flagged that the threshold-loading logic was copy-pasted across both modals; the fix (extract the shared hook) is applied.
- `CaseFilters.tsx`, `CaseModalsManager.tsx`, and various Alerts components have their enum values swapped.

[↑ Back to top](#pr-review-cms-234--feat-implemented-sla-functionality)

---

## Code Quality Analysis

### Strengths

- **Migration commentary.** Every migration file has a block comment explaining the *why*, not just the *what* — the enum-rename dance, the reason for the transaction split, the choice to leave closed cases' `sla_due_at` NULL. This is by far the strongest documentation in the diff and directly answers the questions a future migration author will ask.
- **Derived-not-stored `sla_state`.** Persisting it would create the classic staleness problem where the DB row lies until the next cron tick. Keeping it derived and pushing the ratio lookup out of the loop is the right shape.
- **Three-level fallback chain.** `tenant row → DEFAULT tenant row → hardcoded constant`, applied identically in both `SlaPolicyUtil` and `CasePriorityUtil`. Consistent and easy to reason about.
- **Idempotency of escalation.** `sla_escalation_records` is a UNIQUE `(case_id, sla_state)` ledger. `escalate()` reads-before-write, and if a concurrent tick beats it, it swallows the `P2002` and moves on. This is the correct pattern for cron-driven notifications.
- **Failure isolation in the cron.** `Promise.allSettled` + per-failure log + surface-first-failure is a considered choice (the author explicitly justified it in the CodeRabbit thread), and the domain event `CaseSlaBreachedEvent` cleanly decouples the notification path from downstream subscribers.
- **Guard-level role enforcement.** `@RequireSupervisorRole()` on the controller is the single source of truth for who can change priority — no defensive duplicate check in the service.
- **Test breadth.** 5 new spec files, 8 modified — covers `sla-state.util`, `sla-policy.util`, `case-priority.util`, `case-priority.service`, and a fully-rewritten `alert-priority.service.spec.ts` (290 additions, 225 deletions).

### Issues and Observations

#### Issue 1 — CI-style lint failure risk on the `throw` cast

**Severity: Minor (Code Quality)**

CodeRabbit flagged this against commit `fc52c64` in `alert-priority.service.ts` line 71:

```ts
if (failures.length > 0) {
  throw failures[0].reason;   // failures[0].reason is any → trips @typescript-eslint/only-throw-error
}
```

The current head already re-wraps it as:

```ts
if (failures.length > 0) {
  const [{ reason }] = failures;
  throw reason instanceof Error ? reason : new Error(String(reason));
}
```

which is even better than the `as Error` cast CodeRabbit suggested. This is **already resolved** in head — no action needed. Recording it here only because it appears prominently in the review thread and might be re-flagged if the file is rebased.

#### Issue 2 — `usePriorityThresholds.calculatePriority` recreated every render

**Severity: Minor (Code Quality — Frontend)**

CodeRabbit flag (commit `0d836a6`, `usePriorityThresholds.ts:41-48`): `calculatePriority` is recreated on every render inside the hook, so any consumer that puts it in a `useEffect`/`useMemo` dependency array will either re-fire on every render or trip exhaustive-deps.

**Fix:**

```ts
const calculatePriority = useCallback(
  (score: number): Priority => {
    if (!thresholds) return Priority.LOW;
    if (score >= thresholds.highThreshold) return Priority.HIGH;
    if (score >= thresholds.mediumThreshold) return Priority.MEDIUM;
    return Priority.LOW;
  },
  [thresholds],
);
```

Not blocking — no consumer currently puts it in a dep array — but worth doing before someone tries.

#### Issue 3 — Duplicate query when `tenantId === DEFAULT_TENANT_KEY`

**Severity: Informational (Performance)**

CodeRabbit flag against `sla-policy.util.ts:32-44`. When the caller's tenant is literally the string `'DEFAULT'`, both the tenant-specific lookup and the fallback lookup hit the same row. Cost is one wasted query per call.

**Fix:**

```ts
async getTargetSeconds(tenantId: string, priority: Priority): Promise<number> {
  if (tenantId !== DEFAULT_TENANT_KEY) {
    const tenantPolicy = await this.prisma.slaPolicy.findFirst({ where: { tenant_id: tenantId, priority } });
    if (tenantPolicy) return tenantPolicy.target_seconds;
  }
  const defaultPolicy = await this.prisma.slaPolicy.findFirst({ where: { tenant_id: DEFAULT_TENANT_KEY, priority } });
  if (defaultPolicy) return defaultPolicy.target_seconds;
  return FALLBACK_TARGET_SECONDS[priority];
}
```

CodeRabbit itself labeled this "trivial / low value" — flag for awareness, not merge-blocking.

#### Issue 4 — `case.repository.findOpenCasesForSlaCheck()` is unbounded

**Severity: Informational (Performance — pre-existing pattern)**

CodeRabbit flagged this and the author responded: "The query is already limited to open, non-abandoned cases that have an SLA, and it only selects six columns. Based on the current workload, this should remain a reasonably bounded dataset." CodeRabbit accepted the response.

I agree this is fine for the current expected workload. Note it for revisit when case volume grows past, say, five figures of concurrently-open cases per hour. No change needed now.

#### Issue 5 — Commented-out alert-update block in `case.service.ts`

**Severity: Minor (Code Quality)**

Lines 1104–1116 (approximately) contain a large commented block:

```ts
// const getAlertIdByCaseId = await this.alertRepository.getAlertByCaseId(caseId);
// if (getAlertIdByCaseId) {
//   const alertUpdateData = { ... };
//   await this.alertRepository.updateAlert(getAlertIdByCaseId, alertUpdateData);
//   this.logger.log(...);
// }
```

If this is dead code from the old flow, delete it. If it's a TODO, leave a one-line comment stating why it's paused and what would re-enable it. Long commented-out blocks are drift-prone and confusing to future readers.

#### Issue 6 — `PR description is empty of the "How was it tested?" details`

**Severity: Informational (Process)**

The template's `## How was it tested?` checklist has "Locally" and "Husky successfully run" and "Unit tests passing" checked, but the section has no free-text on what scenarios were manually exercised (SLA breach → notification → ledger → no re-notify; priority change → sla_due_at re-anchor; DEFAULT-tenant fallback). Given the size of the diff and the operational impact of the cron, a short narrative in the description would help reviewers verify manually.

[↑ Back to top](#pr-review-cms-234--feat-implemented-sla-functionality)

---

## Security Assessment

| Concern | Assessment |
|---|---|
| **Auth on new endpoints** | `PATCH /:caseId/priority` is gated by `@RequireSupervisorRole()`. `GET /priority-thresholds` is gated by `@RequireAuthenticated()`, which is appropriate — the response is scoped to the requesting user's `tenantId`, so no cross-tenant leakage. |
| **Tenant isolation** | `changePriority` calls `findCaseById(caseId, tenantId)` — the tenant scope comes from the JWT (`extractUserData(req)`), not from the request body, so a supervisor can't retarget the mutation to another tenant's case. |
| **Audit trail** | `@Audit()` interceptor now handles `changeCasePriority` with `eventType: 'PRIORITY_CHANGED'`. Old priority, new priority, actor ID, and free-text reason are captured. |
| **Injection risk in email templates** | `caseSupportChase` and `caseSlaBreached` interpolate `data.assignee`, `data.caseId`, `data.caseType` into HTML. These come from `case_owner_user_id` (UUID), `case_id` (int), and `case_type` (enum) — none is user-supplied free-form text, so HTML injection is not a live risk. If any of these fields ever start accepting free text, escaping will need adding. |
| **DoS via cron** | `runSlaEscalationCheck` loads all open cases with an SLA every hour. Pre-existing observation (see Issue 4) — not exploitable, but bounded pagination is a good future-proofing. |
| **Race on the escalation ledger** | Guarded by `UNIQUE (case_id, sla_state)` at the DB level and a `P2002`-catch in the app code. Correct. |
| **Signed commits / DCO** | See [DCO and Commit Verification Analysis](#dco-and-commit-verification-analysis) above — all 30 commits verified, all DCO-clean. |

No new security vulnerabilities introduced by this PR.

[↑ Back to top](#pr-review-cms-234--feat-implemented-sla-functionality)

---

## Test Coverage

**New spec files (5):**

- `test/case-priority.service.spec.ts` (89 LOC) — happy path, old/new priority capture, audit event emission, tenant scoping.
- `test/case-priority.util.spec.ts` (91 LOC) — tenant lookup, DEFAULT fallback, hardcoded fallback, all three score→priority boundaries.
- `test/sla-policy.util.spec.ts` (101 LOC) — tenant, DEFAULT, and hardcoded fallback for `getTargetSeconds` / `getEscalationRatios` / `calculateSlaDueAt`.
- `test/sla-state.util.spec.ts` (66 LOC) — `BREACHED` when past due, boundary tests for `AT_RISK` / `DUE_SOON` / `ON_TRACK`, zero-budget edge case.
- `test/report.service.spec.ts` (27 LOC) — priority bucket reconciliation with closed cases (fixes a reported bug).

**Modified spec files (13):**

- `alert-priority.service.spec.ts` — full rewrite (+290, −225). Covers the cron happy path, `Promise.allSettled` failure isolation ("propagates non-constraint errors"), notification skip on already-notified state, and per-tenant ratio caching.
- `case-query.service.spec.ts` (+124, −17) — verifies `sla_state` computation across a mixed-tenant page.
- `triage.service.spec.ts` (+116, −38) — priority-computation rewiring.
- `case-reopening.service.spec.ts`, `case-creation.service.spec.ts`, `case-creation-approval.service.spec.ts` — SLA-clock restart tests.
- Enum-swap-only updates elsewhere.

**Coverage gaps:**

- The `@Patch(':caseId/priority')` controller route itself has no end-to-end HTTP test — only the service layer is unit-tested. Given the tenant-safety concern in the service (correctly scoping by JWT `tenantId`), a controller-level test that submits a foreign `caseId` would be worth having.
- No test exercises the `PriorityChangeResult` serialisation into the actual JSON response body.
- `SlaEscalationRecord`'s `P2002` swallow path is covered by unit test; concurrency in production is not (nor practical to test end-to-end).

**PR checklist status:**
- [x] Locally
- [ ] Development Environment ← not checked; note above about narrative
- [x] Husky successfully run
- [x] Unit tests passing and Documentation done

No coverage screenshot or CI coverage report is attached to the PR — CI reports build + tests green.

[↑ Back to top](#pr-review-cms-234--feat-implemented-sla-functionality)

---

## CodeRabbit Activity

### Pass 1 — initial review of head at `fc52c64` (2026-07-06 09:36 UTC)

**Findings:** 4 actionable + 1 outside-diff comment

| Finding | Severity | Status |
|---|---|---|
| `caseSupportChase` interpolates `${data.assignee}` with no fallback (vs `caseSlaBreached` which uses `?? 'Unassigned'`) | 🟡 Minor | ✅ Resolved — head uses `?? 'Unassigned'` |
| `Promise.all` short-circuits — one case's error aborts the whole batch and hides the completion log | 🟠 Major | ✅ Resolved — switched to `Promise.allSettled` + per-failure log + throw first rejection |
| `findOpenCasesForSlaCheck` unbounded — could load full open-case set into memory | 🟠 Major → downgraded after discussion | ⚠️ Deferred (author response accepted by CodeRabbit; acknowledged for future work) |
| `ManualTriageModal.tsx:29` — priority filter shows a hardcoded value that no longer exists | 🟠 Major | ✅ Resolved in the enum-swap follow-up |

### Pass 2 — nitpick pass (2026-07-06 10:03 UTC)

| Finding | Severity | Status |
|---|---|---|
| `sla-policy.util.ts:39-44` — redundant duplicate query when `tenantId === DEFAULT_TENANT_KEY` | 🔵 Trivial | ⚠️ Not resolved — see Issue 3. Author has not commented; CodeRabbit itself labelled it low-value. |

### Pass 3 — after subsequent commits (2026-07-06 13:06 UTC)

| Finding | Severity | Status |
|---|---|---|
| `usePriorityThresholds.ts:41-48` — `calculatePriority` recreated every render, exhaustive-deps footgun | 🔵 Trivial | ⚠️ Not resolved — see Issue 2 |
| `ManualTriageModal.tsx` / `CreateCaseModal.tsx` — duplicate threshold-loading logic across two modals | 🟠 Major | ✅ Resolved — shared `usePriorityThresholds` hook extracted |
| `alert-priority.service.ts:74` — `throw failures[0].reason` fails `@typescript-eslint/only-throw-error` (typed as `any`) | 🟠 Major | ✅ Resolved in head — wrapped in `instanceof Error ? … : new Error(String(reason))` |

Ibad's inline replies acknowledge and justify each accepted change; CodeRabbit's follow-up messages confirm the fixes and, in Pass 1, explicitly withdraw the pagination request after the workload-bounding argument.

[↑ Back to top](#pr-review-cms-234--feat-implemented-sla-functionality)

---

## Summary and Verdict

**Verdict: Approve with minor cleanup requested**

This is a large, well-executed PR. The domain modelling is thoughtful (derived-not-stored SLA state, three-level tenant fallback, `sla_started_at` as the clock anchor rather than `created_at`), the migrations carry unusually good "why" documentation, the escalation ledger is idempotent by construction, and test breadth is genuinely high for the new domain. The two-phase enum rename is exactly the pattern PostgreSQL forces you into, and it's implemented correctly.

CI is fully green — including both DCO and GPG-verify — and all 30 commits are signed and DCO-attributed. There are no blocking issues in the diff itself. The remaining items are cleanups (an outstanding CodeRabbit nit on `usePriorityThresholds`, a `DEFAULT_TENANT_KEY` duplicate-query micro-optimisation, and a commented-out block in `case.service.ts`) and one process observation (the PR description could benefit from a manual-testing narrative given the scope).

Since `mergeable_state: blocked` reflects only branch protection (needs a human approval), and no human reviewer has yet approved, the practical gate here is a human review, not a code change. Once one of the cleanups above is addressed (or explicitly deferred) and someone approves, this is ready to merge.

### Blocking

None.

### Non-blocking but recommended

1. **Memoise `usePriorityThresholds.calculatePriority` with `useCallback`** — see Issue 2. Small change; prevents an exhaustive-deps footgun before someone trips it.
2. **Short-circuit the tenant lookup when `tenantId === DEFAULT_TENANT_KEY` in `SlaPolicyUtil`** — see Issue 3. Trivial change; removes one wasted query per call for the DEFAULT tenant.
3. **Delete the commented-out alert-update block in `case.service.ts`** — see Issue 5. Either delete or leave a one-line "paused because X" note.
4. **Add a controller-level HTTP test for `PATCH /:caseId/priority`** — see Test Coverage. Covers the JWT-scoped tenant safety at the layer where it matters.
5. **Expand the PR description with a manual-testing narrative** — see Issue 6.

[↑ Back to top](#pr-review-cms-234--feat-implemented-sla-functionality)

---

## GitHub Review Comment

````markdown
**Approve with minor cleanup requested**

Large, well-executed PR. The SLA domain modelling is thoughtful (derived-not-stored `sla_state`, three-level tenant fallback, `sla_started_at` as the clock anchor), the enum-rename migrations are correctly split into two phases with excellent commentary, and the escalation ledger is idempotent by construction. All 30 commits are DCO-signed and GPG-verified; CI is fully green. No blocking issues in the diff itself — the remaining items are cleanups.

---

### Non-blocking (please address in this PR if possible)

**1. Memoise `usePriorityThresholds.calculatePriority`**

`calculatePriority` is currently recreated every render, which is an exhaustive-deps footgun for any consumer that puts it in a `useEffect` / `useMemo` dep array.

```ts
const calculatePriority = useCallback(
  (score: number): Priority => {
    if (!thresholds) return Priority.LOW;
    if (score >= thresholds.highThreshold) return Priority.HIGH;
    if (score >= thresholds.mediumThreshold) return Priority.MEDIUM;
    return Priority.LOW;
  },
  [thresholds],
);
```

**2. Short-circuit the `DEFAULT_TENANT_KEY` case in `SlaPolicyUtil.getTargetSeconds` / `getEscalationRatios`**

When the caller's tenant is literally `'DEFAULT'`, both the tenant-specific lookup and the fallback lookup hit the same row. Skip the first query when they're identical:

```ts
async getTargetSeconds(tenantId: string, priority: Priority): Promise<number> {
  if (tenantId !== DEFAULT_TENANT_KEY) {
    const tenantPolicy = await this.prisma.slaPolicy.findFirst({ where: { tenant_id: tenantId, priority } });
    if (tenantPolicy) return tenantPolicy.target_seconds;
  }
  const defaultPolicy = await this.prisma.slaPolicy.findFirst({ where: { tenant_id: DEFAULT_TENANT_KEY, priority } });
  if (defaultPolicy) return defaultPolicy.target_seconds;
  return FALLBACK_TARGET_SECONDS[priority];
}
```

Apply the same guard to `getEscalationRatios` and to `CasePriorityUtil.getThresholds`.

**3. Remove the commented-out alert-update block in `case.service.ts`**

Around lines 1104–1116 there's a ~10-line commented-out `alertRepository.updateAlert` block from the old flow. Either delete it, or leave a one-line note explaining why it's paused and what would re-enable it — the block as-is is drift-prone.

**4. Add a controller-level HTTP test for `PATCH /:caseId/priority`**

The service layer is well-tested but the controller wiring — specifically that `tenantId` is pulled from the JWT and not the body — isn't exercised end-to-end. A single spec that submits a `caseId` belonging to a different tenant and asserts a 404 would cover the tenant-safety contract at the layer where it matters.

**5. Expand the PR description**

Given the scope of the change and the operational impact of the cron, a few lines on which manual scenarios were exercised (SLA breach → notification → ledger → no re-notify; priority change → `sla_due_at` re-anchor; DEFAULT-tenant fallback path) would help future reviewers verify. The checklist alone doesn't say what was actually walked through.
````

[↑ Back to top](#pr-review-cms-234--feat-implemented-sla-functionality)
