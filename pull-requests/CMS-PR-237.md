# PR Review: CMS #237 — Paysys/sla task backup

**Repo:** tazama-lf/case-management-system
**Branch:** `paysys/SLA-task-backup` → `dev`
**Author:** ibadkhan088 (Ibad Ahmed Khan)
**Date Reviewed:** 2026-07-08
**Label:** _(none)_
**Size:** +5,602 / −3,926 lines across 160 files
**Commits:** 38 (`b315e59f`, `b8567a1b`, `84c80db1`, `686e9912`, `587fd3c1`, `cdb83154`, `89348ed2`, `632d3cf5`, `d61a4493`, `5972f7e6`, `88f02e55`, `9ad6fdbb`, `f63eadfd`, `50097f0f`, `3f512122`, `ff85a86f`, `ef56d59a`, `502d0028`, `0a6d35f6`, `0d836a6d`, `1f78ebc4`, `f2528ff5`, `ff17b7f6`, `25c62dc`, `f5d50fc7`, `56cce1cb`, `a14c45d1`, `81c8def1`, `5eeb4adb`, `dd7b31d0`, `c06a3a76`, `d6f7386c`, `c0e2ff7e`, `e97101e5`, `4c561a4c`, `f5c856e3`, `afd1b216`, `30881f94`)
**State:** OPEN (mergeState: DIRTY — conflicts against `dev`)
**Existing approvals:** none — 0 reviews recorded on this PR
**CI:** CodeQL (Analyze/actions, Analyze/js-ts) SUCCESS · CodeRabbit SUCCESS (review skipped due to 158-file limit)
**Created:** 2026-07-08 (branch backup of long-running `paysys/SLA-task`)

---

## Table of Contents

