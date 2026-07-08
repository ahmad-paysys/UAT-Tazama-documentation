# Issue #108 — DLH BUG: transaction_data Naming Collision

**Repository:** tazama-lf/biar
**Issue:** [DLH BUG: transaction_data Naming Collision](https://github.com/tazama-lf/biar/issues/108)
**Author:** Sandy-at-Tazama
**State:** Open
**Report Date:** 2026-07-08
**Verified against:** biar `ef0c856` (dev), frms-coe-lib `513d91d` (dev), case-management-system `2b14c12f` (dev), Full-Stack-Docker-Tazama `e218211` (dev), live NiFi at `http://10.10.80.19:8088`.

---

## Executive Summary

The identifier `transaction_data` (and its camelCase variant `transactionData`) appears in five unrelated places across the Tazama Lakehouse and CMS surfaces, each referring to something different. The origin is a single misname — the Hudi column `bronze/transactions.transactionData` holds a raw pacs.008/002 ISO 20022 payload but is named as if it holds TMS-normalised `event_history.transaction` data. That misname then spreads outward through a view-builder alias so that the token `transaction_data` appears throughout the view code, colliding with an unrelated NiFi processor of the same name, an unrelated CMS ODS table (`tazama_cms.transaction_data`), and an unrelated CMS column inside that table (also named `transactionData`).

The naming problem is a symptom, not the cause. The root cause is that `TransactionsETL` is sourced from the wrong upstream (pacs bronze, not `event_history.transaction`). Once issue #110 rewrites `TransactionsETL` to source from `event_history.transaction`, the `transactionData` column disappears entirely along with the pacs-fed pipeline that produces it — and all five collisions collapse as a natural consequence. No standalone code change is required or recommended for #108. The one live risk that must be handled during the #110 rewrite is the view-builder candidate detection list: it includes `"transaction"` as a fallback and will silently pick up the new payload column if that column happens to be named `transaction`, producing all-null rows with no error.

---

## How It Works Today — Confirmed in Code

### 1. `bronze/transactions.transactionData` — pacs payload written by `TransactionsETL`

[`TransactionsETL.py:72-102`](../../../repos/biar/automation-orchestrator/Table_ETLs/TransactionsETL.py) — `_transactions_from_pacs` reads pacs bronze, aliases the coalesced `document_json`/`document` payload as `transactionData`, and this alias is persisted to the Hudi bronze table at [line 128](../../../repos/biar/automation-orchestrator/Table_ETLs/TransactionsETL.py) (`.withColumn("transactionData", F.col("transactionData").cast("string"))`).

The column literally holds an ISO 20022 pacs.008 or pacs.002 message body. The name implies TMS-normalised transaction data. The two are unrelated.

### 2. View builder alias — the `transaction_data` variable inside view code

Both `network_navigator_view.py` and `transaction_detail_view.py` (and `transaction_history_view.py`) resolve the payload column via a hardcoded candidate list, then re-alias it as `transaction_data`.

[`network_navigator_view.py:68-81`](../../../repos/biar/automation-orchestrator/Table_ETLs/network_navigator_view.py):

```python
candidates = ["transactionData", "transaction_data", "transaction", "payload", "raw_payload"]
json_col = next((c for c in candidates if c in tx.columns), None)
...
tx = tx.withColumn("transaction_data", F.col(json_col).cast("string"))
```

[`transaction_detail_view.py:53-69`](../../../repos/biar/automation-orchestrator/Table_ETLs/transaction_detail_view.py):

```python
candidates = ["transactionData", "transaction_data", "transaction", "payload", "raw_payload", "raw_json", "transaction_json"]
```

Every subsequent `F.get_json_object("transaction_data", "$.…")` inside these view builders refers to the pacs payload from `bronze/transactions.transactionData`, **not** to the CMS ODS table of the same name.

### 3. NiFi `Transaction_Data` processor — reads CMS `tazama_cms.transaction_data`

Present in the committed [`biar/nifi/tazama.xml`](../../../repos/biar/nifi/tazama.xml) at:
- Label `Transaction_Data` — line 5905
- `Table Name = transaction_data` — line 26173
- Pool `b0b69311-…` — line 26165, which resolves to `name = tazama_cms` and URL `jdbc:postgresql://10.10.80.18:15433/tazama_cms` at lines 4779–4784.

**Live NiFi cross-check (2026-07-08, `http://10.10.80.19:8088`):** the deployed flow does **not** contain a running `Transaction_Data` processor. Distinct `Table Name` values observed on RUNNING processors: account, account_holder, alerts, cases, cms_usernames, comments, condition, entity, evaluation, network_map, pacs002, pacs008, pain001, pain013, rule, tasks, transaction, typology. `transaction_data` is absent from the live flow.

The naming collision documented by this issue is therefore purely a repo-level and design-level concern; there is no live incident *from this collision* on the deployed flow at snapshot time. Committed flow contains it — deployed flow does not.

### 4. `tazama_cms.transaction_data` — the CMS ODS table

Defined by the CMS Prisma schema on dev at `case-management-system/backend/prisma/schema.prisma`:

```prisma
model TransactionData {
  transactionId   Int      @id @default(autoincrement())
  tenantId        String
  endToEndId      String   @db.VarChar(50)
  transactionData Json
  createdAt       DateTime @default(now()) @db.Timestamp(6)

  @@map("transaction_data")
}
```

The `@@map("transaction_data")` confirms the SQL table name. Column `transactionData` (nested-collision #5) is a `Json` payload column. No Lakehouse code references this table as a source.

### 5. Guiding principle already recognised

Sandy's own proposed guiding principle (in the issue body) is that the four core TMS tables — `event_history.transaction`, `event_history.entity`, `event_history.account`, `event_history.account_holder` — are the definitive source of truth for BIAR analytics; raw pacs message tables should only be used when info is genuinely absent from those. The naming collision is a downstream symptom of a wrong-source decision, not a naming defect in isolation.

---

## Root Cause Analysis

`TransactionsETL` sources from the wrong upstream (pacs bronze rather than `event_history.transaction` ODS). That single wrong source produces a Hudi column called `transactionData` whose name misrepresents its contents; the view-builder alias then propagates the mis-token throughout the code base, where it happens to collide with correctly-named CMS objects. Every downstream symptom (developer confusion, latent silent-null-parse risk in the candidate list, apparent-but-fake CMS coverage) traces back to this one wrong source. Fixing the source (issue #110) dissolves the collisions; renaming the column in isolation would be throwaway work that gets deleted when #110 lands.

---

## Blast Radius — Full File Inventory

### Files with the misnamed column or alias

| File | What's affected | Lines |
|---|---|---|
| `automation-orchestrator/Table_ETLs/TransactionsETL.py` | Creates `transactionData` column from pacs payload | 72–102, 128 |
| `automation-orchestrator/Table_ETLs/network_navigator_view.py` | Candidate list + `transaction_data` alias | 68–81, 84–102 |
| `automation-orchestrator/Table_ETLs/transaction_detail_view.py` | Candidate list + `transaction_data` alias | 53–69, 74–125 |
| `automation-orchestrator/Table_ETLs/transaction_history_view.py` | Candidate list + `transaction_data` alias | 55–66 and beyond |

### Files with the correctly-named CMS objects that collide with the alias

| File | What's affected | Lines |
|---|---|---|
| `nifi/tazama.xml` (committed) | Processor `Transaction_Data` reads `tazama_cms.transaction_data` | 5905, 26173, 26165 (pool) |
| `case-management-system/backend/prisma/schema.prisma` (dev) | CMS `TransactionData` model → table `transaction_data` with column `transactionData` | model block |

### Files indirectly affected

| File | Why |
|---|---|
| `automation-orchestrator/Table_ETLs/alert_history_view.py` | Consumes `vw_transaction_detail` — inherits the alias context |
| `data-lineage.md` | Documents `bronze/transactions.transactionData` as if the naming is correct |

---

## Side-Effect Map

| Consequence if left unfixed | Impact |
|---|---|
| Developer follows the `transaction_data` token in view builder code, expects to find CMS ODS data or `event_history.transaction` data, wastes time | Chronic slow tax on any transaction-data feature work |
| A "surface `tazama_cms.transaction_data` in a dashboard" requirement lands and gets implemented against `bronze/transactions.transactionData` — silently ships pacs payload instead | Feature ships wrong; caught late |
| During #110 rewrite, if the new `bronze/transaction` payload column happens to be named `transaction`, the existing view builders' candidate list picks it up as a fallback and parses TMS-normalised JSON as if it were ISO 20022. All fields become null; no error. | Silent all-null output post-#110 |
| Anyone reading `nifi/tazama.xml` + view builder code together concludes the NiFi processor feeds the views (it doesn't) | Wrong mental model, wrong debugging path |

---

## Effort Assessment

### Track A — No standalone code change (recommended)

`transactionData` is a throwaway column. #110 replaces the entire pipeline that produces it. Standalone renaming means changing the ETL, all three view builders' candidate lists, docs, and any snapshot Hudi state — all of which is discarded when #110 lands.

**Only relevant Track A action** is a hardening acceptance criterion that must be folded into the #110/#111 view-builder rewrite: eliminate the `"transaction"` fallback in the candidate list and require an explicit named-column resolution or a schema check that raises on mismatch.

| Step | File | Effort |
|---|---|---|
| Add hardening AC to #110/#111 PR | (issue tracker only) | ~15 min |

Schema migration: no. Frontend changes: no.

### Track B — Full remediation (subsumed by #110)

Track B is #110 itself. See [issues/biar/110](../110/). All five collision vectors dissolve as a natural consequence of that rewrite.

### Recommended sequencing

1. Do **not** ship a standalone rename for #108.
2. When #110 lands, ensure its acceptance criteria include: (a) view builders' candidate list is tightened to fail loudly on unknown columns, (b) `data-lineage.md` refreshed.
3. Close #108 as "resolved by #110".

---

## Acceptance Criteria

### Track A (hardening)

- [ ] `#110` PR removes the `"transaction"` fallback from the candidate lists in `transaction_detail_view.py`, `transaction_history_view.py`, and `network_navigator_view.py`.
- [ ] Candidate resolution raises a clear error on unknown/unexpected columns rather than silently aliasing.
- [ ] `data-lineage.md` no longer describes `transactionData` as a pacs payload column on `bronze/transactions`.

### Track B (via #110)

See [issues/biar/110/issue-110.md](../110/issue-110.md) — this issue closes when #110 closes.

---
