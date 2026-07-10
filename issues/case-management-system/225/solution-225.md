# Solution — Issue #225: BIAR Bug: Case Management Trend Dashboard — no investigator data displayed

## Complete Fix — Exact Code Changes

Track A can merge independently — it is a NiFi template patch across two repos with prepared PRs already open. Track B must land as a coordinated release across CMS backend + biar ETL + a new database migration and cannot be split further, but Track B is a follow-up architectural fix, not a ship-blocker for #225.

---

### Track A — Merge NiFi processor patch (PR 1 of 2 — already open)

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

#### Impact of Track A alone

- `gold/cms_usernames` populates after NiFi's next poll.
- The Case Management Trend Dashboard "TOP 5 INVESTIGATORS BY WORKLOAD" table shows names for users who have logged in since deployment.
- The notebook's fallback line ("Name not found in cms_usernames — showing ID…") stops firing for logged-in users.
- Track A does not solve name freshness for users who don't log in, and does not solve the disabled-user case.

---

### Track B — Event-driven Keycloak identity sync (PR 2 of 2 — proposed)

Track B replaces the login-time upsert with a Keycloak-event-driven sync so `cms_usernames` reflects Keycloak state without depending on users logging in. It also adds columns to track sync freshness. **Track B must land as a single release: migration + backend + backfill + auth.service change.**

#### Step B1 — Schema migration

**File:** `backend/prisma/migrations/{new-timestamp}_cms_usernames_sync_fields/migration.sql` (new)

```sql
ALTER TABLE "cms_usernames"
    ADD COLUMN "last_synced_at" TIMESTAMP(6),
    ADD COLUMN "disabled_at"    TIMESTAMP(6),
    ADD COLUMN "source"         TEXT NOT NULL DEFAULT 'keycloak-event';

CREATE INDEX "cms_usernames_disabled_at_idx" ON "cms_usernames"("disabled_at");
```

**Data migration** (optional; safe to skip if the backfill script covers it):

```sql
-- Mark all pre-existing rows as coming from the legacy login-driven path
UPDATE "cms_usernames"
SET "source" = 'login-legacy'
WHERE "source" = 'keycloak-event' AND "created_at" < '2026-07-15';  -- or Track B deploy date
```

**Caveats:** additive columns only. `last_synced_at` starts NULL and is set by the sync service. Safe under concurrent writes from the still-running login-time upsert.

#### Step B2 — Prisma schema update

