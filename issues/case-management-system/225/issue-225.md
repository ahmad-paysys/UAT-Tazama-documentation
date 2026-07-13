# Issue #225 — BIAR Bug: Case Management Trend Dashboard — no investigator data displayed

**Repository:** tazama-lf/case-management-system
**Issue:** [BIAR Bug: Case Management Trend Dashboard - no investigator data displayed](https://github.com/tazama-lf/case-management-system/issues/225)
**Author:** Sandy-at-Tazama
**State:** Open
**Report Date:** 2026-07-10

---

## Executive Summary

The Case Management Trend Dashboard's *"Workload & Investigator Productivity"* panel shows no investigator names. The Jupyter notebook joins case rows against a Hudi gold table (`gold/cms_usernames`) to resolve `case_owner_user_id` UUIDs to display names — but that gold table is empty because NiFi never ingests it.

The upstream plumbing is otherwise complete on `dev`:
- CMS has a `cms_usernames` Postgres table (migration `20260616124311_username`) that is upserted on every successful login from the inner-token `name` claim ([backend/src/modules/auth/auth.service.ts:267–301](../../../repos/case-management-system/backend/src/modules/auth/auth.service.ts#L267-L301)). This is IAM-agnostic — it reads generic OIDC-shape claims (`sub`, `name`, `tenant_id`) via `extractInnerToken`, so any IAM producing that token shape works.
- A `CmsUsernamesETL` pass-through ETL is registered in `_ETL_REGISTRY["cms_usernames"]` in the biar automation orchestrator ([automation-orchestrator/lakehouse_automation_pipeline.py:52–69](../../../repos/biar/automation-orchestrator/lakehouse_automation_pipeline.py#L52-L69)).
- The notebook already reads from `f"{WAREHOUSE_ROOT}/gold/cms_usernames"` and left-joins by `user_id` ([JupyterHub/notebooks/Case_Management_Trend_Dashboard.ipynb](../../../repos/biar/JupyterHub/notebooks/Case_Management_Trend_Dashboard.ipynb) cell 1 & cell 14).

What is missing is a single `QueryDatabaseTableRecord` NiFi processor in `biar/nifi/tazama.xml` pointing at `tazama_cms.cms_usernames`. A patch has been prepared in Full-Stack-Docker-Tazama PR [#236](https://github.com/tazama-lf/Full-Stack-Docker-Tazama/pull/236) (mirrored as biar PR [#89](https://github.com/tazama-lf/biar/pull/89)) but neither is merged. This is the root cause originally documented in biar issue [#112](https://github.com/tazama-lf/biar/issues/112), which Sandy asked to be checked in the 2026-07-07 comment.

The fix is Track A: merge the two open NiFi processor PRs. The login-driven staleness gap (users who never log into CMS are absent from the DLH; Keycloak name changes lag until the user's next login) is a **known limitation** documented at the bottom of this issue — no in-CMS fix is proposed because CMS must remain IAM-agnostic (plug-and-play IAM contract).

---

## How It Works Today — Confirmed in Code

### 1. CMS captures the user name on every login (already on `dev`; IAM-agnostic)

[backend/src/modules/auth/auth.service.ts:60–68](../../../repos/case-management-system/backend/src/modules/auth/auth.service.ts#L60-L68) calls `storeUserName(token)` after each successful `login()`. The upsert body:

```ts
// backend/src/modules/auth/auth.service.ts:267-301
private async storeUserName(token: string): Promise<void> {
  const innerDecoded = this.authGuard.extractInnerToken(token);
  const userId = innerDecoded.sub as string;
  const name = innerDecoded.name as string;
  const tenantId = innerDecoded.tenant_id as string;

  const userDetails = await this.prisma.cms_usernames.findFirst({
    where: { user_id: userId },
  });

  if (userDetails) {
    if (userDetails.name !== name) {
      await this.prisma.cms_usernames.update({
        where: { id: userDetails.id },
        data: { name, updated_at: new Date() },
      });
    }
  } else {
    await this.prisma.cms_usernames.create({
      data: { user_id: userId, name, tenant_id: tenantId, created_at: new Date() },
    });
  }
}
```

The Prisma model ([backend/prisma/schema.prisma:546–555](../../../repos/case-management-system/backend/prisma/schema.prisma#L546-L555)):

```prisma
model cms_usernames {
  id         Int      @id @default(autoincrement())
  user_id    String   @unique @db.Uuid
  tenant_id  String
  name       String
  created_at DateTime @default(now()) @db.Timestamp(6)
  updated_at DateTime @updatedAt @db.Timestamp(6)

  @@map("cms_usernames")
}
```

The migration is applied ([backend/prisma/migrations/20260616124311_username/migration.sql](../../../repos/case-management-system/backend/prisma/migrations/20260616124311_username/migration.sql)):

```sql
CREATE TABLE "cms_usernames" (
    "id" SERIAL NOT NULL,
    "user_id" UUID NOT NULL,
    "tenant_id" TEXT NOT NULL,
    "name" TEXT NOT NULL,
    "created_at" TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "updated_at" TIMESTAMP(6) NOT NULL,
    CONSTRAINT "cms_usernames_pkey" PRIMARY KEY ("id")
);
CREATE UNIQUE INDEX "cms_usernames_user_id_key" ON "cms_usernames"("user_id");
```

`extractInnerToken` reads standard OIDC-shape claims (`sub`, `name`, `tenant_id`) — this code has no IAM-specific coupling and is compatible with the plug-and-play IAM contract.

### 2. The biar automation orchestrator has an ETL for `cms_usernames` (already on `dev`)

The registry mapping ([automation-orchestrator/lakehouse_automation_pipeline.py:21, 52–69](../../../repos/biar/automation-orchestrator/lakehouse_automation_pipeline.py#L21)):

```python
from Table_ETLs.cms_usernames import CmsUsernamesETL
# ...
_ETL_REGISTRY: dict[str, type] = {
    # ...
    "cms_usernames": CmsUsernamesETL,
}
# and dispatch at line 165–166:
if table in self._ETL_REGISTRY:
    self._ETL_REGISTRY[table](self.spark, self.warehouse_root).run(source_path)
```

The ETL itself is a pass-through that writes to `gold/cms_usernames` with Hudi record key = `id` ([automation-orchestrator/Table_ETLs/cms_usernames.py:15–65](../../../repos/biar/automation-orchestrator/Table_ETLs/cms_usernames.py#L15-L65)):

```python
class CmsUsernamesETL(BaseETL):
    TABLE_NAME = "cms_usernames"

    @property
    def gold_path(self) -> str:
        return f"{self.warehouse_root}/gold/{self.TABLE_NAME}"

    def run(self, source_path: str) -> str:
        raw = self.spark.read.json(source_path)
        df = (
            raw
            .withColumn("ingested_at_ts", F.current_timestamp())
            .withColumn("source_file_path", F.input_file_name())
            .withColumn("record_hash", F.sha2(F.concat_ws("||",
                *[F.coalesce(F.col(c).cast("string"), F.lit("")) for c in raw.columns]), 256))
        )
        self.write_hudi(df, self.gold_path,
            self.hudi_opts(table_name=self.TABLE_NAME,
                           record_key="id", precombine="created_at"))
        return self.gold_path
```

The ETL is wired, correct, and would happily write incoming batches to `gold/cms_usernames`. It receives nothing.

### 3. The notebook already knows how to render investigator names

Data load in cell 1 of [JupyterHub/notebooks/Case_Management_Trend_Dashboard.ipynb](../../../repos/biar/JupyterHub/notebooks/Case_Management_Trend_Dashboard.ipynb):

```python
cms_usernames_path = f"{WAREHOUSE_ROOT}/gold/cms_usernames"
cms_usernames = load_tenant_hudi(cms_usernames_path)
```

Investigator lookup in cell 14:

```python
if cms_usernames is not None:
    investigator_lookup = (
        cms_usernames
        .withColumn("_rn", F.row_number().over(
            Window.partitionBy("user_id").orderBy(F.desc("updated_at"))))
        .filter(F.col("_rn") == 1)
        .select(F.col("user_id"), F.col("name").alias("investigator_name"))
    )
    investigator_workload = investigator_workload.join(
        investigator_lookup,
        investigator_workload.case_owner_user_id == investigator_lookup.user_id,
        "left"
    ).drop("user_id")
```

The left-join means: if there is no matching `user_id` in `gold/cms_usernames`, the case row still comes through with `investigator_name = NULL`, and the presenter loop falls back to printing the raw UUID:

```python
if pd.notna(investigator_name) and str(investigator_name).strip():
    print(f"  {int(idx+1)}. {investigator_name}  (ID: {investigator_id})")
else:
    print(f"  {int(idx+1)}. Name not found in cms_usernames — showing ID: {investigator_id}")
```

The notebook is correct and defensive. It silently degrades to UUIDs when `gold/cms_usernames` is empty — matching the symptom Sandy reported on 2026-05-21.

### 4. NiFi has no processor for `cms_usernames`

Full-Stack-Docker-Tazama repo, [biar/nifi/tazama.xml](../../../repos/Full-Stack-Docker-Tazama/biar/nifi/tazama.xml) on `dev`:

```bash
$ grep -c 'cms_usernames' repos/Full-Stack-Docker-Tazama/biar/nifi/tazama.xml
0
```

Extracting every `<value>` immediately following a `<key>Table Name</key>` block yields:

```
account, account_holder, alerts, cases, comments, condition, entity,
evaluation, network_map, rule, tasks, transaction, transaction_data, typology
```

Every other `tazama_cms` table (`alerts`, `cases`, `tasks`, `comments`) has a `QueryDatabaseTableRecord` processor pointed at it. `cms_usernames` does not.

### 5. A fix has been prepared but not merged

- [Full-Stack-Docker-Tazama PR #236](https://github.com/tazama-lf/Full-Stack-Docker-Tazama/pull/236) — `feat(biar/nifi): update NiFi template for cms_usernames.` — **OPEN, not merged.**
- [biar PR #89](https://github.com/tazama-lf/biar/pull/89) — mirror of the same NiFi patch on the biar repo — **OPEN, not merged.**

Diff excerpt from PR #236:

```xml
<key>Table Name</key>
<value>cms_usernames</value>
```

added inside a new `QueryDatabaseTableRecord` block, alongside supporting `RemoteProcessGroup` `<key>table</key><value>cms_usernames</value>` entries — the same shape used by the existing `alerts`, `cases`, `tasks`, and `comments` processors.

---

## Root Cause Analysis

**Root cause:** the NiFi ingest template (`biar/nifi/tazama.xml`) never had a `QueryDatabaseTableRecord` processor added for `tazama_cms.cms_usernames`, so the automation orchestrator never receives batches for the `cms_usernames` table and the Hudi gold table stays empty.

Every downstream symptom follows from that:
- **`gold/cms_usernames` is empty** → the Spark reader in the notebook returns an empty DataFrame.
- **The notebook's left-join finds no matches** → every `case_owner_user_id` maps to `NULL` for `investigator_name`.
- **The presenter falls back to raw UUIDs** → users see "Name not found in cms_usernames — showing ID: {UUID}" instead of names.
- **All `Case_Management_Trend_Dashboard.ipynb` "Workload & Investigator Productivity" charts and top-N tables show UUIDs** → the visible symptom Sandy reported.

The CMS auth-service upsert, the Prisma migration, and the `CmsUsernamesETL` were all completed in June 2026. The NiFi step was skipped or lost — the June 18 comment claiming "The NiFi pipeline is ready and data is now being successfully ingested into Ozone" was ahead of the actual template update, which is still open as a PR.

---

## Blast Radius — Full File Inventory

### Files that must change (Track A — merge the prepared PRs)

| File | What is affected | Line numbers |
|---|---|---|
| `biar/nifi/tazama.xml` (in Full-Stack-Docker-Tazama) | Add a `QueryDatabaseTableRecord` processor + supporting connections for `tazama_cms.cms_usernames` | New XML blocks; see PR #236 diff |
| `nifi/tazama.xml` (in biar repo) | Same patch, mirror copy | See biar PR #89 diff |

Track A is a pure NiFi template edit. No source code changes are required in any repo.

### Files indirectly affected (no code change, but behavior depends on Track A landing)

| File | Behavior that changes when Track A lands | Line numbers |
|---|---|---|
| `biar/JupyterHub/notebooks/Case_Management_Trend_Dashboard.ipynb` | Cell 14's investigator-name join starts returning names; the "Name not found" fallback stops firing for logged-in users | Cell 1, cell 14 |
| `biar/automation-orchestrator/Table_ETLs/cms_usernames.py` | Starts receiving batches; today it is unreachable | L33–65 |
| Any future dashboard that reads `gold/cms_usernames` | Same — starts having data to read | N/A |

---

## Side-Effect Map

| Consequence if left unfixed | Downstream effect |
|---|---|
| `gold/cms_usernames` remains empty | Every consumer that joins on `user_id` sees NULL names |
| Case Management Trend Dashboard shows UUIDs | Supervisors cannot attribute case actions to human investigators without cross-referencing IAM manually |
| Any future dashboard (Executive Overview, Fraud Trend, TMS Performance) that plans to resolve investigator names will hit the same wall | Silent degradation across BIAR reports |
| The `CmsUsernamesETL` code and Prisma table sit unused | Dead-code perception; if anyone "cleans up" the ETL not realising why it exists, the eventual fix regresses |
| Every CMS login continues to upsert a row into `cms_usernames` that nothing reads | Table grows but stays orphan; low but real DB churn |

---

## Effort Assessment

### Track A — Merge NiFi processor patch

| Step | File | Effort |
|---|---|---|
| 1. Review Full-Stack-Docker-Tazama PR #236 | `biar/nifi/tazama.xml` | ~30 min (large XML diff; but existing patch is prepared) |
| 2. Review biar PR #89 (mirror) | `nifi/tazama.xml` | ~15 min |
| 3. Merge both PRs to `dev` | — | 5 min |
| 4. Deploy NiFi with the new template in UAT | Ops | ~30 min |
| 5. Trigger a `cms_usernames` login on UAT & confirm `gold/cms_usernames` populates | Ops + Sandy | ~15 min |
| 6. Re-run Case Management Trend Dashboard, confirm names appear | Sandy | ~10 min |

- Files: 2 XML templates
- Schema migration required: **No** (migration already applied on `dev`)
- Frontend changes required: **No**
- Backend code changes required: **No**
- Estimated total: **~2 hours** end-to-end, most of it review + deploy

### No Track B is proposed

An event-driven identity sync (e.g. a Keycloak SPI event listener writing into `cms_usernames`) would eliminate the login-driven staleness described in the Known Limitations section below. It is not proposed here because CMS must remain IAM-agnostic — the CMS repository is required to be a plug-and-play IAM consumer with no IAM-vendor-specific code. Any solution to the staleness gap must live **outside** the CMS repo (in the deployment/ops layer, or in NiFi tapping a standardized IAM export), not inside it. Since no such external component exists today and the immediate #225 report is fully resolved by Track A, deferring the staleness fix is the right call.

### Recommended sequencing

1. **Ship Track A immediately.** It is the specific fix for #225 as reported. Both PRs are already open and reviewed.
2. **Document the known limitations** (below) so operators and reporting stakeholders know to expect stale-name / missing-name cases for users who never log in.

---

## Known Limitations — Login-Driven Staleness

Track A fixes the reported symptom for the common case (users who have logged into CMS at least once). It leaves three residual gaps, all rooted in the fact that `cms_usernames` is populated only at CMS-login time:

| Limitation | Effect on the dashboard |
|---|---|
| User exists in IAM but has never logged into CMS | Their `user_id` never appears in `cms_usernames`, so the dashboard falls back to UUID for cases they own |
| User's display name is changed in IAM but they don't subsequently log in | Their `cms_usernames` row keeps the old name; the dashboard shows the stale name |
| User is disabled in IAM | `cms_usernames` has no mechanism to mark them disabled — they still appear in reports |

**Why not fix in CMS:** the CMS repo is a plug-and-play IAM consumer. Any code that reaches into the IAM (SPI event listeners, admin-API polling, disabled-user detection) would introduce vendor-specific coupling. That work belongs in the deployment/ops layer or in NiFi tapping a standardized IAM export — not in CMS.

**Operational workaround:** ask supervisors to have each active investigator log into CMS at least once after deployment. This backfills the common case.

---

## Acceptance Criteria

### Track A

- [ ] Full-Stack-Docker-Tazama PR #236 merged to `dev`.
- [ ] biar PR #89 merged to `dev`.
- [ ] After redeploying NiFi in UAT and performing at least one CMS login as a known user, a `SELECT COUNT(*) FROM tazama_cms.cms_usernames` returns ≥ 1, and the corresponding Hudi table at `gold/cms_usernames` contains ≥ 1 row.
- [ ] `Case_Management_Trend_Dashboard.ipynb` re-run against UAT displays the investigator's display name (not UUID) in the "TOP 5 INVESTIGATORS BY WORKLOAD" section for at least one investigator.
- [ ] The "Name not found in cms_usernames — showing ID: …" fallback line does not appear for any investigator who has logged into CMS at least once since deployment.
- [ ] Known-limitation cases (never-logged-in / renamed-but-not-logged-in users) are documented in a UAT release note so stakeholders know to expect them.

---
