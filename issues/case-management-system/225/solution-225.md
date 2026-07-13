# Solution — Issue #225: BIAR Bug: Case Management Trend Dashboard — no investigator data displayed

## Complete Fix — Exact Code Changes

Track A is a NiFi template patch across two repos with prepared PRs already open. There is no Track B — see [issue-225.md](issue-225.md) and [impact-225.md](impact-225.md) for why (CMS must remain IAM-agnostic; any staleness-closing work must live outside CMS and is out of scope for #225).

---

### Track A — Merge NiFi processor patch

#### File 1 — `Full-Stack-Docker-Tazama/biar/nifi/tazama.xml`

The change is delivered by [Full-Stack-Docker-Tazama PR #236](https://github.com/tazama-lf/Full-Stack-Docker-Tazama/pull/236). NiFi's XML export rewrites `versionedComponentId`s across the entire template, so the PR diff is ~48k lines — but the *meaningful* additions are the ones below. Reviewers should focus on these.

**BEFORE — no `cms_usernames` processor exists**

```bash
$ grep -c 'cms_usernames' repos/Full-Stack-Docker-Tazama/biar/nifi/tazama.xml
0
```

The set of `<key>Table Name</key><value>…</value>` entries in `tazama.xml` on `dev` is:

```
account, account_holder, alerts, cases, comments, condition, entity,
evaluation, network_map, rule, tasks, transaction, transaction_data, typology
```

**AFTER — new `QueryDatabaseTableRecord` processor added**

The key additions from PR #236:

```xml
<!-- Inside the tazama_cms process group -->
<processor>
    <id>...</id>
    <name>QueryDatabaseTableRecord</name>
    <type>org.apache.nifi.processors.standard.QueryDatabaseTableRecord</type>
    <properties>
        <entry>
            <key>Database Connection Pooling Service</key>
            <value>b0b69311-8545-36e2</value>  <!-- existing tazama_cms pool -->
        </entry>
        <entry>
            <key>Table Name</key>
            <value>cms_usernames</value>
        </entry>
        <entry>
            <key>Maximum-value Columns</key>
            <value>updated_at</value>
        </entry>
        <entry>
            <key>Fetch Size</key>
            <value>500</value>
        </entry>
        <entry>
            <key>Max Rows Per Flow File</key>
            <value>500</value>
        </entry>
    </properties>
</processor>
```

Plus supporting `RemoteProcessGroup` `<key>table</key><value>cms_usernames</value>` entries alongside the existing entries for `alerts`, `cases`, `tasks`, and `comments`, so the batch is routed to the automation orchestrator with `table=cms_usernames`.

**What this does and why:** the automation orchestrator's `_ETL_REGISTRY` in [automation-orchestrator/lakehouse_automation_pipeline.py:52-69](../../../repos/biar/automation-orchestrator/lakehouse_automation_pipeline.py#L52-L69) dispatches by table name; without a NiFi processor emitting `table=cms_usernames`, `CmsUsernamesETL.run()` is never called and `gold/cms_usernames` stays empty. Adding this processor makes the ETL dispatch fire.

#### File 2 — `biar/nifi/tazama.xml` (mirror in biar repo)

Delivered by [biar PR #89](https://github.com/tazama-lf/biar/pull/89). Same patch as File 1, applied to the mirror copy that lives in the biar repo. Merge both to keep the two copies in lockstep.

#### Impact of Track A

- `gold/cms_usernames` populates after NiFi's next poll.
- The Case Management Trend Dashboard "TOP 5 INVESTIGATORS BY WORKLOAD" table shows names for users who have logged in since deployment.
- The notebook's fallback line ("Name not found in cms_usernames — showing ID…") stops firing for logged-in users.
- Track A does not solve name freshness for users who don't log in, and does not solve the disabled-user case. These are captured as known limitations in [issue-225.md](issue-225.md) and cannot be closed in CMS itself (IAM-agnostic constraint).

---

## Test Cases

### Integration / E2E scenarios (UAT — Track A verification)

1. **Merge PRs #236 and #89** to `dev` and redeploy NiFi in UAT.
2. **Verify processor is active:** in NiFi UI, confirm the `cms_usernames` `QueryDatabaseTableRecord` processor is running and has a positive "In" count within one poll interval after a CMS login.
3. **Trigger a login:** log a known investigator user (say, the CIMS ODS Tazama-tenant investigator Sandy referenced) into the CMS UI. Confirm one row appears in `tazama_cms.cms_usernames`:
   ```sql
   SELECT id, user_id, name, tenant_id, created_at, updated_at
   FROM tazama_cms.cms_usernames
   WHERE tenant_id = 'TAZAMA';
   ```
4. **Confirm ETL fired:** in the automation-orchestrator logs, look for `[CmsUsernamesETL] Ingesting … → …/gold/cms_usernames` and `[CmsUsernamesETL] Gold written → …`.
5. **Confirm Hudi gold table has the row:** in a Spark session,
   ```python
   spark.read.format("hudi").load(f"{WAREHOUSE_ROOT}/gold/cms_usernames").show()
   ```
6. **Run the notebook:** open `Case_Management_Trend_Dashboard.ipynb` in JupyterHub against UAT, execute all cells for the "Last 90 days" period. In section 7 ("Workload & Investigator Productivity"), confirm names appear in the TOP 5 list; no "Name not found in cms_usernames" fallback lines for logged-in users.

### Manual / UAT checks

| # | Scenario | Steps | Expected Result |
|---|---|---|---|
| 1 | Freshly deployed Track A, single logged-in investigator | Log in as investigator X, wait one NiFi poll interval, refresh notebook | Investigator X's name (not UUID) appears in TOP 5 by workload |
| 2 | Multiple investigators with cases, only some logged in | Log in as X, do not log in as Y (both own cases in DLH) | X shows by name; Y shows the "Name not found — showing ID: {UUID}" fallback |
| 3 | Investigator renamed in IAM, then logs in | Rename in IAM → log in → wait one NiFi poll → refresh notebook | Notebook shows new name (login-time upsert path updates row) |
| 4 | Rename in IAM, no login (known limitation) | Rename in IAM, do NOT log in, wait, refresh notebook | Notebook shows the *old* name — this is the documented known limitation, not a bug |
| 5 | Smoke test — existing ingest unaffected | Verify `alerts`, `cases`, `tasks`, `comments` still populate `gold/` tables after deploy | No regression in existing dashboards |

---

## Overall Impact of the Fix

| Area | Before | After (Track A) |
|---|---|---|
| `gold/cms_usernames` | Empty | Populated for logged-in users |
| Case Management Trend Dashboard "TOP 5 INVESTIGATORS" | Shows UUIDs and "Name not found" fallback | Shows names for logged-in users; UUIDs for others (documented limitation) |
| Investigator name freshness | N/A (empty) | On-login only (documented limitation) |
| Disabled users in DLH | Not surfaced | Not surfaced (documented limitation) |
| `CmsUsernamesETL` code | Dead-code (never invoked) | Active |
| CMS ↔ IAM coupling | None (login-time upsert reads generic OIDC claims) | Unchanged — no IAM-specific code introduced |

---

## Fix Summary

The Case Management Trend Dashboard shows no investigator names because the Hudi gold table it joins against (`gold/cms_usernames`) is empty. Every other piece of the plumbing is in place on `dev` — CMS writes a row to a Postgres `cms_usernames` table on every successful login (already merged, June 2026), a Spark ETL is registered to consume `cms_usernames` batches and write them to Hudi (already merged), and the notebook already knows how to left-join on `user_id` and fall back gracefully to UUIDs when the name is missing. The one missing link is a single `QueryDatabaseTableRecord` processor in the NiFi ingest template — the piece that would pull rows out of Postgres and hand them to the automation orchestrator.

Track A is: merge the two open PRs that add that NiFi processor (Full-Stack-Docker-Tazama #236 and biar #89), redeploy NiFi, and confirm names appear. This is a ~2-hour operation and fixes the issue as literally reported by Sandy. It is safe to ship in isolation and has no overlap with the open issues #214 or #220.

An event-driven identity sync — the natural follow-up that would fix the residual "user never logged in" and "renamed in IAM without re-login" gaps — is not proposed here because it would require IAM-vendor-specific code, and the CMS repository must remain a plug-and-play IAM consumer. Those gaps are captured as known limitations. If they ever need to be closed, it must be done outside CMS (an external identity-sync service, a NiFi processor against a standardized IAM export, or a generic REST intake endpoint fed by a deployment-side connector) — but none of that is required to close #225.

For a non-technical reader: the dashboard was blind because one XML file was missing one entry. That entry is written and waiting in a PR. Once merged, names will start appearing for anyone who has logged in. Names for users who *haven't* logged in are a documented limitation, not something CMS can fix without breaking its plug-and-play IAM contract.
