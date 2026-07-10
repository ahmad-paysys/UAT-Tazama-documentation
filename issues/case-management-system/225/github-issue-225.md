# GitHub Issue & PR Templates — Issue #225

---

## GitHub Issue Body

### Problem

The Case Management Trend Dashboard in JupyterHub shows raw `user_id` UUIDs instead of investigator display names in the "Workload & Investigator Productivity" section, making supervisor-level workload and productivity attribution impossible.

### Root Cause

The Hudi gold table `gold/cms_usernames` — which the dashboard notebook joins against to resolve `case_owner_user_id` → display name — is empty. Every upstream stage is already in place on `dev`:

- CMS Postgres has a `cms_usernames` table (migration `20260616124311_username`).
- CMS `auth.service.storeUserName` upserts a row for the logged-in user from the Keycloak inner-token `name` claim on every successful login.
- A biar `CmsUsernamesETL` is registered in `_ETL_REGISTRY["cms_usernames"]` and writes to `gold/cms_usernames`.
- The dashboard notebook left-joins on `user_id` and falls back to UUIDs when no name is found.

What is missing: a `QueryDatabaseTableRecord` NiFi processor in `biar/nifi/tazama.xml` pointing at `tazama_cms.cms_usernames`. Every other `tazama_cms` table (`alerts`, `cases`, `tasks`, `comments`) has one; this table does not. This is the same defect documented in [biar #112](https://github.com/tazama-lf/biar/issues/112).

Fixes are already prepared in [Full-Stack-Docker-Tazama PR #236](https://github.com/tazama-lf/Full-Stack-Docker-Tazama/pull/236) and [biar PR #89](https://github.com/tazama-lf/biar/pull/89) but neither is merged.

### Impact

- Case Management Trend Dashboard "TOP 5 INVESTIGATORS BY WORKLOAD" prints UUIDs and the "Name not found in cms_usernames — showing ID: …" fallback for every row.
- Any future BIAR dashboard that plans to resolve investigator names hits the same wall.
- The `CmsUsernamesETL` code exists but is unreachable.
- Login-driven upsert pattern also means that investigators who never log in are absent from the DLH, and Keycloak name changes lag until the user's next login (this is the Track B concern, not the immediate #225 bug).

### Proposed Fix

**Track A — Merge NiFi processor patch (immediate)**
- Files: 2 (`biar/nifi/tazama.xml` in Full-Stack-Docker-Tazama and biar repos)
- Change: add a `QueryDatabaseTableRecord` processor for `tazama_cms.cms_usernames` mirroring the existing pattern for `alerts`/`cases`/`tasks`/`comments`.
- Migration required: No
- Effort: ~2 hours (review + deploy)

**Track B — Event-driven Keycloak identity sync (follow-up, not a ship-blocker)**
- Files: ~6 (new `KeycloakSyncService`, additive Prisma migration, remove login-time upsert in `auth.service.ts`, optional Hudi record-key change in `Table_ETLs/cms_usernames.py`)
- Change: replace login-driven upsert with Keycloak event listener + backfill; add `last_synced_at`, `disabled_at`, `source` columns.
- Migration required: Yes (additive columns)
- Effort: ~6 developer-days

### Acceptance Criteria

**Track A**
- [ ] Full-Stack-Docker-Tazama PR #236 merged to `dev`.
- [ ] biar PR #89 merged to `dev`.
- [ ] After NiFi redeploy + one CMS login, `gold/cms_usernames` contains ≥ 1 row.
- [ ] Case Management Trend Dashboard shows display names for users who have logged in.
- [ ] "Name not found in cms_usernames — showing ID" fallback no longer appears for logged-in users.

**Track B**
- [ ] `KeycloakSyncService` receives Keycloak user CRUD events and reflects them into `cms_usernames`.
- [ ] Backfill script populates `cms_usernames` for all pre-existing Keycloak users.
- [ ] Names in DLH stay fresh even for users who don't log in.
- [ ] `disabled_at` set when a user is disabled in Keycloak.
- [ ] Login-time upsert removed from `auth.service.ts` (or explicitly retained as documented fallback).

---

## PR Descriptions

### Track A — PR Title

`fix: add QueryDatabaseTableRecord for cms_usernames to NiFi template`

### Track A — PR Body

**Summary**
- Adds a `QueryDatabaseTableRecord` NiFi processor in `biar/nifi/tazama.xml` pointing at `tazama_cms.cms_usernames`, mirroring the existing pattern for `alerts`, `cases`, `tasks`, and `comments`.
- Uses the existing `tazama_cms` connection pool (`b0b69311-8545-36e2`) — no new pool required.
- Adds supporting `RemoteProcessGroup` `table=cms_usernames` entries so batches are routed to the biar automation orchestrator with the right dispatch key.
- Uses `Maximum-value Columns=updated_at`, `Fetch Size=500`, `Max Rows Per Flow File=500` to match the other CMS processors.
- No changes to `CmsUsernamesETL`, the CMS backend, or the Prisma schema — all downstream code is already in place on `dev`.

**Test Plan**
- [ ] Deploy the updated NiFi template to UAT.
- [ ] Log in as a known investigator user; verify a row appears in `tazama_cms.cms_usernames`.
- [ ] Verify NiFi processor picks up the row within one poll interval (In > 0 in NiFi UI).
- [ ] Verify automation-orchestrator logs show `[CmsUsernamesETL] Ingesting … → …/gold/cms_usernames`.
- [ ] `spark.read.format("hudi").load(f"{WAREHOUSE_ROOT}/gold/cms_usernames").show()` returns ≥ 1 row.
- [ ] Re-run `Case_Management_Trend_Dashboard.ipynb` for "Last 90 days"; confirm investigator names appear in "TOP 5 INVESTIGATORS BY WORKLOAD".
- [ ] Confirm existing `alerts`/`cases`/`tasks`/`comments` ingests still work (smoke test — the PR is a full NiFi template re-export and touches `versionedComponentId`s across the file).

**Related Issue**
- Closes [tazama-lf/case-management-system#225](https://github.com/tazama-lf/case-management-system/issues/225)
- Closes [tazama-lf/biar#112](https://github.com/tazama-lf/biar/issues/112)

**Note**
- Two mirror PRs must merge together: [Full-Stack-Docker-Tazama PR #236](https://github.com/tazama-lf/Full-Stack-Docker-Tazama/pull/236) and [biar PR #89](https://github.com/tazama-lf/biar/pull/89). The two copies of `tazama.xml` must stay in lockstep.

---

### Track B — PR Title

`feat(identity): event-driven Keycloak sync for cms_usernames`

### Track B — PR Body

**Summary**
- Adds `KeycloakSyncService` in `backend/src/modules/identity-sync/` that consumes Keycloak `USER_CREATE`, `USER_UPDATE`, `USER_DELETE`/`USER_DISABLE` events and upserts `cms_usernames` accordingly.
- Adds `last_synced_at`, `disabled_at`, `source` columns to `cms_usernames` (additive migration).
- Removes the login-time `storeUserName` upsert from `auth.service.ts`.
- Adds a one-time backfill script `backend/scripts/backfill-cms-usernames.ts` to populate `cms_usernames` for pre-existing Keycloak users.
- (Optional) Switches `CmsUsernamesETL` Hudi record key from `id` (SERIAL) to `user_id` (UUID) per biar #112's minor design note; requires a full Hudi reload of `gold/cms_usernames`.

**Test Plan**
- [ ] Apply the additive migration on UAT; confirm no locking/regression under concurrent write load.
- [ ] Trigger Keycloak `USER_CREATE` for a new user; confirm a row lands in `cms_usernames` with `source='keycloak-event'` and `last_synced_at` populated — without the user logging in.
- [ ] Rename a Keycloak user; confirm `cms_usernames.name` updates within one sync interval.
- [ ] Disable a Keycloak user; confirm `cms_usernames.disabled_at` is set.
- [ ] Run backfill script; confirm all Keycloak users are present in `cms_usernames`.
- [ ] Log in as an existing user; confirm login now succeeds without touching `cms_usernames` (regression check for the removed upsert).
- [ ] Re-run notebook; confirm names appear for users who never logged into CMS.
- [ ] Unit tests for `KeycloakSyncService` (create/update/delete/disable/idempotency).

**Migration**
- Deploy the additive schema migration before deploying the new `KeycloakSyncService` — additive columns default to safe values (NULL / `'keycloak-event'`), so the running login-time upsert continues to work during the migration window.
- Deploy `KeycloakSyncService` next; confirm event feed is receiving events.
- Run backfill script once.
- Only then remove the login-time `storeUserName` call. This ordering avoids a window where neither path writes.

**Sequencing Note**
- Track B must be merged **after** Track A. Track A ships the immediate fix and unblocks Sandy; Track B is the durable follow-up.
- Track B has no file overlap with open issues [#214](https://github.com/tazama-lf/case-management-system/issues/214) or [#220](https://github.com/tazama-lf/case-management-system/issues/220) — the shared file is `schema.prisma` but different models. Prisma orders migrations by directory timestamp, so as long as Track B's migration is timestamped after whichever of #214/#220 lands last, there is no ordering conflict.

**Related Issue**
- Follows [tazama-lf/case-management-system#225](https://github.com/tazama-lf/case-management-system/issues/225)
- Addresses architectural concerns raised in [UmairKhan-Paysys' user-story comment](https://github.com/tazama-lf/case-management-system/issues/225#issuecomment-4843752888).
