# GitHub Issue Comment — Issue #225

## User Story

**Title:** Add NiFi `QueryDatabaseTableRecord` processor for `tazama_cms.cms_usernames`

**Body:**
The Case Management Trend Dashboard shows raw `user_id` UUIDs instead of investigator names because the Hudi gold table `gold/cms_usernames` is empty. Every upstream stage is in place on `dev` (CMS Postgres table, login-time upsert reading generic OIDC claims, biar `CmsUsernamesETL`, notebook left-join). The single missing link is a NiFi `QueryDatabaseTableRecord` processor for `tazama_cms.cms_usernames` in `biar/nifi/tazama.xml`. Every other `tazama_cms` table (`alerts`, `cases`, `tasks`, `comments`) has one. Fixes are already prepared in [Full-Stack-Docker-Tazama PR #236](https://github.com/tazama-lf/Full-Stack-Docker-Tazama/pull/236) and [biar PR #89](https://github.com/tazama-lf/biar/pull/89) — both **OPEN**. Merge them, redeploy NiFi, and names will resolve for anyone who has logged into CMS at least once. Same root cause as [biar #112](https://github.com/tazama-lf/biar/issues/112).

**Scope:**
- Files: `biar/nifi/tazama.xml` in **Full-Stack-Docker-Tazama** and in **biar** (mirror copies must merge together).
- Change: add one `QueryDatabaseTableRecord` processor for `tazama_cms.cms_usernames` (uses existing pool `b0b69311-8545-36e2`, `Maximum-value Columns = updated_at`, `Fetch Size = 500`) + supporting `RemoteProcessGroup` `table=cms_usernames` entries. Same shape as the existing `alerts`/`cases`/`tasks`/`comments` processors.
- Migration required: No (the CMS-side Prisma migration is already on `dev`)
- Frontend changes: No
- Backend code changes: No

**Acceptance Criteria:**
- [ ] Full-Stack-Docker-Tazama PR #236 merged to `dev`.
- [ ] biar PR #89 merged to `dev`.
- [ ] After NiFi redeploy in UAT + one CMS login by a known user, `SELECT COUNT(*) FROM tazama_cms.cms_usernames` ≥ 1 **and** the Hudi table at `gold/cms_usernames` has ≥ 1 row.
- [ ] `Case_Management_Trend_Dashboard.ipynb` shows the investigator's display name in "TOP 5 INVESTIGATORS BY WORKLOAD".
- [ ] "Name not found in cms_usernames — showing ID" fallback line does not appear for users who have logged in.
- [ ] Known limitations documented in the UAT release note: users who have never logged into CMS, or whose IAM name was changed without a subsequent login, will still show UUIDs / stale names respectively — this is inherent to the login-driven mirror pattern and cannot be fixed inside CMS (IAM-agnostic constraint).

**Testing:**
- Confirm NiFi UI shows the new `cms_usernames` processor with In > 0 within one poll interval after a CMS login.
- Confirm automation-orchestrator logs show `[CmsUsernamesETL] Ingesting … → …/gold/cms_usernames`.
- Confirm `spark.read.format("hudi").load(f"{WAREHOUSE_ROOT}/gold/cms_usernames").show()` returns ≥ 1 row.
- Smoke-test existing `alerts` / `cases` / `tasks` / `comments` ingests are unaffected (PR is a full NiFi template re-export and touches `versionedComponentId`s across the file).
- Manually verify a rename-without-login case still shows the old name — this is the expected known limitation, not a bug.

**Note — no event-driven follow-up in CMS:**
An event-driven identity sync would fix the residual "user never logged in" / "renamed without login" / "disabled user" gaps. It is **not** proposed here because it would require IAM-vendor-specific code, and the CMS repository must remain a plug-and-play IAM consumer. If those gaps ever need to be closed, it must be done outside CMS (external identity-sync service, or a NiFi processor tapping a standardized IAM export, or a generic REST intake endpoint fed by a deployment-side connector) — but none of that is required to close #225.
