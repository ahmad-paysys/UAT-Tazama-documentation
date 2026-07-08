# Impact Study — Issue #108: transaction_data Naming Collision

**Issue:** [#108](https://github.com/tazama-lf/biar/issues/108)
**Study Date:** 2026-07-08
**Author:** Ahmad Khalid
**Repository:** tazama-lf/biar

---

## Summary

The name `transaction_data` / `transactionData` appears in five unrelated places across the Lakehouse and CMS code. The root cause is that `TransactionsETL` sources from the wrong upstream (pacs bronze rather than `event_history.transaction`), and its Hudi column `transactionData` name misrepresents what it holds. The naming problem is a symptom of that wrong source. **Track A** is a hardening acceptance criterion folded into the #110 view-builder rewrite (no standalone code change). **Track B** is #110 itself — once that rewrite lands, all five collisions dissolve as a natural consequence.

---

## Confirmed Root Cause

- `bronze/transactions.transactionData` holds raw ISO 20022 pacs.008/002 payload, not TMS-normalised transaction data. Confirmed at `TransactionsETL.py:72-102` and `line 128`.
- Both `network_navigator_view.py:68-81` and `transaction_detail_view.py:53-69` (and `transaction_history_view.py`) resolve the payload column through a candidate detection list `["transactionData", "transaction_data", "transaction", "payload", …]` and alias it as `transaction_data`. The `"transaction"` fallback is the load-bearing latent risk for #110.
- NiFi processor `Transaction_Data` (committed at `biar/nifi/tazama.xml:5905`, table `transaction_data` at `line 26173`, pool `tazama_cms` at `lines 4779-4784`) is a *correctly*-named processor targeting the CMS ODS. It happens to collide with the mis-token used inside view builders.
- **Live NiFi (2026-07-08 at `10.10.80.19:8088`) does not currently contain a running `Transaction_Data` processor.** The committed repo XML has it; the deployed flow does not. The collision is a repo-hygiene concern; it is not causing any live incident *from the naming itself* today.
- `tazama_cms.transaction_data` is defined by CMS Prisma model `TransactionData` (`case-management-system/backend/prisma/schema.prisma` on dev) with a nested `transactionData Json` column and PK `transactionId Int @default(autoincrement())`. Confirmed against dev.

---

## Track A — Hardening AC on #110 (recommended)

### What Changes

No standalone code changes for #108. Instead, add hardening acceptance criteria to the #110 rewrite: tighten view-builder candidate-column resolution to fail loudly on unexpected columns. See [issues/biar/110/solution-110.md](../110/solution-110.md).

### Impact

| Property | Value |
|---|---|
| Files changed | 0 (this issue) — 3 view builders in #110 |
| Schema migration required | No |
| Frontend changes required | No |
| Downtime required | No |
| Risk of regression | Low (fail-fast on schema mismatch is safer than silent alias) |
| Reversibility | Trivial |

What this fixes immediately:
- Nothing standalone; the hardening AC only takes effect when #110 ships.

What this does not fix:
- Everything until #110 lands. Developer confusion, doc drift, latent risk during migration.

Track A is safe to ship in isolation (it is a documentation-only acceptance criterion).

---

## Track B — Delete the wrong-source pipeline (via #110)

### What Changes

Track B is #110 in its entirety. Rewriting `TransactionsETL` to source from `event_history.transaction`, deleting `CombinedPacsETL`, and rebuilding `bronze/transaction`, `silver/transaction`, `gold/transaction` as singular tables. The `transactionData` column disappears entirely — not renamed, replaced.

### Schema Impact

| Change | Note |
|---|---|
| Drop `bronze/transactions`, `silver/transactions`, `gold/transactions` Hudi tables | Rebuild as `bronze/transaction` (singular) etc. under #110 |

Migration notes: pipeline reset (empty existing plural tables; NiFi replays event_history.transaction via its own watermark).

### Backend Code Impact

| Area | Files |
|---|---|
| Core rewrite | `TransactionsETL.py`, `lakehouse_automation_pipeline.py`, `CombinedPacs.py` (deletion) |
| View builders (candidate list hardening — the #108-specific hook) | `network_navigator_view.py`, `transaction_detail_view.py`, `transaction_history_view.py` |
| Downstream (via #111) | See [issues/biar/111/impact-111.md](../111/impact-111.md) |

### Frontend Code Impact

None from #108 directly. CMS-side changes (from #111) do not concern the naming issue.

---

## Side Effects and Risks

### Risks of Track A Alone

| Risk | Likelihood | Mitigation |
|---|---|---|
| Track A hardening AC forgotten during #110 landing | Medium (easy to lose in a big PR) | Include as explicit acceptance criterion on the #110 PR body |

### Risks of Track B

See [issues/biar/110/impact-110.md](../110/impact-110.md). The #108-specific risk within Track B is the candidate-list trap: if the view builders retain the `"transaction"` fallback, and the new `bronze/transaction` bronze table's payload column ends up called `transaction` (the natural name given the ODS column), consumers silently return all-nulls with no error.

### Cross-Issue Dependencies

- **#110**: This issue closes as a consequence of #110. #108 has no independent code fix.
- **#109**: The NiFi `Transaction_Data` processor discussed in #108 is not currently deployed (see live NiFi finding); #109 addresses its remediation should it be enabled.
- **#111**: The view-builder candidate list hardening called out here is executed in the #110/#111 rewrite scope.

---

## Effort Estimate

| Track | Files | Effort |
|---|---|---|
| A — Hardening AC | Issue tracker only | ~15 min |
| B — Subsumed by #110 | See #110 | See #110 |

---

## Acceptance Criteria (Verification Checklist)

### Track A

- [ ] `#110` PR body lists: "candidate lists in view builders raise on unknown columns; `\"transaction\"` fallback removed."
- [ ] `data-lineage.md` updated in #110 to reflect that `transactionData` is no longer a bronze column.

### Track B

- [ ] All five collision vectors verified as resolved after #110 lands (verify by grep of `transaction_data` / `transactionData` — only the CMS Prisma model remains, and that is unambiguously CMS-scoped).

---

## Recommended Sequencing

1. Do not ship a standalone fix for #108.
2. Add the two Track A ACs to the #110 PR body.
3. Verify closure on merge of #110.
4. Close #108 with a link to the #110 merge commit.
