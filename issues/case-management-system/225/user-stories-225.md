# User Stories — Issue #225

Each story below is a discrete piece of work. Track A is a single story (US-225-1). Track B is broken into five stories (US-225-2 … US-225-6). Order matters — see the "Depends on" line at the top of each story.

---

## US-225-1 — Add NiFi processor for `cms_usernames`

**Track:** A
**Depends on:** none — this is the top of the tree.

**Body**
The Case Management Trend Dashboard shows raw UUIDs where investigator names should appear because the Hudi gold table `gold/cms_usernames` is empty. Every upstream stage is in place except one: `biar/nifi/tazama.xml` has no `QueryDatabaseTableRecord` processor for `tazama_cms.cms_usernames`. Two PRs already exist ([Full-Stack-Docker-Tazama #236](https://github.com/tazama-lf/Full-Stack-Docker-Tazama/pull/236) and [biar #89](https://github.com/tazama-lf/biar/pull/89)) that add the processor. This story is: review, merge, and deploy.

**Scope**
- Repo(s): `Full-Stack-Docker-Tazama`, `biar`
- Files: `biar/nifi/tazama.xml` (both copies)
- Change: XML template — add one `QueryDatabaseTableRecord` processor + supporting connections/RemoteProcessGroup entries for `cms_usernames`
- Migration required: No
- Frontend changes: No

**Acceptance Criteria**
- [ ] Full-Stack-Docker-Tazama PR #236 merged to `dev`.
- [ ] biar PR #89 merged to `dev`.
- [ ] NiFi redeployed to UAT with the new template.
- [ ] `grep -c 'cms_usernames' biar/nifi/tazama.xml` on `dev` returns > 0.
- [ ] After one CMS login in UAT, `SELECT COUNT(*) FROM tazama_cms.cms_usernames` ≥ 1.
- [ ] Within one NiFi poll interval, the Hudi table at `gold/cms_usernames` has ≥ 1 row.
- [ ] `Case_Management_Trend_Dashboard.ipynb` shows the investigator's display name in "TOP 5 INVESTIGATORS BY WORKLOAD".

**Testing**
- In NiFi UI, confirm the new `QueryDatabaseTableRecord` processor is running and has In > 0 within one poll interval.
- Automation-orchestrator logs show `[CmsUsernamesETL] Ingesting … → …/gold/cms_usernames`.
- `spark.read.format("hudi").load(f"{WAREHOUSE_ROOT}/gold/cms_usernames").show()` returns rows.
- Smoke-test existing `alerts`/`cases`/`tasks`/`comments` ingest is unaffected (PR is a full template re-export).

---

## US-225-2 — Add sync-tracking columns to `cms_usernames`

**Track:** B
**Depends on:** US-225-1 (Track A must land first)

**Body**
To support event-driven identity sync we need columns that track when a row was last synced from Keycloak and whether the underlying Keycloak user is disabled. This is a purely additive migration — the login-time upsert path continues to work throughout the deployment window.

**Scope**
- Repo: `case-management-system`
- Files:
  - `backend/prisma/migrations/{new-timestamp}_cms_usernames_sync_fields/migration.sql` (new)
  - `backend/prisma/schema.prisma` — update `cms_usernames` model
- Change: ALTER TABLE — three additive nullable/defaulted columns + one index
- Migration required: Yes (additive)
- Frontend changes: No

**Acceptance Criteria**
- [ ] Migration adds `last_synced_at TIMESTAMP(6) NULL`, `disabled_at TIMESTAMP(6) NULL`, `source TEXT NOT NULL DEFAULT 'keycloak-event'` to `tazama_cms.cms_usernames`.
- [ ] Prisma schema updated to reflect the new columns.
- [ ] `SELECT column_name FROM information_schema.columns WHERE table_name='cms_usernames'` shows the new columns.
- [ ] Existing login-time upsert continues to succeed against the migrated schema without changes.
- [ ] Index on `disabled_at` created.

**Testing**
- Apply migration in UAT; log in with an existing user; confirm the login-time upsert continues to write the row (with default `source='keycloak-event'` and `last_synced_at=NULL`).
- Confirm no query regressions on `cms_usernames` reads by biar ETL.

---

## US-225-3 — `KeycloakSyncService` (event listener)

**Track:** B
**Depends on:** US-225-2

**Body**
Add a NestJS service that receives Keycloak user CRUD events (via SPI webhook or Kafka consumer — pick one during design) and upserts `cms_usernames` accordingly. `USER_CREATE` / `USER_UPDATE` upsert; `USER_DELETE` / `USER_DISABLE` set `disabled_at`. Every write refreshes `last_synced_at`.

**Scope**
- Repo: `case-management-system`
- Files:
  - `backend/src/modules/identity-sync/keycloak-sync.service.ts` (new)
  - `backend/src/modules/identity-sync/keycloak-sync.module.ts` (new)
  - `backend/src/modules/identity-sync/keycloak-sync.controller.ts` (new — only if using webhook transport)
  - `backend/src/modules/identity-sync/keycloak-sync.service.spec.ts` (new — tests)
  - `backend/src/app.module.ts` — wire the module
- Change: New Nest module + service + tests
- Migration required: No (relies on US-225-2)
- Frontend changes: No

**Acceptance Criteria**
- [ ] `USER_CREATE` event → new row in `cms_usernames` with `source='keycloak-event'`, `last_synced_at` set.
- [ ] `USER_UPDATE` event with a new name → existing row's `name` updated, `last_synced_at` set.
- [ ] `USER_UPDATE` event with no name change → `last_synced_at` still updated (heartbeat).
- [ ] `USER_DELETE` or `USER_DISABLE` → `disabled_at` set, `last_synced_at` refreshed.
- [ ] Handler is idempotent — replaying the same event twice produces the same terminal state.
- [ ] Unit tests cover all four event types plus the idempotency case.
- [ ] Login flow is unaffected — this story does not touch `auth.service.ts` yet.

**Testing**
- Fire each event type against a UAT instance; verify row state after each.
- Fire an event twice; verify no row duplication and `last_synced_at` moves forward on both fires.

---

## US-225-4 — Backfill script for pre-existing Keycloak users

**Track:** B
**Depends on:** US-225-3

**Body**
Users who existed in Keycloak before Track B deployment need to be backfilled into `cms_usernames`. Add an idempotent one-shot script that iterates Keycloak's Admin API and drives `KeycloakSyncService.handleEvent({ eventType: 'USER_CREATE', … })` for each user.

**Scope**
- Repo: `case-management-system`
- Files: `backend/scripts/backfill-cms-usernames.ts` (new)
- Change: New script; no service or schema changes
- Migration required: No
- Frontend changes: No

**Acceptance Criteria**
- [ ] Running the script populates `cms_usernames` with one row per Keycloak user, sourced with a distinguishing marker (e.g. `source='backfill'`).
- [ ] Script is idempotent — running twice does not duplicate rows.
- [ ] Failure on one user does not abort the whole run — errors are logged per user.
- [ ] After the run, `SELECT COUNT(*) FROM cms_usernames` matches the Keycloak Admin API user count for the tenant.

**Testing**
- Run against UAT; compare `cms_usernames` count to Keycloak `/admin/realms/{realm}/users?briefRepresentation=true` count.
- Re-run script; confirm no new rows added.

---

## US-225-5 — Remove login-time upsert from `auth.service.ts`

**Track:** B
**Depends on:** US-225-3, US-225-4 (event feed must be live and backfill must have run before this lands)

**Body**
Once the event listener is producing rows and the backfill has completed, the login-driven upsert is redundant and violates separation of concerns (per Umair's user-story comment on the issue). Remove the call and the private method.

**Scope**
- Repo: `case-management-system`
- Files: [backend/src/modules/auth/auth.service.ts:60-68, 267-301](../../../repos/case-management-system/backend/src/modules/auth/auth.service.ts#L60-L68)
- Change: Delete the `storeUserName(token).catch(…)` block and the entire `private async storeUserName(...)` method
- Migration required: No
- Frontend changes: No

**Acceptance Criteria**
- [ ] `auth.service.ts` no longer references `cms_usernames` or `storeUserName`.
- [ ] Login flow still succeeds end-to-end.
- [ ] Existing users retain fresh names in `cms_usernames` after this deploy (proving the event feed is the sole writer).
- [ ] No regressions in the notebook — names still appear.

**Testing**
- Log in as a user; confirm login succeeds and no DB writes to `cms_usernames` originate from `auth.service`.
- Rename user in Keycloak (no login); confirm `cms_usernames.name` updates within one sync interval.

---

## US-225-6 — (Optional) Switch Hudi record key from `id` to `user_id`

**Track:** B
**Depends on:** US-225-5 (or US-225-3 if run in parallel — no code dependency, just easier to schedule after Track B backend stabilises)

**Body**
biar issue #112 flagged that `CmsUsernamesETL` uses Hudi record key = `id` (the Postgres SERIAL surrogate) rather than `user_id`. In practice the notebook already de-duplicates via a windowed `row_number`, but switching to `user_id` gives Hudi the correct dedup semantics and removes the safety-net dependency. Requires a one-time full reload of `gold/cms_usernames`.

**Scope**
- Repo: `biar`
- Files: [automation-orchestrator/Table_ETLs/cms_usernames.py:54-62](../../../repos/biar/automation-orchestrator/Table_ETLs/cms_usernames.py#L54-L62)
- Change: `record_key="id"` → `record_key="user_id"`; `precombine="created_at"` → `precombine="updated_at"`
- Migration required: No (but requires Hudi table reload)
- Frontend changes: No

**Acceptance Criteria**
- [ ] `Table_ETLs/cms_usernames.py` uses `record_key="user_id"` and `precombine="updated_at"`.
- [ ] `gold/cms_usernames` fully reloaded from the current CMS Postgres state.
- [ ] After reload, `SELECT user_id, COUNT(*) FROM gold_cms_usernames GROUP BY user_id HAVING COUNT(*) > 1` (or Spark equivalent) returns zero rows.
- [ ] Notebook still displays names correctly.

**Testing**
- Trigger a full reload of `gold/cms_usernames` after the ETL change is deployed.
- Verify no duplicate `user_id` rows in the reloaded table.
- Re-run notebook and confirm names still appear.
