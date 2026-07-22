# Full-Stack-Docker-Tazama (FSDT) — Pre-Release Gaps Report

<a id="top"></a>

**Repository under review:** [tazama-lf/Full-Stack-Docker-Tazama](https://github.com/tazama-lf/Full-Stack-Docker-Tazama) — branch `dev`
**Local snapshot:** commit `3f88eea` (Merge PR #236, feat-paysys-nifi)
**Report generated:** 2026-07-20
**Author:** ahmad.khalid@paysyslabs.com

FSDT is the deployment repository — it carries the docker-compose stacks, env files, DDL/DML seeds, NiFi templates, and infra config that stand up the complete Tazama ecosystem. This report reviews FSDT's `dev` branch against every other Tazama repository (all synced to their respective `origin/dev`) to enumerate what is missing before the next client release. Particular focus: **rule-studio** deployability, the **CMS-220 SLA** track, the **CMS-214 Fraud & AML** track, and outstanding PR **#239**.

**Scope notes:**
- `rule-studio-devtestops` (RSDTO) is not treated as a required deployable service in this report — per project owner it is not a live service to be shipped.
- Rule-21 has been re-investigated: it **is** a real, seeded bootstrap rule that FSDT already ships. See §5.2. This corrects an earlier draft.
- **The SQL findings in earlier drafts of this document were largely wrong** and have been retracted after direct file inspection and a live-DB check against `tazama_cms` on tazama-b. See §3.1 and §6.4 for the corrected picture.

### Verification method for the CMS sections

Sections §3 and §4 are backed by a read-only `pg_dump -s` of the live `tazama_cms` database on tazama-b (production extensions postgres). The applied Prisma migrations were cross-checked against the migration folders of `origin/dev` and `origin/paysys/visualization` on `case-management-system`. Findings are stated as observed, not inferred.

---

## Executive Summary

- **PR #239** is still open on FSDT (`feat-paysys-trs-query` → `dev`, ReebaPaysys, +3/−1 lines in `core/postgres/migration/base/02-trs-init.sql`). Owner will send.
- **CMS-220 (SLA) is NOT deployable via FSDT dev today.** [extensions/.env#L24](../repos/Full-Stack-Docker-Tazama/extensions/.env#L24) pins `CMS_BRANCH=paysys/visualization`, a branch that diverged from `case-management-system/dev` **before** PR #247 (SLA, commit `fa31dec6`, 2026-07-16) and the six earlier SLA migrations landed. To be corrected in the in-flight `fixes-psl` PR.
- **CMS-214 (Fraud & AML) is NOT deployable via FSDT dev today.** Same root cause — `paysys/visualization` predates PR #233 (Fraud/AML, commit `68da7161`, 2026-07-08) which introduces `investigation_groups`, removes `STATUS_84_COMPLETED`, and adds a backfill script.
- **Rule-Studio env file is materially incomplete.** [extensions/env/trs.env](../repos/Full-Stack-Docker-Tazama/extensions/env/trs.env) provides only the base app vars, auth, admin URL, CORS, encryption keys, and OpenSearch. It is missing the **Redis block** (backend runtime dependency for BullMQ/socket.io-redis — the code reads `REDIS_HOST`/`REDIS_PORT`/`REDIS_PASSWORD` directly in [backend/src/…](../repos/rule-studio/backend/src/)), the **SMTP block** (used by [notification.service.ts](../repos/rule-studio/backend/src/services/notification/notification.service.ts) — degrades gracefully if absent; needed for the release feature-set), the **DockerHub block** (required by [dockerhub.service.ts](../repos/rule-studio/backend/src/services/simulation-studio/dockerhub/dockerhub.service.ts) — throws at first use if absent), and other simulation-studio vars (`API_HOST`, `BULL_PREFIX`, `DEMS_BASE_URL`, `DLH_URL`, `TAZAMA_REPO_BRANCH`, `WRITE_SWAGGER_JSON`). Full diff and required/optional split in §5.3.
- **Rule-21 IS shipped in FSDT** — seeded in [core/postgres/migration/base/02-trs-init.sql](../repos/Full-Stack-Docker-Tazama/core/postgres/migration/base/02-trs-init.sql) (rule metadata + rule flow rows). rule-studio's `BASE_RULE_ID='21'` (see [backend/src/constants/constant.ts](../repos/rule-studio/backend/src/constants/constant.ts)) is used by the create-rule flow at [backend/src/services/rules/rules.service.ts](../repos/rule-studio/backend/src/services/rules/rules.service.ts) L125-134 to clone rule-21's `flow_json_rule_builder` and `flow_json_test_case` as the default template for every new rule. **Verify no upstream PR has advanced the rule-21 seed content** and that a fresh FSDT bring-up executes 02-trs-init.sql (it does; ordering is intact).
- **CMS schema is 100% Prisma-owned; FSDT ships no CMS SQL.** Every `.sql` file under [core/postgres/migration/](../repos/Full-Stack-Docker-Tazama/core/postgres/migration/) is for the *core*, *config*, *TRS*, or *TCS* databases — none references `tazama_cms` or any CMS table. The CMS schema is materialised entirely by the `migrate` container which runs CMS's own `backend/migrate.sh` → `prisma migrate deploy`. The mechanism that determines which schema you get differs between dev mode and hub mode — see §3.1.
- ~~**NiFi template drift.**~~ **RETRACTED.** Direct byte-diff (after normalising CRLF vs LF) shows FSDT's [biar/nifi/tazama.xml](../repos/Full-Stack-Docker-Tazama/biar/nifi/tazama.xml) is content-identical to biar's canonical [nifi/tazama.xml](../repos/biar/nifi/tazama.xml). The SHAs differ because commits were cherry-picked between repos rather than merged — subject lines line up 1:1. No update needed. (See §6.2.)
- ~~**Connection-studio schema drift is CRITICAL.**~~ **RETRACTED.** Direct diff shows FSDT's [core/postgres/migration/base/01-tcs-init.sql](../repos/Full-Stack-Docker-Tazama/core/postgres/migration/base/01-tcs-init.sql) already contains `tcs_config`, `tcs_cron_jobs`, `tcs_pull_jobs`, `tcs_push_jobs`, `job_history`, and the `rotate_table_with_data` procedure. It is *ahead* of connection-studio's own `backend/tcs-init.sql` because it adds `payload_json`, `payload_xml`, `related_transaction` columns that `admin-service` actively queries. No change needed. (See §6.3.)
- ~~**DEMS `config` table is not created by FSDT baseline.**~~ **RETRACTED.** event-monitoring-service source queries `tcs_config` ([config-notify.service.ts:34,53](../repos/event-monitoring-service/src/config-notify/config-notify.service.ts) and [dems-engine.service.ts:71](../repos/event-monitoring-service/src/dems-engine/dems-engine.service.ts)), not `config`. The `config` table in event-monitoring-service's own `database/init/01-create-tables.sql` appears to be a legacy artefact. FSDT's `tcs_config` already has every column DEMS reads. No change needed. (See §6.4.)
- **frms-coe-lib and frms-coe-startup-lib version drift.** admin-service, rule-executer, biar/unstructured-pipeline pin `frms-coe-lib@8.2.0-rc.6` and `frms-coe-startup-lib@3.1.0-rc.8`. event-monitoring-service and connection-studio/backend pin the older `8.0.0-rc.5` / `3.0.2-rc.5`. Confirm intentional or align.
- **admin-service simulation studio env is missing** (`SIMULATION_DATABASE_*`, `HOST`) — introduced by admin-service commit `510e9b7` (feat: sim studio). Currently absent from [core/env/admin.env](../repos/Full-Stack-Docker-Tazama/core/env/admin.env).
- **`event-monitoring-service` is only wired as an extension (DEMS)**; no core-stack compose exists.
- **`TAZAMA_VERSION=rc`** floating tag should be pinned to explicit SHAs/immutable RC tags before release to freeze the deployed set.

[Back to top](#top)

---

## Table of Contents

- [1. Repository sync status](#1-repository-sync-status)
- [2. Outstanding PR #239 in FSDT](#2-outstanding-pr-239-in-fsdt)
- [3. CMS-220 — SLA Management gaps](#3-cms-220--sla-management-gaps)
  - [3.1 How FSDT actually deploys CMS (verified)](#31-how-fsdt-actually-deploys-cms-verified)
  - [3.2 What the branches actually contain](#32-what-the-branches-actually-contain-verified-2026-07-20)
  - [3.3 Live-DB state on tazama-b (verified via pg_dump)](#33-live-db-state-on-tazama-b-verified-via-pg_dump)
  - [3.4 What this means for FSDT](#34-what-this-means-for-fsdt)
  - [3.5 Fix](#35-fix)
- [4. CMS-214 — Fraud & AML gaps](#4-cms-214--fraud--aml-gaps)
- [5. Rule-Studio deployment gaps](#5-rule-studio-deployment-gaps)
  - [5.1 Deployment surface currently wired in FSDT](#51-deployment-surface-currently-wired-in-fsdt)
  - [5.2 Rule-21 template — corrected finding](#52-rule-21-template--corrected-finding)
  - [5.3 Backend env & config gaps](#53-backend-env--config-gaps)
  - [5.4 Rule-Executer and rule processor packaging](#54-rule-executer-and-rule-processor-packaging)
  - [5.5 Rule seed data (DDL/DML)](#55-rule-seed-data-ddldml)
- [6. Per-repo concrete breakdown vs FSDT dev](#6-per-repo-concrete-breakdown-vs-fsdt-dev)
  - [6.1 admin-service](#61-admin-service)
  - [6.2 biar](#62-biar)
  - [6.3 connection-studio](#63-connection-studio)
  - [6.4 event-monitoring-service (DEMS)](#64-event-monitoring-service-dems)
  - [6.5 frms-coe-lib](#65-frms-coe-lib)
  - [6.6 frms-coe-startup-lib](#66-frms-coe-startup-lib)
  - [6.7 rule-executer](#67-rule-executer)
  - [6.8 tcs-lib](#68-tcs-lib)
- [7. Consolidated release-blocker checklist](#7-consolidated-release-blocker-checklist)
- [8. Appendix — repo remote & branch state](#8-appendix--repo-remote--branch-state)

[Back to top](#top)

---

## 1. Repository sync status

All local repos under [repos/](../repos/) were fetched and fast-forwarded to `origin/dev` before this review. Two remotes were rewritten from HTTPS to SSH per project convention:

- `Full-Stack-Docker-Tazama` → `git@github.com:tazama-lf/Full-Stack-Docker-Tazama.git`
- `rule-executer` → `git@github.com:tazama-lf/rule-executer.git`

Repos that were on feature branches at report time and were checked out to `dev` for review (working trees auto-stashed):

| Repo | Prior branch | Now on |
|------|--------------|--------|
| biar | paysys/fixEFRuP | dev |
| case-management-system | paysys/dashboard-fixes | dev |
| rule-studio | feat-paysys-fix-simulation | dev |
| rule-studio-devtestops | fix/bootstrap | dev |
| rule-studio-example | feat-paysys-poc | dev |

All other repos were already on `dev`. Full state in [Appendix — repo remote & branch state](#8-appendix--repo-remote--branch-state).

[Back to top](#top)

---

## 2. Outstanding PR #239 in FSDT

- **PR:** [tazama-lf/Full-Stack-Docker-Tazama#239](https://github.com/tazama-lf/Full-Stack-Docker-Tazama/pull/239)
- **Title:** feat: Add new node query in trs-int queries
- **Author:** ReebaPaysys (Reeba Siddiqui)
- **Head → Base:** `feat-paysys-trs-query` → `dev`
- **State:** OPEN — head SHA `8fa7c69e` not on `origin/dev`; no merge commit references #239.
- **Diff:** single file, `core/postgres/migration/base/02-trs-init.sql`, +3/−1.
- **Change:** adds a "Set Variable With Type" node option (typed variable assignment with TypeScript-style cast syntax) to the default seeded content for fresh TRS installs.
- **Dependencies:** none in body.
- **Client-facing implication:** every fresh client bring-up misses the typed-variable node until this PR merges. Existing clients get it only if the seed is re-run.

**Owner action:** send the PR to the client. Trivial and independent of every other item in this report.

[Back to top](#top)

---

## 3. CMS-220 — SLA Management gaps

**Issue:** [case-management-system#220](https://github.com/tazama-lf/case-management-system/issues/220) — *Divorce SLA Management from Prioritisation Beyond Creation*.

Design summary (from [issues/case-management-system/220/solution-220.md](../issues/case-management-system/220/solution-220.md)):
- Split the current 4-value `Priority` enum (`NEW / URGENT / CRITICAL / BREACH`) — which conflates severity and SLA elapsed time — into two orthogonal axes:
  - `priority` (severity): `LOW / MEDIUM / HIGH`, human-set at triage
  - `sla_state` (timing): `ON_TRACK / AT_RISK / DUE_SOON / BREACHED`, derived on read
- Retire the hourly cron in `alert-priority.service.ts` that overwrites `case.priority` from SLA elapsed time.

### 3.1 How FSDT actually deploys CMS (verified)

FSDT contributes zero SQL for `tazama_cms`. The CMS schema is applied by the `migrate` container defined in [extensions/docker-compose.extensions.infrastructure.yaml](../repos/Full-Stack-Docker-Tazama/extensions/docker-compose.extensions.infrastructure.yaml), which pulls image `tazamaorg/case-management-system-migrate:${TAZAMA_VERSION}` and runs `sh /home/app/migrate.sh` — that script (source in [case-management-system/backend/migrate.sh](../repos/case-management-system/backend/migrate.sh)) does `CREATE DATABASE tazama_cms`, then `npx prisma migrate deploy`, then seeds `reference_ids`.

Two mechanisms feed CMS versioning into FSDT, and they behave differently:

| Mode | Backend / Frontend / Voila | Migrate |
|------|-----------------------------|---------|
| **Dev** ([docker-compose.dev.extensions.yaml](../repos/Full-Stack-Docker-Tazama/extensions/docker-compose.dev.extensions.yaml)) | Built from source at `tazama-lf/case-management-system.git#${CMS_BRANCH}` (lines 84, 105, 123). | Uses the prebuilt `tazamaorg/case-management-system-migrate:${TAZAMA_VERSION}` image. `CMS_BRANCH` is NOT consulted here. |
| **Hub** ([docker-compose.hub.extensions.yaml](../repos/Full-Stack-Docker-Tazama/extensions/docker-compose.hub.extensions.yaml)) | Prebuilt `tazamaorg/case-management-system-*:${TAZAMA_VERSION}` images. `CMS_BRANCH` is NOT consulted. | Same prebuilt migrate image as dev mode. |

Implication: `CMS_BRANCH` affects dev-mode CMS *code* but never the *schema*. Hub mode is entirely a function of `TAZAMA_VERSION`.

### 3.2 What the branches actually contain (verified 2026-07-20)

- **`case-management-system/origin/dev`** — 23 Prisma migrations in `backend/prisma/migrations/`. Latest merge: PR #247 (commit `fa31dec6`, 2026-07-16). Earlier SLA-track merges: #244, #240, #234.
- **`case-management-system/origin/paysys/visualization`** — only **3 migrations**. Diverged before the entire SLA track landed.
- **Migrations exclusive to `dev` that `paysys/visualization` is missing** (21 of them, from `git ls-tree` diff):

```
20260416200412_final_outcome
20260601105534_add_tenant_id_to_event_log
20260602080454_remove_dead_code_columns
20260616124311_username
20260618055232_tenant_id_admin
20260618071604_remove_unique_reference_ids
20260619054750_add_reference_ids_txtp_tenant_unique
20260701000001_add_investigation_group
20260702100000_priority_severity_rename
20260702100100_priority_severity_drop_breach
20260702110000_case_sla_due_at
20260702120000_remove_case_status_completed
20260702120000_sla_escalation_records
20260702140000_add_investigation_group_relations
20260705161141_add_sla_escalation_thresholds
20260705170000_add_case_priority_thresholds
20260706120000_sla_policy_target_seconds
20260707120000_case_sla_started_at
20260709000000_add_created_updated_at_to_sla_escalation_records
20260716120000_add_tenant_id_to_sla_escalation_records
```

### 3.3 Live-DB state on tazama-b (verified via pg_dump)

`SELECT migration_name FROM _prisma_migrations` on the live `tazama_cms` (production extensions postgres) returned **24 applied migrations**. All 23 that exist on `dev` are applied. One additional migration (`20260601091334`, applied 2026-06-03, no logs) is present on live but exists on no branch — likely a manual/out-of-band DDL. **Not FSDT's problem to reproduce**; separately flagged to whoever maintains the live DB.

`\dt` on the live DB returned 27 tables (26 CMS tables + `_prisma_migrations`). Every table maps cleanly to a Prisma model on `origin/dev`'s `backend/prisma/schema.prisma` (accounting for `@@map`). Zero drift, zero orphans. Live is fully consistent with `case-management-system/origin/dev`.

### 3.4 What this means for FSDT

- **Dev-mode deployments** currently ship backend/frontend/voila code from `paysys/visualization` (missing all SLA logic) but the `migrate` container applies whichever schema is baked into the `rc`-tagged migrate image. If `rc` was built from a recent CMS `dev`, the schema on the client is ahead of the code — actively broken. Repointing `CMS_BRANCH=dev` aligns them.
- **Hub-mode deployments** are entirely driven by `TAZAMA_VERSION=rc`. If the `rc` tag was rebuilt post-`fa31dec6`, hub-mode clients already have CMS-220. If not, `CMS_BRANCH` won't help — the `rc` image itself must be rebuilt or SHA-pinned.
- [extensions/env/cms.env](../repos/Full-Stack-Docker-Tazama/extensions/env/cms.env) L54–L58 still contains the pre-220 threshold model (`PRIORITY_FIRST_HALF=0.33`, `PRIORITY_SECOND_HALF=0.66`, `PRIORITY_THIRD_HALF=1.0`, `DEFAULT_SLA_HOURS=72`, `ALERT_PRIORITY_CRON_SCHEDULE="0 * * * *"`). Even after the branch fix, these hard-coded thresholds contradict the tenant-configurable SLA policy design and should be revisited.

### 3.5 Fix

1. **Dev mode:** repoint [extensions/.env#L24](../repos/Full-Stack-Docker-Tazama/extensions/.env#L24) `CMS_BRANCH=paysys/visualization` → `CMS_BRANCH=dev`. Covered by the `feat-paysys-fsdt-fixes` branch.
2. **Hub mode:** confirm the `tazamaorg/case-management-system-*:rc` tags were rebuilt from CMS `dev` at or after `fa31dec6`. If not, rebuild or SHA-pin — this is *not* fixable from FSDT alone.
3. Revisit the hard-coded threshold env vars in [extensions/env/cms.env](../repos/Full-Stack-Docker-Tazama/extensions/env/cms.env) once the tenant-level SLA policy table is populated.

[Back to top](#top)

---

## 4. CMS-214 — Fraud & AML gaps

**Issue:** [case-management-system#214](https://github.com/tazama-lf/case-management-system/issues/214). Replace the three-row FRAUD_AND_AML container-case model with a lightweight `InvestigationGroup` link entity. Removes the synthetic `STATUS_84_COMPLETED`, the parent↔child status propagation, and the reporting triple-count.

**State in `case-management-system/dev` — MERGED.** PR #233 (`paysys/fraud_and_aml` → `dev`), merge commit `68da7161`, 2026-07-08. Release version tag `4.0.0-rc.0`. 77 commits, +2306/−2139.

Key artifacts on CMS dev:
- Table + migrations:
  - `backend/prisma/migrations/20260701000001_add_investigation_group`
  - `backend/prisma/migrations/20260702140000_add_investigation_group_relations`
  - `backend/prisma/migrations/20260702120000_remove_case_status_completed`
- Backfill: `backend/prisma/migrations/backfill-investigation-groups.sql` (must run for existing tenants).
- New service module: `backend/src/modules/investigation-group/`.
- Enum: `backend/src/utils/enums/case-enum.ts` removes `STATUS_84_COMPLETED`.
- Frontend rename: `parent_id` → `group_id` in case/alert modals; STATUS_84 closure paths deleted.

**State on live tazama-b — APPLIED.** The `investigation_groups` table exists on the live `tazama_cms` DB (verified via `\dt`), and all three PR #233 migrations are in `_prisma_migrations` (verified via SELECT). Live has been on the CMS-214 schema since at least 2026-07-08.

**State in FSDT dev — same dev/hub split as §3.**

- **Dev mode:** backend/frontend/voila built from `paysys/visualization`, which is missing all three CMS-214 migrations and the `investigation-group` module. Repointing `CMS_BRANCH=dev` picks them up.
- **Hub mode:** entirely dependent on whether `tazamaorg/case-management-system-*:rc` tags were rebuilt post-`68da7161`. Not fixable from FSDT alone if they weren't.
- [biar/nifi/tazama.xml](../repos/Full-Stack-Docker-Tazama/biar/nifi/tazama.xml) already references `investigation_groups`, but was authored before PR #233's final schema — needs a diff pass against the merged model for any residual `parent_id` references.

**Fix:**
1. Same `CMS_BRANCH=dev` repoint as CMS-220 (both issues share the fix). Covered by `feat-paysys-fsdt-fixes`.
2. Hub-side image rebuild — see §3.5 item 2.
3. Backfill: `backend/prisma/migrations/backfill-investigation-groups.sql` is part of CMS `dev`'s migration set and runs as part of `prisma migrate deploy`, so the migrate container handles it automatically on any fresh bring-up. For live tenants already migrated pre-`68da7161`, run it manually (already applied on tazama-b — no action).
4. Diff [biar/nifi/tazama.xml](../repos/Full-Stack-Docker-Tazama/biar/nifi/tazama.xml) against the CMS-214 finalized schema for any residual `parent_id` references, replace with `group_id`.
5. Audit [extensions/env/cms.env](../repos/Full-Stack-Docker-Tazama/extensions/env/cms.env) for any `PARENT_CASE_*` or `STATUS_84_*` flags — none should remain.

[Back to top](#top)

---

## 5. Rule-Studio deployment gaps

Rule-Studio (`trs-backend`, `trs-frontend`) is wired into FSDT; RSDTO is intentionally out of scope. Findings below focus on runtime dependencies and templates the frontend/backend actually need at bring-up.

### 5.1 Deployment surface currently wired in FSDT

Files referencing rule-studio on FSDT dev (`3f88eea`):

- [extensions/docker-compose.dev.extensions.yaml](../repos/Full-Stack-Docker-Tazama/extensions/docker-compose.dev.extensions.yaml#L41-L79) — `trs-backend` (3005) and `trs-frontend` (5174), both built from GitHub source `tazama-lf/rule-studio#${TRS_BRANCH_BACKEND|FRONTEND}` (default `dev`), depending on `opensearch-node1`.
- [extensions/docker-compose.hub.extensions.yaml](../repos/Full-Stack-Docker-Tazama/extensions/docker-compose.hub.extensions.yaml#L37-L68) — production-image variants.
- [extensions/.env#L19-L21](../repos/Full-Stack-Docker-Tazama/extensions/.env#L19-L21) — branch overrides.
- [extensions/env/trs.env](../repos/Full-Stack-Docker-Tazama/extensions/env/trs.env) — env for both.

Frontend env vars set in FSDT reference:
- `VITE_SANDBOX_API_URL=http://${SERVER_A_HOST}:3050` — pointer to devtestops (RSDTO, not deployed; out of scope per owner)
- `VITE_NATS_API_URL=…:4000`, `VITE_SIMULATION_ENDPOINT=…:5000`, `VITE_DEMS_ENDPOINT`, `VITE_ADMIN_ENDPOINT`

[Back to top](#top)

### 5.2 Rule-21 template — corrected finding

**Rule-21 IS a real, seeded bootstrap rule that FSDT already ships. There is nothing missing on the FSDT side for rule-21 itself.** Earlier draft claimed rule-21 was created on demand — that was wrong.

Evidence:

- `rule-studio/backend/src/constants/constant.ts` defines `export const BASE_RULE_ID = '21';`.
- `rule-studio/backend/src/services/rules/rules.service.ts` L125-134: on rule creation, the service calls `getRuleFlow(BASE_RULE_ID, …)` and then `adminServiceClient.createRuleFlow(rule.id, { flow_json_rule_builder: baseRuleFlow.result.flow_json_rule_builder ?? {}, flow_json_test_case: baseRuleFlow.result.flow_json_test_case ?? {}, …})`. Rule-21 supplies the default `flow_json_rule_builder` and `flow_json_test_case` for every new rule.
- FSDT seeds it: [core/postgres/migration/base/02-trs-init.sql](../repos/Full-Stack-Docker-Tazama/core/postgres/migration/base/02-trs-init.sql) line ~242 inserts `id=21, rule_name='DEFAULT-rule-21', rule_type='AML', rule_config_id='021@1.0.0'`, and line ~246 seeds the corresponding `trs_rule_flow` row for `rule_id=21`.

**Implication:** the rule-creation flow will work on a fresh FSDT bring-up as long as the base ordering (00-CREATE → 01-tcs-init → 02-trs-init) runs cleanly. Note that PR #239 also touches 02-trs-init.sql and improves the seeded node set — merging it strictly improves rule-21's default builder content.

**Nothing to fix here** — this section exists to correct the record and confirm the seed pipeline.

[Back to top](#top)

### 5.3 Backend env & config gaps

Findings below are derived from a direct grep of `rule-studio/backend/src/` for every `process.env.*` reference plus explicit reads of the env validator, notification service, and DockerHub service. Cross-referenced against the running-machine `trs.env` shared by the project owner (which has vars FSDT currently ships without).

#### Vars the backend reads (source of truth)

From `grep -rn "process.env" rule-studio/backend/src/`:

```
ADMIN_SERVICE_URL, ALLOWED_ORIGINS, API_HOST, BULL_PREFIX, CRYPTO_SECRET_KEY,
DEMS_BASE_URL, DLH_URL, DOCKERHUB_NAMESPACE, ENCRYPTION_KEY, IV_LENGTH,
NODE_ENV, PORT, REDIS_HOST, REDIS_PASSWORD, REDIS_PORT, TAZAMA_AUTH_URL,
TAZAMA_REPO_BRANCH, TESTCONTAINERS_HOST_OVERRIDE, WRITE_SWAGGER_JSON
```

Plus (via `ConfigService.get(...)`, not `process.env` directly):

```
SMTP_HOST, SMTP_PORT, SMTP_USER, SMTP_PASS, SMTP_SECURE, SMTP_FROM_EMAIL, SMTP_FROM_NAME,
DOCKERHUB_TOKEN, DOCKERHUB_USERNAME
```

Plus `class-validator` schema in [backend/src/services/config/env.validation.ts](../repos/rule-studio/backend/src/services/config/env.validation.ts) — these are **validated at boot** and the process refuses to start if missing:

```
NODE_ENV (enum), MAX_CPU (numeric string), FUNCTION_NAME, TAZAMA_AUTH_URL,
AUTH_PUBLIC_KEY_PATH, CERT_PATH_PUBLIC
```

#### Required-at-boot vs optional split

| Var | Where used | Required? |
|-----|------------|-----------|
| `NODE_ENV` | env validator | **Yes** — boot fails if not one of `development`/`production`/`test`/`dev`/`prod` |
| `MAX_CPU` | env validator | **Yes** — must be numeric string |
| `FUNCTION_NAME` | env validator | **Yes** |
| `TAZAMA_AUTH_URL` | env validator | **Yes** |
| `AUTH_PUBLIC_KEY_PATH` | env validator | **Yes** |
| `CERT_PATH_PUBLIC` | env validator | **Yes** |
| `PORT` | server bootstrap | **Yes** in practice |
| `REDIS_HOST` / `REDIS_PORT` / `REDIS_PASSWORD` | queue + socket.io-redis | **Yes** — connect on startup |
| `ADMIN_SERVICE_URL` | rules service (rule-21 flow fetch) | **Yes** — first user action fails otherwise |
| `ENCRYPTION_KEY` / `IV_LENGTH` / `CRYPTO_SECRET_KEY` | crypto module | **Yes** — used to encrypt tokens/secrets at rest |
| `ALLOWED_ORIGINS` | CORS middleware | **Yes** for browser clients |
| `SMTP_HOST` / `SMTP_PORT` / `SMTP_USER` / `SMTP_PASS` / `SMTP_SECURE` / `SMTP_FROM_EMAIL` / `SMTP_FROM_NAME` | notification service ([notification.service.ts:37-89](../repos/rule-studio/backend/src/services/notification/notification.service.ts)) | Optional at boot (logs a warning: *"Set SMTP_HOST, SMTP_USER, SMTP_PASS in .env to enable emails"*), **but throws on first send** if `SMTP_FROM_EMAIL` missing. Required for release feature-set. |
| `DOCKERHUB_TOKEN` / `DOCKERHUB_USERNAME` / `DOCKERHUB_NAMESPACE` | simulation studio ([dockerhub.service.ts:20-25](../repos/rule-studio/backend/src/services/simulation-studio/dockerhub/dockerhub.service.ts)) | Optional at boot, **throws first time simulation studio calls DockerHub**. Required if simulation studio is in scope. |
| `API_HOST`, `BULL_PREFIX`, `DEMS_BASE_URL`, `DLH_URL`, `TAZAMA_REPO_BRANCH`, `WRITE_SWAGGER_JSON` | various | Optional in most codepaths — verify per-feature use. |
| `AUDIT_PROVIDER`, `OPENSEARCH_*`, `OPENSEARCH_INDEX` | audit logging | Effectively required for auditing feature. |
| `TESTCONTAINERS_HOST_OVERRIDE` | tests | Test-only. |

#### What FSDT ships today vs the gap

Current [extensions/env/trs.env](../repos/Full-Stack-Docker-Tazama/extensions/env/trs.env) (26 lines) provides:

```
NODE_ENV, MAX_CPU, FUNCTION_NAME, PORT, TAZAMA_AUTH_URL, AUTH_PUBLIC_KEY_PATH,
CERT_PATH_PUBLIC, ADMIN_SERVICE_URL, ALLOWED_ORIGINS, CRYPTO_SECRET_KEY,
ENCRYPTION_KEY, IV_LENGTH, AUDIT_PROVIDER, OPENSEARCH_NODE, OPENSEARCH_USERNAME,
OPENSEARCH_PASSWORD, OPENSEARCH_INDEX, VITE_CRYPTO_KEY, VITE_APP_NAME, VITE_APP_VERSION
```

**Missing (append to `trs.env`):**

```env
# Redis / valkey — required at boot
REDIS_HOST=valkey
REDIS_PORT=6379
REDIS_PASSWORD=<valkey password from core/.env>

# SMTP — required for release notifications; NestJS notification.service.ts
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_SECURE=false
SMTP_USER=<client SMTP user>
SMTP_PASS=<client SMTP app password>
SMTP_FROM_EMAIL=<client from address>
SMTP_FROM_NAME="Tazama Rule Studio"

# DockerHub — required for simulation studio image pull/push
DOCKERHUB_TOKEN=<PAT>
DOCKERHUB_USERNAME=<user>
DOCKERHUB_NAMESPACE=<namespace>

# Optional but likely needed
API_HOST=trs-backend
BULL_PREFIX=trs
DEMS_BASE_URL=http://${SERVER_B_HOST}:3106
DLH_URL=http://${SERVER_B_HOST}:<port>
TAZAMA_REPO_BRANCH=dev
OPENSEARCH_SSL_REJECT_UNAUTHORIZED=false
```

#### Additional observations from the shared env

- `IV_LENGTH=abcdefghijklmnop` in both the shared file **and** FSDT's committed trs.env — the variable name suggests a length (integer), but the value is a 16-character string. This is likely being used as an AES-CBC IV; if the code expects an integer length, this is misconfigured. Confirm against the crypto module before shipping.
- `CRYPTO_SECRET_KEY`, `ENCRYPTION_KEY`, `VITE_CRYPTO_KEY` are all identical placeholders (`rCRrt08XYmYZrUoKffOQWXa327AJyBxv`) checked into FSDT. **These MUST be rotated per-client before release** and must not remain in the public dev branch.
- **`DOCKERHUB_TOKEN` and `SMTP_PASS` are secrets** — do not commit real values to FSDT. Provide via a `.env.local` overlay (or the tofu / Secrets Manager path used by [infra/aws](../repos/Full-Stack-Docker-Tazama/infra/) elsewhere) and only commit placeholders/sample values.

#### Redis service wiring

Core deploys `valkey:7.2.5` (Redis-compatible). `trs-backend` is not currently pointed at it and does not `depends_on: [valkey]`. Add the dependency in both [extensions/docker-compose.dev.extensions.yaml](../repos/Full-Stack-Docker-Tazama/extensions/docker-compose.dev.extensions.yaml) and [extensions/docker-compose.hub.extensions.yaml](../repos/Full-Stack-Docker-Tazama/extensions/docker-compose.hub.extensions.yaml) alongside `opensearch-node1`.

**Note on the sample env shared by the owner:** `REDIS_HOST=10.10.80.37`, `REDIS_PORT=6379`, `REDIS_PASSWORD=redis-password`, and `ADMIN_SERVICE_URL=http://localhost:3100` are hardcoded to a specific host layout. In FSDT these must remain parameterized (`${SERVER_A_HOST}` etc.) so the compose stack is portable.

[Back to top](#top)

### 5.4 Rule-Executer and rule processor packaging

`rule-executer/dev` (`5025364`) is a minimal wrapper — Dockerfile installs `@<org>/rule-<id>@latest` from GitHub Packages at build time. There are no per-rule folders. FSDT-side implication:

- Each rule that runs in production needs (a) a published npm package on GitHub Packages, (b) a config-DB row telling orchestration which rule-id/version to invoke.
- FSDT [core/postgres/migration/config/05-studio-rules.sql](../repos/Full-Stack-Docker-Tazama/core/postgres/migration/config/05-studio-rules.sql) seeds rules `901` and `902` only for production flow.
- rule-executer commit `c999cc2` (fix CI token for rule 901/902 dispatch) — confirm FSDT release automation uses the fixed workflow.

[Back to top](#top)

### 5.5 Rule seed data (DDL/DML)

- Core TRS schema: [core/postgres/migration/base/02-trs-init.sql](../repos/Full-Stack-Docker-Tazama/core/postgres/migration/base/02-trs-init.sql) — needs PR #239's typed-node update.
- Config: [core/postgres/migration/config/05-studio-rules.sql](../repos/Full-Stack-Docker-Tazama/core/postgres/migration/config/05-studio-rules.sql) — rules 901/902 only. If the client expects any additional pre-baked rules, they must be added here (with matching npm packages published).

[Back to top](#top)

---

## 6. Per-repo concrete breakdown vs FSDT dev

Each subsection follows the same layout: repo state → what FSDT wires today → env-var diff → schema/migration diff → image/build reference → concrete blockers.

Repos already covered elsewhere are excluded:
- `case-management-system` — see §3 and §4.
- `rule-studio` — see §5.
- `rule-studio-devtestops`, `rule-studio-example` — out of scope per owner.

### 6.1 admin-service

**a) Repo state.**
- `origin/dev` HEAD: `e71cc25` (2026-07-03) — Merge PR #468 (fix/467-rule-parameters-strip).
- Recent commits include `510e9b7` (feat: sim studio) and `2c8abb9` (feat: service-channel endpoint).

**b) What FSDT wires today.**
- [core/docker-compose.dev.core.yaml#L9-L27](../repos/Full-Stack-Docker-Tazama/core/docker-compose.dev.core.yaml#L9-L27) — service `admin-service`, builds from `github.com/tazama-lf/admin-service.git#${ADMIN_BRANCH}`.
- [core/docker-compose.hub.core.yaml](../repos/Full-Stack-Docker-Tazama/core/docker-compose.hub.core.yaml) — uses image `tazamaorg/admin-service:${TAZAMA_VERSION}`.
- [core/env/admin.env](../repos/Full-Stack-Docker-Tazama/core/env/admin.env) — env file (~69 lines).
- [core/.env#L8](../repos/Full-Stack-Docker-Tazama/core/.env#L8) — `ADMIN_BRANCH=dev`.

**c) Env var diff.** Missing in [core/env/admin.env](../repos/Full-Stack-Docker-Tazama/core/env/admin.env), present in `admin-service/.env.template`:
- `HOST=0.0.0.0` (template L8)
- `SIMULATION_DATABASE_PORT` (template L26)
- `SIMULATION_DATABASE` (template L27)
- `SIMULATION_DATABASE_USER` (template L28)
- `SIMULATION_DATABASE_PASSWORD` (template L29)
- `SIMULATION_DATABASE_HOST` (template L30)
- `DISTRIBUTED_CACHE_ENABLED` — FSDT sets `false` (admin.env L25); template default `true`. Verify intent.

Extras in FSDT not in template (fine, service tolerates): `APM_SERVICE_NAME`, `APM_ACTIVE`, `LOG_LEVEL`, `STARTUP_TYPE=nats`.

**d) Schema / migration.** admin-service has no repo-level Prisma/SQL migrations. Consumes core Tazama tables (`network_map`, `typology`, `rule`, `evaluation`, `event_history`, `raw_history`) created by FSDT [core/postgres/migration/base/00-CREATE.sql](../repos/Full-Stack-Docker-Tazama/core/postgres/migration/base/00-CREATE.sql). No delta.

**e) Image / build reference.** Dev builds from `${ADMIN_BRANCH}=dev`. Hub uses `tazamaorg/admin-service:rc` (floating). Service HEAD is `e71cc25`.

**f) Concrete blockers.**
- Add to [core/env/admin.env](../repos/Full-Stack-Docker-Tazama/core/env/admin.env):
  ```
  HOST=0.0.0.0
  SIMULATION_DATABASE_HOST=postgres
  SIMULATION_DATABASE_PORT=5432
  SIMULATION_DATABASE=simulation
  SIMULATION_DATABASE_USER=postgres
  SIMULATION_DATABASE_PASSWORD=<secret>
  ```
- Decide whether `DISTRIBUTED_CACHE_ENABLED` should be `true` (template default). If yes, wire `valkey` connectivity in admin.env as well.
- Pin `TAZAMA_VERSION` off the floating `rc` tag.

[Back to top](#top)

### 6.2 biar

**a) Repo state.**
- `origin/dev` HEAD: `fa8d362` (2026-07-19) — Merge PR #134 (fix: calibration null).
- Recent: `cdb4437` (Lakehouse Catalog join alignment), `6a674a3` (Add More CMS Database Table for NiFi), `3371908` (nifi template updates).

**b) What FSDT wires today.**
- [biar/docker-compose.dev.biar.yaml#L5-L95](../repos/Full-Stack-Docker-Tazama/biar/docker-compose.dev.biar.yaml#L5-L95) — five services: `nifi`, `automation-orchestrator`, `datalakehouse-api`, `unstructured-pipeline`, `jupyterhub`. Each builds from `github.com/tazama-lf/biar.git#${BIAR_BRANCH}:<subdir>`.
- [biar/docker-compose.hub.biar.yaml](../repos/Full-Stack-Docker-Tazama/biar/docker-compose.hub.biar.yaml) — pre-built `tazamaorg/biar-nifi:rc`, etc.
- [biar/.env#L1](../repos/Full-Stack-Docker-Tazama/biar/.env#L1) — `BIAR_BRANCH=dev`.
- Env files: [biar/env/nifi.env](../repos/Full-Stack-Docker-Tazama/biar/env/nifi.env), [biar/env/automation-orchestrator.env](../repos/Full-Stack-Docker-Tazama/biar/env/automation-orchestrator.env), [biar/env/datalakehouse-api.env](../repos/Full-Stack-Docker-Tazama/biar/env/datalakehouse-api.env), [biar/env/unstructured-pipeline.env](../repos/Full-Stack-Docker-Tazama/biar/env/unstructured-pipeline.env), [biar/env/jupyterlab.env](../repos/Full-Stack-Docker-Tazama/biar/env/jupyterlab.env).
- NiFi flow template: [biar/nifi/tazama.xml](../repos/Full-Stack-Docker-Tazama/biar/nifi/tazama.xml) — last touched at FSDT commit `7817cda`.

**c) Env var diff.** biar has no root `.env.template`. Each subdir has undocumented config surface. Cannot mechanically diff; recommend a manual pass over each Dockerfile / compose in the biar repo to extract required vars and diff against FSDT env files.

**d) Schema / migration.** No Prisma/SQL migrations in biar. NiFi/Solr/S3 stores are self-managed.

**e) Image / build reference.** Dev builds from `${BIAR_BRANCH}=dev`. Hub uses `tazamaorg/biar-*:rc`. Service HEAD is `fa8d362`.

**f) Concrete blockers.**
- ~~**NiFi template refresh.**~~ **RETRACTED.** Direct byte-diff after normalising line endings (FSDT copy is CRLF, biar canonical is LF) shows the two files are 100% content-identical at 52899 lines. Commit SHAs differ (`7817cda` in FSDT vs `356343c` in biar) but the subject lines correspond 1:1 (`fix(nifi): Fix Some Issue in the UpdateAttributes`, `feat(nifi): Add More CMS Database Table`, `feat(nifi): update NiFi template for cms_usernames`), so the commits were cherry-picked between repos and content was kept in sync. **No update needed.**
- **frms-coe-lib version.** biar/unstructured-pipeline pins `@tazama-lf/frms-coe-lib@8.2.0-rc.6` — ensure the build context resolves this.
- **Env surface audit** — extract required env from each biar subdir; today the FSDT env files are minimal.

[Back to top](#top)

### 6.3 connection-studio

**a) Repo state.**
- `origin/dev` HEAD: `d97d7c0` (2026-07-07) — Merge PR #98 (fix ENCRYPTION_KEY validation).
- Recent: `3e0b656` (ci: sync workflows), `b2c20a9` / `ebc38dd` (ENCRYPTION_KEY 32-byte enforcement).

**b) What FSDT wires today.**
- [extensions/docker-compose.dev.extensions.yaml#L2-L39](../repos/Full-Stack-Docker-Tazama/extensions/docker-compose.dev.extensions.yaml#L2-L39) — `tcs-backend` and `tcs-frontend`, built from `github.com/tazama-lf/connection-studio.git#${TCS_BRANCH}:backend|frontend`.
- [extensions/docker-compose.hub.extensions.yaml](../repos/Full-Stack-Docker-Tazama/extensions/docker-compose.hub.extensions.yaml) — pre-built `tazamaorg/connection-studio-backend:rc`, `tazamaorg/connection-studio-frontend:rc`.
- [extensions/env/tcs.env](../repos/Full-Stack-Docker-Tazama/extensions/env/tcs.env) — env file (~66 lines).
- [extensions/.env#L17](../repos/Full-Stack-Docker-Tazama/extensions/.env#L17) — `TCS_BRANCH=dev`.
- Init SQL: [core/postgres/migration/base/01-tcs-init.sql](../repos/Full-Stack-Docker-Tazama/core/postgres/migration/base/01-tcs-init.sql) — ~100 lines.

**c) Env var diff.** Missing in [extensions/env/tcs.env](../repos/Full-Stack-Docker-Tazama/extensions/env/tcs.env), present in `connection-studio/backend/.env.example`:
- `SESSION_TIMEOUT_MINUTES=30` (backend .env.example L14)
- `DEMS_URL=http://…:3106` (.env.example L55)
- `DEAPI_URL=http://…:3001` (.env.example L56)

Extras in FSDT not in .env.example (verify still consumed by backend): `SERVER_URL=nats://…:14222`, `STARTUP_TYPE=nats`, `PRODUCER_STREAM=config.notification`, `CONSUMER_STREAM=config.notification`, `STREAM_SUBJECT=config.notification`.

Secret hygiene: `ENCRYPTION_KEY` in FSDT is a checked-in 32-byte placeholder (`8fj29dkd82hs91kd93kd82hs91kd83ks`). Verify byte length (recent commits enforce it) and rotate before release.

**d) Schema / migration.** **No drift.** Direct diff shows FSDT's [core/postgres/migration/base/01-tcs-init.sql](../repos/Full-Stack-Docker-Tazama/core/postgres/migration/base/01-tcs-init.sql) already contains `tcs_config`, `tcs_cron_jobs`, `tcs_pull_jobs`, `tcs_push_jobs`, `job_history`, and the `rotate_table_with_data` procedure. It is *ahead* of connection-studio's own `backend/tcs-init.sql`:

- FSDT names the config table `tcs_config` (not `config`) and adds `payload_json`, `payload_xml`, `related_transaction` columns — all of which `admin-service/src/repositories/configuration/tcs.config.repository.ts` actively INSERTs and SELECTs (lines 53, 60, 118, 200, 225, 252, 288).
- FSDT also adds `tazama_data_model_json` + a seed INSERT that connection-studio's file lacks.
- Missing from FSDT compared to connection-studio: `destination`, `destination_type`, `destination_type_fields` tables and their INSERTs. **However, no service references these** — grepping `connection-studio/backend/src/` and `admin-service/src/` for these table names returns zero hits. They appear to be legacy.

**No change required.**

**e) Image / build reference.** Dev builds from `${TCS_BRANCH}=dev`. Hub uses `:rc`. Service HEAD `d97d7c0`.

**f) Concrete blockers.**
- **Add** to [extensions/env/tcs.env](../repos/Full-Stack-Docker-Tazama/extensions/env/tcs.env):
  ```
  SESSION_TIMEOUT_MINUTES=30
  DEMS_URL=http://${SERVER_B_HOST}:3106
  DEAPI_URL=http://${SERVER_B_HOST}:3001
  ```
- Rotate `ENCRYPTION_KEY` from placeholder to a real 32-byte secret before release.

[Back to top](#top)

### 6.4 event-monitoring-service (DEMS)

**a) Repo state.**
- `origin/dev` HEAD: `cd78442` (2026-07-07) — Merge PR #52 (feat: tests coverage).
- Recent: `bc318fb` (lint scope), `ccedf09` (rename event-adjudicator → event-monitoring-service).

**b) What FSDT wires today.**
- [extensions/docker-compose.dev.extensions.apis.yaml#L2-L16](../repos/Full-Stack-Docker-Tazama/extensions/docker-compose.dev.extensions.apis.yaml#L2-L16) — service `dems`, builds from `github.com/tazama-lf/event-monitoring-service.git#${DEMS_BRANCH}`.
- [extensions/docker-compose.hub.extensions.apis.yaml](../repos/Full-Stack-Docker-Tazama/extensions/docker-compose.hub.extensions.apis.yaml) — pre-built `tazamaorg/event-monitoring-service:rc`.
- [extensions/env/dems.env](../repos/Full-Stack-Docker-Tazama/extensions/env/dems.env) — env file (~52 lines).
- [extensions/.env#L27](../repos/Full-Stack-Docker-Tazama/extensions/.env#L27) — `DEMS_BRANCH=dev`.
- **No core-stack compose** — DEMS is extensions-only today.

**c) Env var diff.** Minor differences vs service `.env.sample`:
- `AUTH_PUBLIC_KEY_PATH` FSDT sets `/auth/test-public-key.pem`; sample expects `public-key.pem`. Reconcile.
- `CORS_ORIGINS` parameterization OK.

FSDT extras (verify still consumed): `CONFIGURATION_DATABASE_URL` (single URL vs sample's split DB_HOST/PORT/USER/PASSWORD), `SIDECAR_HOST`.

**d) Schema / migration.** **No drift.** Retracting the earlier "missing `config` table" claim. DEMS source queries `tcs_config`, not `config`:

- [event-monitoring-service/src/config-notify/config-notify.service.ts:34,53](../repos/event-monitoring-service/src/config-notify/config-notify.service.ts) — `SELECT … FROM tcs_config`.
- [event-monitoring-service/src/dems-engine/dems-engine.service.ts:71](../repos/event-monitoring-service/src/dems-engine/dems-engine.service.ts) — `SELECT … FROM tcs_config WHERE …`.

FSDT's `tcs_config` (defined in [core/postgres/migration/base/01-tcs-init.sql](../repos/Full-Stack-Docker-Tazama/core/postgres/migration/base/01-tcs-init.sql)) has every column DEMS reads (`endpoint_path`, `schema`, `mapping`, `functions`, `related_transaction`, `publishing_status`, `tenant_id`). The `config` table in `event-monitoring-service/database/init/01-create-tables.sql` appears to be a legacy artefact — no runtime code references it. FSDT correctly ships `dems_quarantine` in [core/postgres/migration/base/00-CREATE.sql](../repos/Full-Stack-Docker-Tazama/core/postgres/migration/base/00-CREATE.sql).

**No change required.**

**e) Image / build reference.** Dev builds from `${DEMS_BRANCH}=dev`. Hub uses `:rc`. Service HEAD `cd78442`.

**f) Concrete blockers.**
- **Reconcile** `AUTH_PUBLIC_KEY_PATH` between FSDT and DEMS runtime expectation.
- **Add a core-stack compose** (`core/docker-compose.dev.event-monitoring.yaml` + `core/env/dems.env`) if standalone Server A deploys need DEMS without extensions.
- Grep FSDT for residual `event-adjudicator` references (clean at report time).

[Back to top](#top)

### 6.5 frms-coe-lib

**a) Repo state.**
- `origin/dev` HEAD: `31a2bd6` (2026-07-01) — Merge PR #433 (security sweep).
- Recent: `5513ac0` (feat: make SIMULATION_DATABASE optional), `513d91d` (CloudEvents service-channel Part A).
- Library only (npm package `@tazama-lf/frms-coe-lib`); no docker image.

**b) FSDT wiring.** N/A — consumed as npm dependency at build time by service images.

**c) Env var diff.** N/A — no env surface.

**d) Schema / migration.** N/A.

**e) Version pinning across FSDT-deployed services.**

| Consumer | Pinned version |
|----------|-----------------|
| admin-service | `8.2.0-rc.6` |
| rule-executer | `8.2.0-rc.6` |
| biar/unstructured-pipeline | `8.2.0-rc.6` |
| event-monitoring-service | `8.0.0-rc.5` |
| connection-studio/backend | `8.0.0-rc.5` |

**f) Concrete blockers.**
- **Version drift.** Two consumers lag by a minor version. `5513ac0` (SIMULATION_DATABASE optional) and `513d91d` (CloudEvents service-channel) are in 8.2.x but not 8.0.x. Either bump event-monitoring-service and connection-studio/backend to `8.2.0-rc.6` (preferred, unless breaking) or explicitly document why they lag.

[Back to top](#top)

### 6.6 frms-coe-startup-lib

**a) Repo state.**
- `origin/dev` HEAD: `9510073` (2026-07-01) — Merge PR #287 (security sweep).
- Recent: `188b001` (build deps bump), `cfe7810` (NATS startup hardening), `f86f5d2` (neuter STARTUP_TYPE toggle for security), `acd2f87` (remove GoogleBucketsRelay and BigQueryRelay).

**b) FSDT wiring.** N/A — npm dependency.

**c) Env var diff.** N/A directly, but `STARTUP_TYPE` semantics changed: the toggle is neutered upstream, and legacy `googlebuckets` / `bigquery` transports are removed. FSDT env files that still set `STARTUP_TYPE=nats` are fine; anything else must be purged.

**d) Schema / migration.** N/A.

**e) Version pinning across FSDT-deployed services.**

| Consumer | Pinned version |
|----------|-----------------|
| admin-service | `3.1.0-rc.8` |
| rule-executer | `3.1.0-rc.8` |
| event-monitoring-service | `3.0.2-rc.5` |
| connection-studio/backend | `3.0.2-rc.5` |

**f) Concrete blockers.**
- **Version drift** parallels §6.5. Same recommendation: bump the two laggards or explicitly document.
- **Grep FSDT env files for `STARTUP_TYPE` values** — anything other than `nats` should be removed. At report time only `nats` values are set.
- Confirm no compose file references `googlebuckets` or `bigquery` relays.

[Back to top](#top)

### 6.7 rule-executer

**a) Repo state.**
- `origin/dev` HEAD: `5025364` (2026-07-03) — Merge PR #446 (dispatch token fix).
- Recent: `962cb6f` (ci: sync workflows), `c999cc2` (fix CI token for rule 901/902 dispatch).

**b) What FSDT wires today.**
- [core/docker-compose.dev.core.yaml#L74-L92](../repos/Full-Stack-Docker-Tazama/core/docker-compose.dev.core.yaml#L74-L92) — service `rule-901`, builds from `github.com/tazama-lf/rule-executer.git#${RULE_901_BRANCH}`.
- [core/docker-compose.full.rules.yaml](../repos/Full-Stack-Docker-Tazama/core/docker-compose.full.rules.yaml) — `rule-901` and `rule-902` instances; each with env files [core/env/rule-901.env](../repos/Full-Stack-Docker-Tazama/core/env/rule-901.env) and [core/env/rule-902.env](../repos/Full-Stack-Docker-Tazama/core/env/rule-902.env).
- [core/.env#L11-L12](../repos/Full-Stack-Docker-Tazama/core/.env#L11-L12) — `RULE_901_BRANCH=dev`, `RULE_902_BRANCH=dev`.
- Hub uses `tazamaorg/rule-executer:${TAZAMA_VERSION}` per rule id.

**c) Env var diff.**
- Template `.env.template` sets `FUNCTION_NAME='rule-901'`; FSDT sets `FUNCTION_NAME=rule-901-rel-1-0-0` — FSDT more specific, OK.
- Template `RULE_NAME='DEFAULT-901'`; FSDT `RULE_NAME="901"` — reconcile.
- FSDT extras (correct to have): `SIDECAR_HOST=event-sidecar:${EVENT_SIDECAR_PORT}`, `SUPPRESS_ALERTS`, `LOG_LEVEL`.

**d) Schema / migration.** No repo-level migrations. Depends on FSDT's core Tazama tables.

**e) Image / build reference.** Dev builds from per-rule `${RULE_9xx_BRANCH}=dev`. Hub uses `:rc`. Service HEAD `5025364`. rule-executer pins `frms-coe-lib@8.2.0-rc.6` and `frms-coe-startup-lib@3.1.0-rc.8` (aligned with admin-service, ahead of DEMS/TCS).

**f) Concrete blockers.**
- Confirm `RULE_NAME` value in [core/env/rule-901.env](../repos/Full-Stack-Docker-Tazama/core/env/rule-901.env) matches what the wrapper expects (`901` vs `DEFAULT-901`).
- Verify FSDT release automation runs the CI workflow that includes fix `c999cc2` so rule-9xx image dispatches don't 403 on the write token.

[Back to top](#top)

### 6.8 tcs-lib

**a) Repo state.**
- `origin/dev` HEAD: `d578132` (2026-07-02) — Merge PR #53 (related transaction field).
- Library only.

**b) FSDT wiring.** N/A — npm dependency.

**c) Env var diff.** N/A.

**d) Schema / migration.** N/A directly, but the `related_transaction` field added by `840c3b7` may show up in downstream tables (verify via CMS/TCS schema).

**e) Consumers.**

| Consumer | Pinned version |
|----------|-----------------|
| connection-studio/backend | `1.0.156` |

**f) Concrete blockers.** None identified. Single consumer, pinned. Confirm `1.0.156` includes the related-transaction change; if not, decide whether to bump.

[Back to top](#top)

---

## 7. Consolidated release-blocker checklist

Ordered by client impact. **BLOCKER** = must be resolved before release.

1. **BLOCKER — CMS branch pin (dev mode).** [extensions/.env#L24](../repos/Full-Stack-Docker-Tazama/extensions/.env#L24): change `CMS_BRANCH=paysys/visualization` → `dev`. Fixes dev-mode CMS code drift. Does NOT fix hub-mode; see #2. Tracked in the `feat-paysys-fsdt-fixes` branch. (§3, §4)
2. **BLOCKER — CMS docker images (hub mode).** Confirm `tazamaorg/case-management-system-backend:rc`, `…-frontend:rc`, `…-migrate:rc`, `…-voila:rc` were rebuilt from CMS `dev` at or after `fa31dec6`. If not, rebuild or SHA-pin — this is *not* fixable from FSDT alone. `CMS_BRANCH` is not consulted in hub mode. (§3, §4)
3. **BLOCKER — Rule-Studio env completeness.** Extend [extensions/env/trs.env](../repos/Full-Stack-Docker-Tazama/extensions/env/trs.env) with:
   - Redis block (`REDIS_HOST=valkey`, `REDIS_PORT=6379`, `REDIS_PASSWORD=…`) — required at boot.
   - SMTP block (`SMTP_HOST`, `SMTP_PORT`, `SMTP_SECURE`, `SMTP_USER`, `SMTP_PASS`, `SMTP_FROM_EMAIL`, `SMTP_FROM_NAME`) — required for notification feature.
   - DockerHub block (`DOCKERHUB_TOKEN`, `DOCKERHUB_USERNAME`, `DOCKERHUB_NAMESPACE`) — required for simulation studio.
   - Add `depends_on: [valkey, opensearch-node1]` to `trs-backend` in both dev and hub extensions compose files.
   - Consider adding `API_HOST`, `BULL_PREFIX`, `DEMS_BASE_URL`, `DLH_URL`, `TAZAMA_REPO_BRANCH` per feature scope. (§5.3)
4. **Secrets hygiene for trs.env.** Placeholder values are baked into FSDT per convention (see `cms.env` SMTP block), but the consumer must edit `DOCKERHUB_USERNAME` / `DOCKERHUB_TOKEN` (currently `<your-...>` markers) before deploying. `CRYPTO_SECRET_KEY`, `ENCRYPTION_KEY`, and `VITE_CRYPTO_KEY` are also placeholder values — rotate per-client. `IV_LENGTH` was fixed to `12` in `feat-paysys-fsdt-fixes` (was `abcdefghijklmnop` → `parseInt` NaN). (§5.3)
5. **Merge PR #239** (`feat-paysys-trs-query`) — owner will send. (§2)
6. **admin-service `HOST` var (deferred).** Adding `HOST=0.0.0.0` to [core/env/admin.env](../repos/Full-Stack-Docker-Tazama/core/env/admin.env) is required by admin-service `.env.template` on dev but was skipped in `feat-paysys-fsdt-fixes` pending verification of whether admin-service actually reads it at boot in the FSDT compose context. Follow-up: confirm and add. (§6.1)
7. **admin-service `DISTRIBUTED_CACHE_ENABLED`.** FSDT sets `false`; the Jenkins working-system env sets `true`. Left as-is per project owner. Revisit if distributed cache is required for the release. (§6.1)
8. **connection-studio env.** `DEMS_URL`, `DEAPI_URL`, `DEMS_STREAM` added in `feat-paysys-fsdt-fixes`. `SESSION_TIMEOUT_MINUTES=30` already present in FSDT tcs.env (L14). `ENCRYPTION_KEY` in tcs.env is a per-service TCS-internal secret (encrypts SFTP passwords at rest — not shared with rule-studio) — rotate per-client. (§6.3)
9. **event-monitoring-service core compose.** Add a core-level compose + env for standalone Server A deploys if required by the release topology. (§6.4)
10. **frms-coe-lib / frms-coe-startup-lib version alignment.** Bump event-monitoring-service and connection-studio/backend from `8.0.0-rc.5` / `3.0.2-rc.5` to `8.2.0-rc.6` / `3.1.0-rc.8` (matching admin-service & rule-executer), or document intentional drift. (§6.5, §6.6)
11. **Audit `STARTUP_TYPE` values** — every FSDT env file must use `nats`; legacy transports are removed upstream. (§6.6)
12. **Live-DB anomaly** — the migration `20260601091334` is present on tazama-b's `tazama_cms._prisma_migrations` but exists on no CMS branch. Applied 2026-06-03 with no logs. Not FSDT's problem to reproduce, but the DB owner should investigate. (§3.3)
13. **rule-studio `send-to-dems` BullMQ coupling (future work).** `SendToDemsService` on rule-studio `dev` is still fully coupled to BullMQ despite the design intent that it be queue-independent. Trickles down to `fetch-from-dlh` and `rerun-simulation`. Details and refactor plan in [BullMQ-findings.md §7](BullMQ-findings.md#7-intent-gap--send-to-dems-should-not-use-bullmq-future-work). Not blocking this release; deferred.
14. **rule-studio `helpers.encrypt/decrypt` dead code (future work).** `ENCRYPTION_KEY` and `IV_LENGTH` in trs.env feed rule-studio's `helpers.ts` `encrypt/decrypt` functions, but no source file imports those functions. Cleanup: either wire them into a real callsite or remove the helpers plus the env vars. Not blocking. (§5.3)

[Back to top](#top)

---

## 8. Appendix — repo remote & branch state

Captured after the sync described in §1.

| Repo | Remote | Branch at report time |
|------|--------|------------------------|
| admin-service | `git@github.com:tazama-lf/admin-service.git` | dev |
| biar | `git@github.com:tazama-lf/biar.git` | dev |
| case-management-system | `git@github.com:tazama-lf/case-management-system.git` | dev |
| connection-studio | `git@github.com:tazama-lf/connection-studio.git` | dev |
| event-monitoring-service | `git@github.com:tazama-lf/event-monitoring-service.git` | dev |
| frms-coe-lib | `git@github.com:tazama-lf/frms-coe-lib.git` | dev |
| frms-coe-startup-lib | `git@github.com:tazama-lf/frms-coe-startup-lib.git` | dev |
| Full-Stack-Docker-Tazama | `git@github.com:tazama-lf/Full-Stack-Docker-Tazama.git` | dev |
| JupyterNotebooks | `git@github.com:ahmad-paysys/UAT-Tazama-documentation.git` | main |
| paysys-pmo | `git@github.com:frmscoe/paysys-pmo.git` | dev |
| rule-executer | `git@github.com:tazama-lf/rule-executer.git` | dev |
| rule-studio | `git@github.com:tazama-lf/rule-studio.git` | dev |
| rule-studio-devtestops | `git@github.com:tazama-lf/rule-studio-devtestops.git` | dev |
| rule-studio-example | `git@github.com:tazama-lf/rule-studio-example.git` | dev |
| tcs-lib | `git@github.com:tazama-lf/tcs-lib.git` | dev |

Any dirty working trees at review time (biar, case-management-system, rule-studio, rule-studio-devtestops, rule-studio-example) were auto-stashed with a timestamped label before checkout — restore with `git stash list | grep auto-stash` in each repo.

[Back to top](#top)