- [Overview](#overview)
- [DCO and Signature Verification Analysis](#dco-and-signature-verification-analysis)
  - [Current State](#current-state)
  - [Why DCO/Verification Failed Earlier](#why-dcoverification-failed-earlier)
  - [Offending Commits and Root Causes](#offending-commits-and-root-causes)
  - [Recommendations to Prevent Recurrence](#recommendations-to-prevent-recurrence)
- [What Changed (High-Level)](#what-changed-high-level)
  - [Backend — Schema & Migrations](#backend--schema--migrations)
  - [Backend — Alert / Case / SLA Domain](#backend--alert--case--sla-domain)
  - [Backend — Tests](#backend--tests)
  - [Frontend — Alerts & Cases UI](#frontend--alerts--cases-ui)
  - [Frontend — Shared / Tests](#frontend--shared--tests)
  - [CI / Config](#ci--config)
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

This PR is a "backup" branch (`paysys/SLA-task-backup`) of the long-running `paysys/SLA-task` feature branch, opened against `dev` on 2026-07-08. It bundles a large set of changes across three concerns:

1. **SLA / Priority domain rewrite** — replaces the legacy four-value priority enum (`NEW`/`URGENT`/`CRITICAL`/`BREACH`) with a severity enum (`LOW`/`MEDIUM`/`HIGH`), stamps an absolute `sla_due_at` on every case at creation, computes `sla_state` dynamically (never persisted), records `PriorityChanged` audit events, and makes SLA escalations and priority thresholds tenant-configurable. Includes 12 new Prisma migrations.
2. **Alert search / triage fixes** — 7+ commits titled "fix: fixed the alert search …" spread across late June / early July.
3. **Merged-in feature work from other authors** — PR #233 (FRAUD_AND_AML container case → `InvestigationGroup` linkage), PR #235 (RC Docker workflow Slack token/matrix fix), Sobia Rizwan's Flowable + visualization + currency + test fixes.

The PR description is empty (all checklist items unchecked, no "What/Why/How tested" content). The branch is currently **conflicting with `dev`** (`mergeStateStatus: DIRTY`) and has zero human reviews recorded. CodeRabbit skipped the review because the PR exceeds its 150-file soft limit (158 processed files).

| File group | Nature of change |
|------|-----------------|
| `backend/prisma/migrations/2026070*` (12 files) | New migrations: investigation_group, priority→severity rename, drop BREACH, sla_due_at, sla_started_at, escalation records, thresholds, target seconds unit conversion |
| `backend/prisma/schema.prisma` | Schema updates to match all new migrations |
| `backend/src/modules/alert-priority/*` | New module: `case-priority.service.ts`, `sla-state.util.ts`, `alert-priority.task.ts` |
| `backend/src/modules/case/services/*` | Case creation/closure/reopening/query services updated for new priority/SLA model |
| `backend/src/modules/investigation-group/*` | New investigation-group module (from merged PR #233) |
| `backend/src/modules/shared/utils/{case-priority,sla-policy}.util.ts` | New shared utilities for tenant-configurable priority/SLA computation |
| `backend/src/utils/rbac/permissionMatrix.json` | RBAC permission updates for new endpoints/actions |
| `backend/test/*` (20 spec files) | New/updated tests for the domain rewrite |
| `frontend/src/features/alerts/**` | Alert search fixes, triage modal updates, dashboard updates |
| `frontend/src/features/cases/**` | Case UI updates for new priority enum, SLA badge, visualization tab, priority-change flow |
| `frontend/src/shared/components/ui/SlaStateBadge.tsx` | New UI component for SLA state display |
| `frontend/src/shared/constants/casePriorityThresholds.ts` | Frontend constants for the new priority model |
| `frontend/src/shared/hooks/usePriorityThresholds.ts` | Hook to consume tenant-configurable thresholds |
| `.github/workflows/dockerhub-image-build-dual-rc.yml` | RC Docker matrix + Slack token fix (from merged PR #235) |
| `README.md`, `backend/.env.example` | Documentation + env additions for SLA/thresholds config |

[↑ Back to top](#pr-review-cms-237--paysyssla-task-backup)

---

## DCO and Signature Verification Analysis

### Current State

At the time of this review (2026-07-08), **all 38 commits in the PR head report `verified: true` with `reason: valid` from GitHub's API**, and all commits carry a trailing `Signed-off-by: Ibad Khan <ibad.ahmed@paysyslabs.com>` line — so DCO and signature checks are passing *now*. The issues the author asked about are historical and were resolved by two remediation commits in the PR history.

### Why DCO/Verification Failed Earlier

Two remediation commits in the PR history are the smoking gun:

| Commit | Author date | Committer date | Message |
|--------|-------------|----------------|---------|
| `502d0028` | 2026-07-06 07:03:56 UTC | 2026-07-06 07:09:22 UTC | **fix: fixed the DCO issue** |
| `0a6d35f6` | 2026-07-06 08:58:07 UTC | 2026-07-06 08:58:07 UTC | **fix: fixed the BOM issues** |

The pattern that explains both:

**1. Batch rewrite / interactive rebase on 2026-07-06 ~07:09 UTC.** Looking at the first 27 commits (Ibad's original work, authored between 2026-06-23 and 2026-07-06 07:03), *every one of them* has a committer date within a 23-second window: `2026-07-06T07:08:59Z` through `2026-07-06T07:09:22Z`. Original author dates span two weeks; committer dates cluster in one batch. This is the fingerprint of an interactive rebase (`git rebase -i` with `exec git commit --amend -s -S` or similar) that:

  - Re-signed every commit with a working GPG/SSH key (bringing them from unverified → verified), and
  - Added the missing `Signed-off-by:` trailer (bringing them from DCO-failing → DCO-passing).

  Any commit made *before* 2026-07-06 07:09 UTC in the local branch, if pushed as-is, would have failed either the DCO check (no `Signed-off-by`) or the "verified" badge (no signature) or both — hence the `fix: fixed the DCO issue` commit at the tail end of that batch.

**2. BOM (byte-order mark) contamination.** The follow-up commit `0a6d35f6` — "fix: fixed the BOM issues" — is a classic DCO-breaker. Some editors (Notepad, older VS Code on Windows, and some Windows Git clients) prepend a UTF-8 BOM (`0xEF 0xBB 0xBF`) to files or to commit messages. If the BOM lands ahead of the `Signed-off-by:` trailer, the DCO bot parses the line as `﻿Signed-off-by:` — which does not match its regex, and the check fails. The commit title strongly suggests this is what happened to one or more commits between `502d0028` and `0a6d35f6` (a ~2-hour window on 2026-07-06).

**3. Cherry-picking / merging other authors' work strips signatures.** Commits `81c8def1`, `5eeb4adb`, `dd7b31d0`, `c06a3a76`, `d6f7386c`, `c0e2ff7e`, `e97101e5`, `4c561a4c`, `f5c856e3`, `afd1b216` all show a mismatch between **author** (Kyle Vorster / Muhammad Ali / KyleVorster7 / Justus Ortlepp / Sobia Rizwan) and **committer** (Ibad Khan). This means Ibad either merged their branches locally or cherry-picked their commits. When a signed commit is cherry-picked or replayed, the *original signature no longer covers the new tree/parent hash* — so it's stripped, and unless the person doing the replay re-signs, the resulting commit is unverified. Every one of these commits carries **both** `Signed-off-by: <original-author>` **and** `Signed-off-by: Ibad Khan …` — evidence that Ibad had to manually add his own sign-off during the replay. If he did that with `git cherry-pick -s` but *without* `-S`, the DCO would pass but the "Verified" badge would not appear.

### Offending Commits and Root Causes

| # | Commit | Author date | Original problem (inferred) | How it was fixed |
|---|--------|-------------|------------------------------|------------------|
| 1–27 | `b315e59f` → `ef56d59a` | 2026-06-23 → 2026-07-06 01:04 (Ibad) | Original commits made without `-s` (no DCO trailer) and/or without `-S` (no signature) | Batch rebase on 2026-07-06 07:09 UTC re-signed and added `Signed-off-by` to all 27 |
| 28 | `502d0028` | 2026-07-06 07:03 | (this is the DCO-fix marker itself) | Marks the completion of the rebase |
| 29 | `0a6d35f6` | 2026-07-06 08:58 | UTF-8 BOM prepended before `Signed-off-by:` trailer, breaking DCO regex | Author re-committed after stripping the BOM |
| 30 | `81c8def1` | 2026-07-07 20:17 | Kyle Vorster's commit, cherry-picked/merged by Ibad — original signature invalidated | Ibad re-signed (`-S`) with own key and added second `Signed-off-by` |
| 31 | `5eeb4adb` | 2026-07-07 12:30 | Muhammad Ali's commit — same as above | Same remediation |
| 32 | `dd7b31d0`, `c06a3a76` | 2026-07-07 20:55, 23:44 | Merge commits for PRs #235, #233 — replayed onto Ibad's branch | Re-signed by Ibad |
| 33–37 | `d6f7386c` → `afd1b216` | 2026-07-08 | Sobia Rizwan's commits — cherry-picked | Ibad re-signed each |

Note: the local `git log` shows `%G? = E` ("missing key") for every commit because the workstation running this review does not have the signers' SSH keys configured locally (`gpg.ssh.allowedSignersFile needs to be configured and exist for ssh signature verification`). GitHub's API, which trusts keys registered on each user's GitHub account, reports `verified: true` for all 38 commits — so the CI-side DCO and verification checks are currently green.

### Recommendations to Prevent Recurrence

1. **Configure git globally to always sign and sign off**, so no commit ever slips through without both:
   ```bash
   git config --global commit.gpgsign true
   git config --global format.signoff true   # or use `-s` in each commit
   git config --global user.signingkey <your-ssh-key-path>
   ```
   Also set `gpg.format` to `ssh` if you sign with an SSH key.

2. **Enable a pre-commit / pre-push hook** that rejects commits missing a valid `Signed-off-by:` trailer matching `user.name <user.email>` — this catches BOM contamination too, because the exact-match regex fails on `﻿Signed-off-by:`.

3. **Never let a Windows editor touch commit messages.** BOMs are almost always Windows-editor artifacts. Set the editor to LF, UTF-8 without BOM. In VS Code: `"files.encoding": "utf8"`, `"files.eol": "\n"`.

4. **When cherry-picking / merging someone else's commits, use `git cherry-pick -s -S <sha>`** so both DCO and signature are preserved. Or better, use `git merge --ff-only` where possible (preserves the original author's signature verbatim).

5. **Avoid the "cherry-pick everyone into my PR" pattern.** Ten of this PR's 38 commits are third-party author work replayed by Ibad. This is the primary source of the recurring verification failures. Prefer stacking their PRs into `dev` first and rebasing this branch onto `dev`, which lets the original signatures verify natively.

6. **Rebase / re-sign in one shot at the end.** If a batch fix is unavoidable, do it once (as was done on 2026-07-06), don't repeat it — every rewrite forces reviewers to re-review from scratch. The 2026-07-06 07:09 batch was correct in principle but revealed that this workflow had never been in place from day one.

[↑ Back to top](#pr-review-cms-237--paysyssla-task-backup)

---

## What Changed (High-Level)

> This section is intentionally high-level per the reviewer's directive. Detailed per-file diffs are not included because CodeRabbit skipped the review (158-file limit) and a line-by-line audit of a 16,937-line diff is out of scope for this pass. See [Summary and Verdict](#summary-and-verdict) for the recommendation on decomposing this PR before merge.

### Backend — Schema & Migrations

- 12 new Prisma migrations dated between 2026-07-01 and 2026-07-07 covering: `investigation_group` table + relations, priority→severity rename, dropping the `BREACH` enum value, adding `Case.sla_due_at` and `Case.sla_started_at`, removing `Case.status = COMPLETED`, adding `sla_escalation_records`, `sla_escalation_thresholds`, `case_priority_thresholds`, and converting SLA policy target from hours to seconds.
- `backend/prisma/schema.prisma` updated to match.
- `backend/prisma/scripts/backfill-investigation-groups.sql` — one-off backfill script.

### Backend — Alert / Case / SLA Domain

- New module `backend/src/modules/alert-priority/` with `alert-priority.service.ts`, `alert-priority.task.ts` (cron/scheduled), `case-priority.service.ts`, `sla-state.util.ts`.
- New module `backend/src/modules/investigation-group/` (from merged PR #233).
- New shared utils: `backend/src/modules/shared/utils/case-priority.util.ts`, `sla-policy.util.ts`.
- Reworked services: `case.service.ts`, `case-creation.service.ts`, `case-closure-approval.service.ts`, `case-creation-approval.service.ts`, `case-reopening.service.ts`, `case-query.service.ts`, `alert.service.ts`, `alert.statistics.service.ts`, `notification.service.ts`, `report.service.ts`, `task.service.ts`, `task-lifecycle.service.ts`, `triage.service.ts`.
- New/updated DTOs and enums for the new priority/severity model.
- `backend/src/utils/rbac/permissionMatrix.json` — permission entries for the new endpoints.
- Audit interceptor updated (`interpectors/audit-log.interceptor.ts` — note the pre-existing typo `interpectors` → `interceptors`).
- `backend/src/modules/bpmn/cms.bpmn20.xml` updated.

### Backend — Tests

- 20 spec files added or updated under `backend/test/`. Coverage includes: alert-priority, alert, alert.statistics, case-priority, case-priority.util, case-query, case-reopening, case-closure-approval, case-creation, case-creation-approval, case, case.repository, flowable, investigation-group, rbac, report, sla-policy.util, sla-state.util, task, task-lifecycle, triage.

### Frontend — Alerts & Cases UI

- Alerts: `AlertDetails.tsx`, `AlertsDetailModal.tsx`, `AlertsSearchAndFilters.tsx`, `ManualTriageModal.tsx`, `useAlertsQuery.ts`, `AlertsDashboard.tsx`, `alertTransformers.ts`, `triage.types.ts`.
- Cases: `CaseFilters.tsx` (both dashboard and main), `CaseModalsManager.tsx`, `CasesTable.tsx`, `CloseCaseModal.tsx`, `CreateCaseModal.tsx`, `TasksDetailsModal.tsx`, `ViewCaseModal.tsx`, `CaseActionsPanel.tsx`, `CaseDetailsTab.tsx`, `InvestigationsSummaryTab.tsx`, `LinkedItemsTab.tsx`, `TaskDetailsTab.tsx`, `TaskLogTab.tsx`, `TransactionDetailsTab.tsx`, `useCase.ts`, `caseService.ts`, `casesTable.utils.ts`.
- New: `frontend/src/shared/components/ui/SlaStateBadge.tsx`, `frontend/src/shared/constants/casePriorityThresholds.ts`, `frontend/src/shared/hooks/usePriorityThresholds.ts`.

### Frontend — Shared / Tests

- `dashboard.types.ts` + tests updated for new enum.
- Test mocks: `caseHandlers.ts`, `server.ts`, `testUtils.tsx`, `triageServiceTest.ts`.

### CI / Config

- `.github/workflows/dockerhub-image-build-dual-rc.yml` — from merged PR #235.
- `backend/.env.example` — new SLA / threshold env vars.
- `README.md` — documentation update.

[↑ Back to top](#pr-review-cms-237--paysyssla-task-backup)

---

## Code Quality Analysis

### Strengths

- **Domain modelling is sound in principle.** Replacing a mixed priority-and-state enum (`NEW`/`URGENT`/`CRITICAL`/`BREACH`) with a clean severity enum (`LOW`/`MEDIUM`/`HIGH`) and deriving SLA state from a persisted absolute deadline (`sla_due_at`) is a well-known correct pattern — state is a function of time, so it should not be stored.
- **`PriorityChanged` audit events** — the right choice for regulated case management. Priority changes must be attributable.
- **Tenant-configurable thresholds** — the shift from hard-coded thresholds to `case_priority_thresholds` + `sla_escalation_thresholds` tables is a real correctness win for multi-tenant deployments.
- **Test coverage keeps pace.** 20 spec files updated/added is proportionate to the scope of the domain change.

### Issues and Observations

#### Issue 1 — PR is too large to review responsibly

**Severity: Major (Process / Reviewability)**

At 5,602 additions / 3,926 deletions across 160 files, this PR exceeds CodeRabbit's file limit (which is why CodeRabbit skipped the review — the "SUCCESS" status is misleading, no review was actually performed) and is impractical for a human reviewer to audit in a single pass. It also bundles at least four independent concerns:

1. SLA / priority domain rewrite (the "real" feature),
2. Alert search fixes (7+ commits, likely a separate bug batch),
3. Cherry-picked work from PRs #233 (Justus / FRAUD_AND_AML) and #235 (Kyle / RC Docker workflow),
4. Sobia Rizwan's Flowable / currency / visualization / test fixes.

Concerns 3 and 4 are especially problematic because they either duplicate PRs that are being merged separately, or they mask independent work behind Ibad's authorship for review purposes. **Recommendation: split into 3–4 PRs before requesting review.**

#### Issue 2 — PR description is empty

**Severity: Major (Process)**

The PR body is the template with every checkbox unticked and no content under **What did we change?**, **Why are we doing this?**, or **How was it tested?**. For a 5,600-line change this is unacceptable — reviewers cannot even determine the scope without reading commit messages, and the "Locally / Development Environment / Unit tests passing" boxes are all unchecked.

#### Issue 3 — Branch is in conflict with `dev`

**Severity: Major (Blocking merge)**

`mergeStateStatus: DIRTY`, `mergeable: CONFLICTING`. Must be resolved before merge. Given the branch cherry-picks from PRs #233 and #235 which are also targeting `dev`, the conflicts likely stem from those two PRs having been (or being) merged independently — meaning some or all of the third-party commits in this branch may become duplicates.

#### Issue 4 — Zero human reviews recorded

**Severity: Major (Governance)**

No reviewer has approved this PR. The passing CI signals (CodeQL + CodeRabbit) are not a substitute for human review, and — as noted — CodeRabbit did not actually review the diff.

#### Issue 5 — Commit history has been rewritten mid-PR

**Severity: Minor (Process)**

The 2026-07-06 07:09 UTC batch re-signing (see [DCO Analysis](#dco-and-signature-verification-analysis)) is not itself wrong, but combined with the pattern of cherry-picking other authors' commits, it means the PR history no longer cleanly attributes work to the person who wrote it in a way that a signature verifier can confirm. Preserve upstream authorship by rebasing this branch onto `dev` *after* #233 and #235 land, rather than replaying their commits.

#### Issue 6 — `backend/src/interpectors/` typo (pre-existing)

**Severity: Informational**

The directory name `interpectors` (should be `interceptors`) is not introduced by this PR — it exists on `dev` — but this PR touches `audit-log.interceptor.ts` inside it and would be a low-risk moment to fix the typo in a separate small PR.

#### Issue 7 — 12 migrations in one PR

**Severity: Minor (Data Integrity)**

Twelve schema migrations dated across seven days is a lot to land in a single deploy, especially when one of them (`20260702120000_remove_case_status_completed/migration.sql`) drops an enum value and another (`20260706120000_sla_policy_target_seconds/migration.sql`) changes the unit of an existing column from hours to seconds. Any misordering (e.g., in a rollback scenario) could corrupt in-flight SLA data. Confirm before merge:

- Order-of-execution is deterministic (Prisma migrations run in filename order — the timestamps look correct).
- The `hours → seconds` unit conversion migration includes an `UPDATE` that multiplies existing values by 3600, not just a column-type change.
- There is a rollback plan for the `BREACH` enum drop if any existing row still carries that value.

[↑ Back to top](#pr-review-cms-237--paysyssla-task-backup)

---

## Security Assessment

Not audited in this pass — the diff is too large for a responsible line-by-line security review, and the PR touches auth-adjacent surfaces (`permissionMatrix.json`, audit interceptor, case-creation approvals). This should be re-reviewed after decomposition.

| Concern | Assessment |
|---------|-----------|
| RBAC changes (`permissionMatrix.json`) | Not audited — needs a dedicated pass to confirm no new endpoints are exposed without appropriate role gates |
| Audit interceptor changes | Not audited — must confirm `PriorityChanged` events include actor identity and cannot be suppressed |
| Notification service changes | Not audited |
| SQL migrations | Not audited for injection risk in `backfill-investigation-groups.sql` |
| Cherry-picked commits from other authors | Signatures re-generated by Ibad — the original authors' cryptographic attestation is lost; the code should be verified against the original PRs (#233, #235) diff-for-diff before trust is granted |

**Statement:** _Security review deferred — this PR must be decomposed before a meaningful security assessment can be performed._

[↑ Back to top](#pr-review-cms-237--paysyssla-task-backup)

---

## Test Coverage

- 20 backend spec files added/updated — proportionate to backend scope.
- Frontend tests updated for all touched components (`__tests__` folders throughout).
- **Checklist in PR description: all four boxes unchecked** — no attestation from the author that the change was tested locally, in dev, that Husky passed, or that unit tests pass.
- **CI evidence:** CodeQL passes; no unit-test workflow status is visible in the status check rollup — this needs confirmation before merge that unit tests actually ran.
- **Integration coverage for the SLA cron/task** (`alert-priority.task.ts`) is not visible in the diff — a scheduled task that mutates state deserves an end-to-end test that ticks the clock and asserts an escalation record was created.

[↑ Back to top](#pr-review-cms-237--paysyssla-task-backup)

---

## CodeRabbit Activity

CodeRabbit **skipped** its review with the message:

> Too many files! This PR contains 158 files, which is 8 over the limit of 150.

The `CodeRabbit: SUCCESS` status in the CI rollup does **not** indicate that CodeRabbit reviewed the code — it indicates the CodeRabbit status check completed without error. No findings were produced. This is worth flagging on the PR because a passing "CodeRabbit" signal may lead reviewers to assume the diff has been machine-audited when it has not.

[↑ Back to top](#pr-review-cms-237--paysyssla-task-backup)

---

## Summary and Verdict

**Verdict: Changes Requested**

The domain rewrite (priority→severity, `sla_due_at`, tenant-configurable thresholds, priority-change audit events) is a genuine improvement to the case-management model and the tests appear to move with the code. However, this PR is not mergeable in its current form for reasons that are structural rather than about the code itself: it bundles four unrelated concerns, exceeds CodeRabbit's review limit (so the "green" CodeRabbit status is misleading), has an empty PR description, has zero human reviews, and is currently conflicting with `dev`. The DCO/verification failures the author was seeing earlier have been resolved by the 2026-07-06 07:09 batch re-signing and the BOM fix, and are unlikely to recur if the author adopts the workflow changes in the DCO section.

### Blocking

1. **PR is too large to review responsibly** — split into (a) SLA/priority domain rewrite, (b) alert search fixes, (c) frontend visualization/currency work, (d) any residual after #233 and #235 land. See Issue 1.
2. **Empty PR description** — populate the What/Why/How-tested sections and tick the appropriate checklist boxes. See Issue 2.
3. **Merge conflicts with `dev`** — rebase (do not merge) once #233 and #235 are in `dev`. See Issue 3.
4. **Zero human reviews on a 5,600-line change** — requires at least one domain owner review before merge. See Issue 4.
5. **Twelve migrations in one PR** — confirm the `hours → seconds` migration performs the `UPDATE * 3600` and that the `BREACH` enum drop is safe against existing rows. See Issue 7.

### Non-blocking but recommended

6. **Adopt the DCO workflow fixes** in the [Recommendations](#recommendations-to-prevent-recurrence) section — global `commit.gpgsign`, `format.signoff`, pre-commit hook, UTF-8-without-BOM editor settings, and cherry-pick with `-s -S`.
7. **Fix the pre-existing `interpectors` typo** in a small separate PR. See Issue 6.
8. **Add an end-to-end test for `alert-priority.task.ts`** that ticks the clock and asserts an SLA escalation record is created. See [Test Coverage](#test-coverage).

[↑ Back to top](#pr-review-cms-237--paysyssla-task-backup)

---

## GitHub Review Comment

````markdown
**Changes Requested**

The SLA/priority domain rewrite is a real improvement, but this PR isn't mergeable as-is: it bundles four unrelated concerns across 160 files (5,602 additions / 3,926 deletions), the PR body is the empty template, it currently conflicts with `dev`, and it has zero human reviews. The green `CodeRabbit` status is misleading — CodeRabbit **skipped** the review because the file count exceeds its 150-file limit, so no automated review actually occurred.

---

### Blocking

**1. Decompose the PR before requesting review.**

The diff bundles at least four independent concerns:
- SLA / priority domain rewrite (12 migrations, `alert-priority` module, `sla-state.util`, tenant-configurable thresholds) — this is the "real" feature.
- Alert search fixes (7+ commits titled "fix: fixed the alert search …").
- Cherry-picked commits from PR #233 (FRAUD_AND_AML → InvestigationGroup) and PR #235 (RC Docker workflow).
- Sobia Rizwan's Flowable / currency / visualization / test fixes.

Please split into 3–4 focused PRs. The PRs #233 and #235 commits should not be re-included here — rebase onto `dev` after they land instead.

**2. Fill in the PR description.**

The body is the empty template with every checkbox unticked. For a 5,600-line change, reviewers need at minimum:
- What changed (bullet list of the domain rewrite, migrations, and each independent concern).
- Why (link to the SLA/priority ticket).
- How it was tested — and tick the checklist boxes honestly.

**3. Resolve merge conflicts with `dev`.**

`mergeStateStatus: DIRTY`. Likely stems from PRs #233 and #235 being merged separately. After they land, rebase this branch and drop the duplicated commits.

**4. Obtain at least one domain-owner review before merge.**

No human reviews are recorded. CodeQL + a skipped CodeRabbit are not sufficient governance for a change of this size.

**5. Confirm migration safety.**

Twelve migrations in one PR is aggressive. Please confirm before merge:
- `20260706120000_sla_policy_target_seconds/migration.sql` includes an `UPDATE` that multiplies existing hour values by 3600, not just a column-type/name change.
- Dropping `BREACH` from the priority enum (`20260702100100_priority_severity_drop_breach`) is safe against any existing row still carrying that value — either the backfill migration converts them first, or there is a documented pre-deploy check.
- The Prisma migration order (by filename timestamp) is what you actually want.

---

### DCO / Verification — root-cause summary (as requested)

All 38 commits currently pass DCO and show `verified: true` on GitHub. The failures you were seeing earlier came from three distinct causes, all of which are now fixed but worth understanding so they don't recur:

**(a) Original commits made without `-s` / `-S`.** The first 27 commits (`b315e59f` → `ef56d59a`, authored 2026-06-23 → 2026-07-06 01:04) were batch-rewritten in a single interactive rebase — every committer date falls in a 23-second window at `2026-07-06T07:08:59Z–07:09:22Z`. This was the moment the missing `Signed-off-by` trailers and SSH signatures were added retroactively. The commit `502d0028` — literally titled "fix: fixed the DCO issue" — marks the tail of that batch.

**(b) UTF-8 BOM contamination.** Commit `0a6d35f6` ("fix: fixed the BOM issues", 2026-07-06 08:58 UTC) fixed a case where a Windows editor prepended a UTF-8 BOM (`0xEF 0xBB 0xBF`) ahead of the `Signed-off-by:` trailer, breaking the DCO regex.

**(c) Cherry-picked / merged commits from other authors.** Commits `81c8def1` (Kyle), `5eeb4adb` (Muhammad Ali), `dd7b31d0` and `c06a3a76` (merge commits for PRs #235 and #233), and `d6f7386c` / `c0e2ff7e` / `e97101e5` / `4c561a4c` / `f5c856e3` / `afd1b216` (Sobia) all show a mismatch between author (someone else) and committer (Ibad). Cherry-picking a signed commit invalidates the original signature because the tree/parent hashes change. If replayed without `-S`, the commit shows as unverified even though DCO passes.

**Prevent recurrence:**

```bash
git config --global commit.gpgsign true
git config --global format.signoff true
git config --global user.signingkey <your-ssh-key-path>
git config --global gpg.format ssh
```

And when replaying someone else's commits: `git cherry-pick -s -S <sha>`. Or, better, don't cherry-pick — let their PR land in `dev` first and rebase yours on top, which preserves their original signature natively.

Editor: use UTF-8 **without BOM** for commit messages. In VS Code: `"files.encoding": "utf8"`, `"files.eol": "\n"`.

---

### Non-blocking (please address in this PR if possible)

**6. Add an end-to-end test for `alert-priority.task.ts`.**

A scheduled task that mutates SLA state deserves a test that ticks the clock and asserts an `sla_escalation_records` row is created and a notification is emitted. I don't see one in `backend/test/` — please add it.

**7. Fix the pre-existing `backend/src/interpectors/` typo** in a separate small PR (not this one) — should be `interceptors`. You're touching a file inside that folder, so it's a good moment to file the follow-up.

**8. Flag the misleading `CodeRabbit: SUCCESS` status** — CodeRabbit skipped the review due to the 150-file limit, so the green check does not mean the diff has been machine-audited. A short comment on the PR noting this will save reviewers time.
````

[↑ Back to top](#pr-review-cms-237--paysyssla-task-backup)
