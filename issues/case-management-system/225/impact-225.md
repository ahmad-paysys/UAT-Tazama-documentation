# Impact Study — Issue #225: BIAR Bug: Case Management Trend Dashboard — no investigator data displayed

**Issue:** [#225](https://github.com/tazama-lf/case-management-system/issues/225)
**Study Date:** 2026-07-10
**Author:** Ahmad Khalid
**Repository:** tazama-lf/case-management-system

---

## Summary

The Case Management Trend Dashboard cannot resolve `case_owner_user_id` UUIDs to investigator display names because the Hudi gold table `gold/cms_usernames` is empty. Every piece of the pipeline exists on `dev` — CMS writes to Postgres on login, an ETL is registered to write to the gold table, the notebook already joins by `user_id` — except one: NiFi has no `QueryDatabaseTableRecord` processor for `tazama_cms.cms_usernames`, so batches never reach the ETL. A patch is ready in Full-Stack-Docker-Tazama PR [#236](https://github.com/tazama-lf/Full-Stack-Docker-Tazama/pull/236) and biar PR [#89](https://github.com/tazama-lf/biar/pull/89) but neither has been merged. Track A is: merge those PRs and redeploy NiFi. Track B is: replace the login-driven upsert with an event-driven Keycloak identity sync so names stay current for users who don't log in.

---

## Confirmed Root Cause

- ✅ `tazama_cms.cms_usernames` table exists — [backend/prisma/migrations/20260616124311_username/migration.sql](../../../repos/case-management-system/backend/prisma/migrations/20260616124311_username/migration.sql) creates it; [backend/prisma/schema.prisma:546-555](../../../repos/case-management-system/backend/prisma/schema.prisma#L546-L555) models it.
- ✅ CMS auth service writes to it on every login — [backend/src/modules/auth/auth.service.ts:60-68](../../../repos/case-management-system/backend/src/modules/auth/auth.service.ts#L60-L68) fires `storeUserName(token)` after each successful login; the upsert body is at L267–301.
- ✅ `CmsUsernamesETL` exists and is registered — [automation-orchestrator/Table_ETLs/cms_usernames.py:15-65](../../../repos/biar/automation-orchestrator/Table_ETLs/cms_usernames.py#L15-L65), registered at [automation-orchestrator/lakehouse_automation_pipeline.py:52-69](../../../repos/biar/automation-orchestrator/lakehouse_automation_pipeline.py#L52-L69).
- ✅ Notebook already reads and joins — cell 1 loads `f"{WAREHOUSE_ROOT}/gold/cms_usernames"`; cell 14 left-joins on `case_owner_user_id == user_id` and picks the row with the latest `updated_at` per user.
- ❌ NiFi `QueryDatabaseTableRecord` processor for `cms_usernames` is **absent** from `dev` — `grep -c 'cms_usernames' repos/Full-Stack-Docker-Tazama/biar/nifi/tazama.xml` returns `0`. Every other `tazama_cms` table (`alerts`, `cases`, `tasks`, `comments`) has one.
- ✅ Fix has been prepared but not merged — Full-Stack-Docker-Tazama PR #236 and biar PR #89 both remain **OPEN**.

This is a full independent confirmation of biar issue [#112](https://github.com/tazama-lf/biar/issues/112), which Sandy asked to be checked in the 2026-07-07 comment. #112's diagnosis is correct; #225 is downstream of the same defect.

---

## Track A — Merge NiFi processor patch

### What Changes

Two NiFi XML templates:
- `Full-Stack-Docker-Tazama/biar/nifi/tazama.xml` — via PR [#236](https://github.com/tazama-lf/Full-Stack-Docker-Tazama/pull/236)
- `biar/nifi/tazama.xml` — via PR [#89](https://github.com/tazama-lf/biar/pull/89)

The patch adds a `QueryDatabaseTableRecord` processor pointed at `tazama_cms.cms_usernames` using the existing `b0b69311-8545-36e2` connection pool, plus the supporting connections/RemoteProcessGroup entries that mirror the pattern already in place for `alerts`, `cases`, `tasks`, and `comments`. The PR is a full NiFi template re-export (~48k lines of XML) because NiFi rewrites `versionedComponentId`s across the file, but the meaningful additions are the `Table Name = cms_usernames` value and its wrapping processor block.

### Impact

| Dimension | Value |
|---|---|
| Files changed | 2 (both are NiFi XML templates in different repos) |
| Schema migration required | No — the Postgres migration is already on `dev` |
| Frontend changes required | No |
| Downtime required | NiFi restart in UAT (rolling; ~1 min) |
| Risk of regression | <span style="background:#d4edda;color:#155724;padding:2px 6px;border-radius:3px">Low</span> — additive template change; no changes to existing processors |
| Reversibility | Fully reversible: revert PR, redeploy NiFi |

**What Track A fixes immediately:**
- `gold/cms_usernames` starts receiving rows after the next NiFi poll interval.
- Case Management Trend Dashboard investigator names resolve for any user who has logged into CMS at least once since deployment.
- Any future dashboard that joins on `cms_usernames.user_id` starts having data.

**What Track A does not fix:**
- Users who have not logged into CMS since deployment stay absent from `cms_usernames` — the dashboard still shows UUIDs for them.
- Users whose display name is changed in Keycloak but who don't subsequently log in stay stale in the DLH.
- Disabled users are never marked as such in `cms_usernames`.
- The architectural pattern (per-service identity mirroring) is unchanged; other services that need investigator names will hit the same pattern.

Track A is safe to ship in isolation.

---

## Track B — Event-driven Keycloak identity sync

### What Changes

Replace the login-driven upsert in CMS with a Keycloak event listener (SPI or Kafka topic) that reflects Keycloak user CRUD directly into `cms_usernames`, plus a one-time backfill for dormant users. Add columns to track sync freshness and disabled status.

### Schema Impact

| Change | Table | Column | Notes |
|---|---|---|---|
| Add column | `cms_usernames` | `last_synced_at TIMESTAMP(6)` | Populated by every sync event; used to detect staleness |
| Add column | `cms_usernames` | `disabled_at TIMESTAMP(6) NULL` | Set when Keycloak disables the user; downstream queries can exclude |
| Add column | `cms_usernames` | `source TEXT NOT NULL DEFAULT 'keycloak-event'` | Distinguishes login-fallback vs. event-driven rows during migration |

Migration risk: additive columns only, no data rewrites. Safe under concurrent writes.

### Backend Code Impact

**Core rewrites**

| File | Change |
|---|---|
| [backend/src/modules/auth/auth.service.ts:60-68](../../../repos/case-management-system/backend/src/modules/auth/auth.service.ts#L60-L68) | Delete the `storeUserName(token)` call (or keep as documented fallback) |
| [backend/src/modules/auth/auth.service.ts:267-301](../../../repos/case-management-system/backend/src/modules/auth/auth.service.ts#L267-L301) | Delete `storeUserName` method |

**New files**

| File | Purpose |
|---|---|
| `backend/src/modules/identity-sync/keycloak-sync.service.ts` | Consumes Keycloak `USER_CREATE`/`USER_UPDATE`/`USER_DELETE` events; upserts `cms_usernames` |
| `backend/src/modules/identity-sync/keycloak-sync.module.ts` | Nest module registration |
| `backend/src/modules/identity-sync/keycloak-sync.controller.ts` | Webhook endpoint (if using Keycloak → CMS webhook rather than SPI) |
| `backend/scripts/backfill-cms-usernames.ts` | One-time backfill from Keycloak Admin API |
| `backend/prisma/migrations/{timestamp}_cms_usernames_sync_fields/migration.sql` | Schema additions above |

**ETL / NiFi**

| File | Change |
|---|---|
| [automation-orchestrator/Table_ETLs/cms_usernames.py:54-62](../../../repos/biar/automation-orchestrator/Table_ETLs/cms_usernames.py#L54-L62) | Consider switching `record_key="id"` → `record_key="user_id"` per biar #112's minor design note; requires a full Hudi reload |
| `biar/nifi/tazama.xml` | Add `disabled_at` to Maximum-value Columns if it becomes a sync signal |

### Frontend Code Impact

None. `Case_Management_Trend_Dashboard.ipynb` already left-joins on the resolved name and handles nulls; no CMS frontend surface consumes `cms_usernames` directly.

---

## Side Effects and Risks

### Risks of Track A Alone

| Risk | Likelihood | Mitigation |
|---|---|---|
| A user in Keycloak who has never logged into CMS is invisible in the dashboard | High (Sandy's exact 2026-05-21 observation about "only one investigator" is likely this) | Ship Track B; interim, ask supervisors to log each investigator in at least once |
| Name in Keycloak changes; user doesn't log in; dashboard shows stale name | Medium | Ship Track B; interim, document that dashboard names lag |
| NiFi template re-export in PR #236 accidentally alters other processors | Low | Careful review of the PR diff — focus on the `<value>cms_usernames</value>` blocks; run a smoke test of alerts/cases ingest after deploy |
| Hudi `record_key="id"` (serial int) means the same `user_id` could appear twice if the row is deleted+recreated | Low | Notebook already de-duplicates via `row_number() over (partition by user_id order by updated_at desc)` in cell 14 |

### Risks of Track B

| Risk | Likelihood | Mitigation |
|---|---|---|
| Keycloak event-listener SPI failure silently causes stale names | Medium | Emit sync metrics; alert on missing `last_synced_at` heartbeats |
| Migration adds columns to a table under concurrent write from login path | Low | Additive columns default-null are safe; deploy migration before switching auth.service |
| Backfill script mis-authenticates and fails partially | Medium | Idempotent backfill; log per-user upsert result; re-runnable |
| Removing login-time upsert introduces regression if event listener not fully deployed | Medium | Ship event listener + backfill first; keep login-time upsert running for one release before removing |

### Cross-Issue Dependencies

Scanned the file lists for open issues [#214](../214/issue-214.md) and [#220](../220/issue-220.md):

| Issue | Files it touches | Overlap with #225 |
|---|---|---|
| #214 (FRAUD_AND_AML container cases) | `triage.service.ts`, `case-creation.service.ts`, `case.service.ts`, `task.service.ts`, `case-reopening.service.ts`, `task-lifecycle.service.ts`, `schema.prisma` (Case model) | **None on `cms_usernames`.** Only shared file is `schema.prisma` (different model) — migration numbering is the only concern. |
| #220 (SLA vs. Priority separation) | `alert-priority.service.ts`, priority enum sweep, `schema.prisma` (Case model), notebook-side SLA fixture | **None on `cms_usernames`.** Same migration-numbering concern for `schema.prisma`. |

Track A has zero overlap with either issue. Track B's migration file needs to be sequenced after whichever of #214/#220 lands last, to avoid migration ordering conflicts — Prisma orders migrations by directory timestamp, so as long as Track B's migration folder timestamp is later, there is no conflict.

Sandy explicitly asked in the 2026-06-29 comment to hold Track B-style work on #225 pending #214/#220. Track A does not touch that surface and can go independently.

---

## Effort Estimate

| Track | Files | Effort |
|---|---|---|
| A — merge NiFi patch | 2 XML templates | ~2 hours (mostly review + deploy) |
| B — event-driven identity sync | ~6 backend files + 1 migration + 1 ETL + 1 NiFi tweak | ~6 developer-days |
| **Total** | | **~6.25 developer-days** |

---

## Acceptance Criteria (Verification Checklist)

### Track A

- [ ] Full-Stack-Docker-Tazama PR #236 merged to `dev`.
- [ ] biar PR #89 merged to `dev`.
- [ ] After NiFi redeploy + one CMS login, `SELECT COUNT(*) FROM tazama_cms.cms_usernames` ≥ 1 **and** the Hudi table at `gold/cms_usernames` contains ≥ 1 row.
- [ ] Case Management Trend Dashboard shows display names for users who have logged in at least once.
- [ ] "Name not found in cms_usernames — showing ID" fallback does not appear for logged-in users.

### Track B

- [ ] `KeycloakSyncService` receives Keycloak user CRUD events and reflects them into `cms_usernames`.
- [ ] Backfill script populates `cms_usernames` for all pre-existing Keycloak users.
- [ ] Names in the DLH update within one sync interval even when the user does not log into CMS.
- [ ] `cms_usernames.last_synced_at` is populated on every sync; `disabled_at` set when a user is disabled in Keycloak.
- [ ] Login-time upsert removed (or explicitly retained as documented fallback).

---

## Recommended Sequencing

1. **Immediately: merge Track A PRs.** They are prepared, reviewed by SamitSaleem's team, and unblock Sandy's dashboard. This is the answer to #225 as literally reported.
2. **After Track A ships:** run the notebook against UAT to confirm names appear; close #225 and biar #112.
3. **Track B is independent of #214/#220** — no file overlap. It can be scheduled whenever there is capacity; do not block Track A on it.
4. **When scheduling Track B**, coordinate with the identity/auth track — Track B is really the seed of a shared identity-sync service and other CMS-adjacent services will likely want to consume the same feed.