**File:** [backend/prisma/schema.prisma:546-555](../../../repos/case-management-system/backend/prisma/schema.prisma#L546-L555)

**BEFORE**

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

**AFTER**

```prisma
model cms_usernames {
  id              Int       @id @default(autoincrement())
  user_id         String    @unique @db.Uuid
  tenant_id       String
  name            String
  created_at      DateTime  @default(now()) @db.Timestamp(6)
  updated_at      DateTime  @updatedAt @db.Timestamp(6)
  last_synced_at  DateTime? @db.Timestamp(6)
  disabled_at     DateTime? @db.Timestamp(6)
  source          String    @default("keycloak-event")

  @@index([disabled_at])
  @@map("cms_usernames")
}
```

#### Step B3 — Add `KeycloakSyncService`

**File:** `backend/src/modules/identity-sync/keycloak-sync.service.ts` (new)

Skeleton — the exact transport (SPI callback, webhook, or Kafka consumer) depends on the Keycloak deployment. Recommended: Keycloak SPI event listener → outbound HTTP webhook → this controller.

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';

interface KeycloakUserEvent {
  eventType: 'USER_CREATE' | 'USER_UPDATE' | 'USER_DELETE' | 'USER_DISABLE';
  userId: string;         // Keycloak user UUID
  tenantId: string;
  name: string | null;
  disabled?: boolean;
}

@Injectable()
export class KeycloakSyncService {
  private readonly logger = new Logger(KeycloakSyncService.name);

  constructor(private prisma: PrismaService) {}

  async handleEvent(evt: KeycloakUserEvent): Promise<void> {
    const now = new Date();

    if (evt.eventType === 'USER_DELETE' || evt.eventType === 'USER_DISABLE') {
      await this.prisma.cms_usernames.updateMany({
        where: { user_id: evt.userId },
        data: { disabled_at: now, last_synced_at: now },
      });
      return;
    }

    await this.prisma.cms_usernames.upsert({
      where: { user_id: evt.userId },
      update: {
        name: evt.name ?? undefined,
        tenant_id: evt.tenantId,
        last_synced_at: now,
        disabled_at: null,
      },
      create: {
        user_id: evt.userId,
        tenant_id: evt.tenantId,
        name: evt.name ?? '',
        source: 'keycloak-event',
        last_synced_at: now,
      },
    });
  }
}
```

#### Step B4 — Remove login-time upsert

**File:** [backend/src/modules/auth/auth.service.ts:60-68](../../../repos/case-management-system/backend/src/modules/auth/auth.service.ts#L60-L68)

**BEFORE**

```ts
this.logger.log('Login successful');

//To store UserName in DB
this.storeUserName(token).catch((error: unknown) => {
  if (error instanceof Error) {
    this.logger.error('Error storing user name:', error);
  }
});
```

**AFTER**

```ts
this.logger.log('Login successful');
// User name is synced from Keycloak events by KeycloakSyncService — no login-time upsert needed.
```

Then delete `storeUserName` at [auth.service.ts:267-301](../../../repos/case-management-system/backend/src/modules/auth/auth.service.ts#L267-L301) entirely.

**Ordering caveat:** deploy B1–B3 first and confirm the event feed is running before deleting the login-time upsert, otherwise you race a window where neither path writes.

#### Step B5 — Backfill script

**File:** `backend/scripts/backfill-cms-usernames.ts` (new)

Idempotent script that iterates Keycloak's Admin API user list, calls `KeycloakSyncService.handleEvent({ eventType: 'USER_CREATE', … })` for each user, and marks its own `source='backfill'`. Run once per environment after Track B deploys.

#### Step B6 — (optional) Switch Hudi record key

**File:** [automation-orchestrator/Table_ETLs/cms_usernames.py:54-62](../../../repos/biar/automation-orchestrator/Table_ETLs/cms_usernames.py#L54-L62)

**BEFORE**

```python
self.write_hudi(
    df,
    self.gold_path,
    self.hudi_opts(
        table_name=self.TABLE_NAME,
        record_key="id",
        precombine="created_at",
    ),
)
```

**AFTER**

```python
self.write_hudi(
    df,
    self.gold_path,
    self.hudi_opts(
        table_name=self.TABLE_NAME,
        record_key="user_id",       # dedupe by Keycloak identity, not surrogate SERIAL
        precombine="updated_at",    # keep the most recently updated row
    ),
)
```

**Caveat:** switching the Hudi record key requires a full reload of `gold/cms_usernames`. If preferred, defer this change — the notebook already de-duplicates via `row_number() over (partition by user_id order by updated_at desc)` in cell 14.

#### Step B7 — NiFi maximum-value column adjustment

**File:** `biar/nifi/tazama.xml` (after Track A has landed)

If Track B adopts `disabled_at` as a sync signal, update the `cms_usernames` processor's `Maximum-value Columns` from `updated_at` to `updated_at,disabled_at` so disabled events are picked up on the next NiFi poll.

---

## Test Cases

### Unit tests

**File:** `backend/src/modules/identity-sync/keycloak-sync.service.spec.ts` (new — for Track B)

```typescript
describe('KeycloakSyncService', () => {
  it('creates a new cms_usernames row on USER_CREATE', async () => {
    await service.handleEvent({
      eventType: 'USER_CREATE',
      userId: '11111111-1111-1111-1111-111111111111',
      tenantId: 'TAZAMA',
      name: 'Jane Doe',
    });
    const row = await prisma.cms_usernames.findUnique({
      where: { user_id: '11111111-1111-1111-1111-111111111111' },
    });
    expect(row.name).toBe('Jane Doe');
    expect(row.last_synced_at).not.toBeNull();
    expect(row.disabled_at).toBeNull();
    expect(row.source).toBe('keycloak-event');
  });

  it('updates name on USER_UPDATE when it differs', async () => {
    // seed then update, assert new name and updated_at moves forward
  });

  it('sets disabled_at on USER_DISABLE', async () => {
    // assert disabled_at is populated and last_synced_at is refreshed
  });

  it('is idempotent under duplicate events', async () => {
    // handle same USER_UPDATE twice, assert row_count stays 1
  });
});
```

### Integration / E2E scenarios (UAT — for Track A verification)

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
6. **Run the notebook:** open `Case_Management_Trend_Dashboard.ipynb` in JupyterHub against UAT, execute all cells for the "Last 90 days" period. In section 7 ("Workload & Investigator Productivity"), confirm names appear in the TOP 5 list; no "Name not found in cms_usernames" fallback lines.

### Data migration validation SQL (Track B)

```sql
-- After Track B migration
SELECT column_name, data_type, is_nullable, column_default
FROM information_schema.columns
WHERE table_schema = 'tazama_cms' AND table_name = 'cms_usernames'
ORDER BY ordinal_position;
-- Expect: id, user_id, tenant_id, name, created_at, updated_at,
--         last_synced_at, disabled_at, source

-- After backfill
SELECT source, COUNT(*) FROM tazama_cms.cms_usernames GROUP BY source;
-- Expect at least one 'backfill' or 'keycloak-event' row per Keycloak user
```

### Manual / UAT checks

| # | Scenario | Steps | Expected Result |
|---|---|---|---|
| 1 | Freshly deployed Track A, single logged-in investigator | Log in as investigator X, wait one NiFi poll interval, refresh notebook | Investigator X's name (not UUID) appears in TOP 5 by workload |
| 2 | Multiple investigators with cases, only some logged in | Log in as X, do not log in as Y (both own cases in DLH) | X shows by name; Y shows the "Name not found — showing ID: {UUID}" fallback |
| 3 | Investigator renamed in Keycloak, then logs in | Rename in Keycloak → log in → wait one NiFi poll → refresh notebook | Notebook shows new name (auth.service upsert path updates row) |
| 4 | Rename in Keycloak, no login (Track A only) | Rename in Keycloak, do NOT log in, wait, refresh notebook | Notebook shows the *old* name — this is the Track A limitation |
| 5 | Rename in Keycloak, no login (after Track B) | Same, but with `KeycloakSyncService` deployed | Notebook shows new name within one sync interval — no login required |
| 6 | Disabled user (after Track B) | Disable user in Keycloak | `cms_usernames.disabled_at` populated; downstream queries can filter |

---

## Overall Impact of the Fix

| Area | Before | After (Track A) | After (Track B) |
|---|---|---|---|
| `gold/cms_usernames` | Empty | Populated for logged-in users; lags on renames without login | Populated and fresh for all Keycloak users |
| Case Management Trend Dashboard "TOP 5 INVESTIGATORS" | Shows UUIDs and "Name not found" fallback | Shows names for logged-in users; UUIDs for others | Shows names for all Keycloak users |
| Investigator name freshness | N/A (empty) | On-login only | On-Keycloak-event (near-real-time) |
| Disabled users in DLH | Not surfaced | Not surfaced | Marked via `disabled_at` |
| `CmsUsernamesETL` code | Dead-code (never invoked) | Active | Active |
| Architectural stance | Login-driven identity mirror in CMS DB | Same (unchanged from before) | Event-driven identity mirror; ready to promote to a shared identity service later |

---

## Fix Summary

The Case Management Trend Dashboard shows no investigator names because the Hudi gold table it joins against (`gold/cms_usernames`) is empty. Every other piece of the plumbing is in place on `dev` — CMS writes a row to a Postgres `cms_usernames` table on every successful login (already merged, June 2026), a Spark ETL is registered to consume `cms_usernames` batches and write them to Hudi (already merged), and the notebook already knows how to left-join on `user_id` and fall back gracefully to UUIDs when the name is missing. The one missing link is a single `QueryDatabaseTableRecord` processor in the NiFi ingest template — the piece that would pull rows out of Postgres and hand them to the automation orchestrator.

Track A is: merge the two open PRs that add that NiFi processor (Full-Stack-Docker-Tazama #236 and biar #89), redeploy NiFi, and confirm names appear. This is a ~2-hour operation and fixes the issue as literally reported by Sandy. It is safe to ship in isolation and has no overlap with the open issues #214 or #220. This is the recommended immediate action.

Track B is the durable fix: replace the "we upsert the name on every login" pattern with a Keycloak-event-driven sync. This closes the two gaps Track A leaves behind — users who never log in are absent from the DLH, and users whose Keycloak name changes without a subsequent login stay stale. It requires a small additive schema migration (three new columns to track sync freshness and disabled state), a new `KeycloakSyncService` in the CMS backend, a one-time backfill script, removal of the login-time upsert, and an optional Hudi record-key change flagged as a minor design note in biar #112. Estimated effort is about six developer-days, and it depends on nothing else — but it is not a #225 blocker.

For a non-technical reader: the dashboard was blind because one XML file was missing one entry. That entry is written and waiting in a PR. Once merged, names will start appearing for anyone who has logged in. Making names appear for users who *haven't* logged in requires wiring CMS to Keycloak's event stream — a separate, follow-on piece of work.
