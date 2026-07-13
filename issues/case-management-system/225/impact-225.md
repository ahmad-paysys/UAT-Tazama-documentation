# Impact Study — Issue #225: BIAR Bug: Case Management Trend Dashboard — no investigator data displayed

**Issue:** [#225](https://github.com/tazama-lf/case-management-system/issues/225)
**Study Date:** 2026-07-10
**Author:** Ahmad Khalid
**Repository:** tazama-lf/case-management-system

---

## Summary

The Case Management Trend Dashboard cannot resolve `case_owner_user_id` UUIDs to investigator display names because the Hudi gold table `gold/cms_usernames` is empty. Every piece of the pipeline exists on `dev` — CMS writes to Postgres on login (via a generic OIDC-claim-reading path, IAM-agnostic), an ETL is registered to write to the gold table, the notebook already joins by `user_id` — except one: NiFi has no `QueryDatabaseTableRecord` processor for `tazama_cms.cms_usernames`, so batches never reach the ETL. A patch is ready in Full-Stack-Docker-Tazama PR [#236](https://github.com/tazama-lf/Full-Stack-Docker-Tazama/pull/236) and biar PR [#89](https://github.com/tazama-lf/biar/pull/89) but neither has been merged. Track A is: merge those PRs and redeploy NiFi. No Track B is proposed — an event-driven identity sync would need IAM-specific code, and CMS must remain IAM-agnostic; the login-driven staleness gap is documented as a known limitation instead.

---

## Confirmed Root Cause

- ✅ `tazama_cms.cms_usernames` table exists — [backend/prisma/migrations/20260616124311_username/migration.sql](../../../repos/case-management-system/backend/prisma/migrations/20260616124311_username/migration.sql) creates it; [backend/prisma/schema.prisma:546-555](../../../repos/case-management-system/backend/prisma/schema.prisma#L546-L555) models it.
- ✅ CMS auth service writes to it on every login — [backend/src/modules/auth/auth.service.ts:60-68](../../../repos/case-management-system/backend/src/modules/auth/auth.service.ts#L60-L68) fires `storeUserName(token)` after each successful login; the upsert body is at L267–301. Reads generic OIDC claims (`sub`, `name`, `tenant_id`) via `extractInnerToken` — no IAM-vendor coupling.
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
| Backend code changes required | No |
| Frontend changes required | No |
| Downtime required | NiFi restart in UAT (rolling; ~1 min) |
| Risk of regression | <span style="background:#d4edda;color:#155724;padding:2px 6px;border-radius:3px">Low</span> — additive template change; no changes to existing processors |
| Reversibility | Fully reversible: revert PR, redeploy NiFi |

**What Track A fixes immediately:**
- `gold/cms_usernames` starts receiving rows after the next NiFi poll interval.
- Case Management Trend Dashboard investigator names resolve for any user who has logged into CMS at least once since deployment.
- Any future dashboard that joins on `cms_usernames.user_id` starts having data.

**What Track A does not fix (known limitations — see below):**
- Users who have not logged into CMS since deployment stay absent from `cms_usernames`.
- Users whose display name is changed in IAM but who don't subsequently log in stay stale in the DLH.
- Disabled users are never marked as such in `cms_usernames`.

Track A is safe to ship in isolation.

---

## No Track B is proposed

An event-driven identity sync (e.g. an IAM event listener writing into `cms_usernames`, or an admin-API poll) would eliminate the login-driven staleness. It is not proposed here because it would require IAM-vendor-specific code, and the CMS repository is required to be a plug-and-play IAM consumer with no vendor coupling.

If the staleness gap ever needs to be closed, it must be done **outside CMS** — options include:

- An external identity-sync service (in the deployment / ops repo) that talks to the IAM and writes directly to `tazama_cms.cms_usernames`, treating CMS's schema as an exported contract.
- A NiFi processor that pulls from a standardized IAM export view/API into `gold/cms_usernames`, bypassing CMS entirely.
- A generic REST intake endpoint on CMS (e.g. `POST /internal/user-sync`) fed by a deployment-side connector; CMS accepts the payload without knowing which IAM sent it.

None of these are in scope for #225 — the immediate report is fully resolved by Track A. The staleness gap is captured under "Known Limitations" below so future stakeholders understand what Track A does and does not cover.

---

## Known Limitations After Track A

| Limitation | Cause | Effect on dashboard | Operational workaround |
|---|---|---|---|
| User exists in IAM but has never logged into CMS | `cms_usernames` is written only at login time | Cases owned by this user show UUID, not name | Have each active investigator log into CMS once after deployment |
| Name changed in IAM without subsequent CMS login | Same | Dashboard shows the previous name | User re-logs in; or accept lag |
| User disabled in IAM | No disable signal reaches `cms_usernames` | Disabled user still appears in reports | Manual filter downstream, or accept |

These limitations are inherent to the login-driven mirror pattern. Closing them requires an out-of-CMS component (see "No Track B is proposed" above); no in-CMS fix is compatible with the plug-and-play IAM contract.

---

## Side Effects and Risks

### Risks of Track A

| Risk | Likelihood | Mitigation |
|---|---|---|
| A user in IAM who has never logged into CMS is invisible in the dashboard | High (Sandy's exact 2026-05-21 observation about "only one investigator" is likely this) | Log each active investigator into CMS once; accept as known limitation |
| Name in IAM changes; user doesn't log in; dashboard shows stale name | Medium | Documented known limitation |
| NiFi template re-export in PR #236 accidentally alters other processors | Low | Careful review of the PR diff — focus on the `<value>cms_usernames</value>` blocks; run a smoke test of alerts/cases ingest after deploy |
| Hudi `record_key="id"` (serial int) means the same `user_id` could appear twice if the row is deleted+recreated | Low | Notebook already de-duplicates via `row_number() over (partition by user_id order by updated_at desc)` in cell 14 |

### Cross-Issue Dependencies

Scanned the file lists for open issues [#214](../214/issue-214.md) and [#220](../220/issue-220.md):

| Issue | Files it touches | Overlap with #225 |
|---|---|---|
| #214 (FRAUD_AND_AML container cases) | `triage.service.ts`, `case-creation.service.ts`, `case.service.ts`, `task.service.ts`, `case-reopening.service.ts`, `task-lifecycle.service.ts`, `schema.prisma` (Case model) | **None.** #225 Track A is NiFi-only; no CMS source or schema changes. |
| #220 (SLA vs. Priority separation) | `alert-priority.service.ts`, priority enum sweep, `schema.prisma` (Case model), notebook-side SLA fixture | **None.** Same — no shared files. |

Track A has zero overlap with either open issue and can ship independently.

Sandy explicitly asked in the 2026-06-29 comment to hold Track B-style work on #225 pending #214/#220. That constraint is now moot because there is no Track B; Track A can proceed at any time.

---

## Effort Estimate

| Track | Files | Effort |
|---|---|---|
| A — merge NiFi patch | 2 XML templates | ~2 hours (mostly review + deploy) |
| **Total** | | **~2 hours** |

---

## Acceptance Criteria (Verification Checklist)

### Track A

- [ ] Full-Stack-Docker-Tazama PR #236 merged to `dev`.
- [ ] biar PR #89 merged to `dev`.
- [ ] After NiFi redeploy + one CMS login, `SELECT COUNT(*) FROM tazama_cms.cms_usernames` ≥ 1 **and** the Hudi table at `gold/cms_usernames` contains ≥ 1 row.
- [ ] Case Management Trend Dashboard shows display names for users who have logged in at least once.
- [ ] "Name not found in cms_usernames — showing ID" fallback does not appear for logged-in users.
- [ ] Known limitations recorded in the UAT release note so stakeholders know to expect UUIDs for users who have never logged in.

---

## Recommended Sequencing

1. **Immediately: merge Track A PRs.** They are prepared, reviewed by SamitSaleem's team, and unblock Sandy's dashboard. This is the answer to #225 as literally reported.
2. **After Track A ships:** run the notebook against UAT to confirm names appear; close #225 and biar #112.
3. **Document known limitations** in the UAT release notes; if any of them ever becomes an operational blocker, treat it as a new (out-of-CMS) work item — do not reopen #225.
