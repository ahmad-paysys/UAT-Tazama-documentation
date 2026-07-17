# CMS #220 — Sandy's Feedback Analysis (comment `#issuecomment-4972261159`)

**Repository:** [tazama-lf/case-management-system](https://github.com/tazama-lf/case-management-system)
**Issue:** [#220 — CMS SLA management - priority and SLA model](https://github.com/tazama-lf/case-management-system/issues/220)
**Comment under analysis:** [issuecomment-4972261159](https://github.com/tazama-lf/case-management-system/issues/220#issuecomment-4972261159) — posted by **Sandy-at-Tazama**
**Prepared:** 2026-07-16 (verified against `origin/dev` @ `fa31dec6`)

---

## 0. Executive summary

Sandy claims to have found **two related defects** in the **manual triage** path (`TriageService.handleManualTriage`) that break the invariant established in issue #220: that `priority` is a **derivative** of `priorityScore`, stamped once at triage from an auditable stored score.

- **Defect 1 (provenance / audit break).** The bucket is computed from a `?? 0.33`-defaulted local, but the persisted `priority_score` column is the raw DTO value (still possibly `undefined`), so Prisma leaves the column untouched. Because `createAlert` never sets `priority_score` at ingestion, the column starts and remains `NULL`, while `priority` gets stamped from `0.33`. Result: an alert with a stored severity that has no stored input — the audit chain the issue was built to enforce is broken on the very path meant to enforce it.
- **Defect 2 (semantic fossil / default is wrong bucket).** The literal `0.33` default was chosen under the **old** ladder to mean "URGENT" (the first rung above `NEW`, since the old ladder used `>= 0.33 → URGENT`). Under the **new** 3-bucket mapper on `dev`, `0.33 < 0.4` (`FALLBACK_MEDIUM_THRESHOLD`) so `0.33` now buckets to `LOW` — the *minimum* severity, which per §5 of the issue is given the *longest* SLA budget (`168h`). The default's meaning silently inverted from "elevated urgency" to "slackest deadline", and nothing records this as deliberate.

Both defects trace to the same underlying error: manual triage is the sole call site that decouples the value used to bucket from the value it persists. Every other bucket-and-persist site (`case-creation.service.ts`, `case-creation-approval.service.ts`, and the AI triage path in the same file) uses **one** variable for both — enforcing the invariant by construction. Sandy's read of the code is accurate on every point I could check against `dev`.

Below is a claim-by-claim verification, plus the context needed to understand why she considers this important, and her proposed fixes.

---

## 1. What the issue #220 model actually requires

Sandy's comment is grounded in the model laid out by the issue body. The load-bearing rules for her critique are:

- **§3 Priority is severity, not urgency.** `LOW / MEDIUM / HIGH`, stamped **once at triage** from `priorityScore`.
- **§8 Priority is the auditable source.** Changes go through an explicit, audited actuator; the SLA recalc job may never write `priority` (this is called "removing the original sin").
- **§10 Settled model.** *"`priority = LOW / MEDIUM / HIGH`, stamped once at triage from `priorityScore`, never written by the SLA job."*
- **§5 Severity calibrates the SLA clock.** Priority is not decorative; it feeds `target_seconds` on `SlaPolicy` (`HIGH` short, `LOW` long — MVP defaults `24h / 72h / 168h`).
- **§12 Deferred (but designed-in).** Severity-bucket thresholds could later become tenant-configurable — which requires the stored `priority_score` to be *auditable and complete*, so any bucket cut-point change can be re-derived from stored inputs.

The essence: **`priorityScore` is the source-of-truth input; `priority` is its output.** They must **agree** on every row, and both must be **persisted**, or the model collapses into an unauditable severity value.

---

## 2. Defect 1 — bucket derived from a value that is never persisted

### 2.1 The exact code Sandy cites

[`backend/src/modules/triage/triage.service.ts`](repos/case-management-system/backend/src/modules/triage/triage.service.ts) — `handleManualTriage`, L303–336 on `dev` (**verified**):

```typescript
// L303-305
const priorityScore = updateAlertDto.priorityScore ?? 0.33;                       // defaulted local
const priority = await this.casePriorityUtil.determinePriority(priorityScore, tenantId); // bucket from the DEFAULTED value
updateAlertData.priority = priority;
...
// L326-336
const alert = await this.alertService.updateAlert(
  alertId,
  userId,
  {
    confidencePer: updateAlertData.confidence_per,
    priority_score: updateAlertData.priorityScore,                                // the RAW DTO value, not the defaulted local
    ...JSON.parse(JSON.stringify(updateAlertData)),
  },
  tx,
  userName,
);
```

Two distinct variables are being sourced here:

| Var                                                    | Value when caller omits`priorityScore`                | Used for                                      |
| ------------------------------------------------------ | ------------------------------------------------------- | --------------------------------------------- |
| `priorityScore` (local, L303)                        | `0.33` (the `??` default kicks in)                  | Compute`priority` bucket (L304)             |
| `updateAlertData.priorityScore` (DTO property, L331) | `undefined` (`@IsOptional()` field with no default) | Persist as`priority_score` on the alert row |

Sandy is right that these are **different** values on the "no score supplied" path — the whole defect crystallises when `updateAlertDto.priorityScore` is `undefined`.

### 2.2 Why Prisma leaves the column NULL

`AlertRepository.updateAlert` at [`backend/src/modules/repository/alert.repository.ts:141`](repos/case-management-system/backend/src/modules/repository/alert.repository.ts#L141) passes the value straight through:

```typescript
data: {
  priority_score: updateData.priority_score,   // undefined -> "field not provided"
  priority: updateData.priority,
  ...
}
```

Prisma semantics: **`undefined` = "do not touch this column"**. So when `priority_score` arrives as `undefined`, the DB row is not updated for that column.

### 2.3 Why the column is NULL at birth

`AlertRepository.createAlert` at [`alert.repository.ts:16-40`](repos/case-management-system/backend/src/modules/repository/alert.repository.ts#L16-L40) (**verified**): the `data` object passed to `client.alert.create` sets `tenant_id`, `priority: Priority.LOW`, `source`, `txtp`, `confidence_per`, `message`, `alert_data`, `transaction`, `network_map`, `case_id` — **and no `priority_score`**. The Prisma column therefore starts life as `NULL`.

Combined with 2.2, if manual triage runs with `priorityScore` omitted:

- **Before triage:** `priority_score = NULL`, `priority = LOW` (baseline from `createAlert`).
- **After triage:** `priority_score = NULL` (still — Prisma didn't touch it), `priority = <bucket of 0.33>` (currently `LOW`).

So we end up with an alert whose **stored severity has no stored input**. Exactly the "audit chain broken" state the issue's §8/§10 language is designed to prevent.

### 2.4 Why Sandy calls this a *defect* rather than a design choice

She points out that **every** other bucket-and-persist site stores the same variable it bucketed:

| Site                                                                     | Bucket call                                                                           | Persist call                                                  | Same variable? |
| ------------------------------------------------------------------------ | ------------------------------------------------------------------------------------- | ------------------------------------------------------------- | -------------- |
| `case-creation.service.ts` L152–153 / L209 (**verified**)       | `determinePriority(priorityScore, ...)`                                             | `priority_score: priorityScore`                             | ✅ yes         |
| `case-creation-approval.service.ts` L63–64 / L78 (**verified**) | `determinePriority(priorityScore, ...)`                                             | `createDraftCase(caseDetail, dto, priorityScore, priority)` | ✅ yes         |
| `triage.service.ts` **AI path** L925/L939 (**verified**)   | `priority` passed in from caller (bucketed from `predictedPriorityScore` at L567) | `updateDto.priority_score = predictedPriorityScore`         | ✅ yes         |
| `triage.service.ts` **manual path** L303–331                    | `determinePriority(priorityScore /* defaulted */, ...)`                             | `priority_score: updateAlertData.priorityScore /* raw */`   | ❌**no** |

Manual triage is the sole outlier. That asymmetry is what makes this look like a mistake, not a policy. The AI triage path in the same file gets it right; the manual path does not.

### 2.5 Why it matters beyond audit

Sandy identifies two downstream consequences:

1. **Blocks the doors §12 keeps open.** Tenant-configurable severity-bucket thresholds (§12) rely on being able to re-derive `LOW/MEDIUM/HIGH` from the *stored* scores if cut points are moved. A `NULL` `priority_score` on a row cannot be re-bucketed — it becomes a permanently orphaned severity.
2. **Poisons the manual-labelling dataset.** The MANUAL triage phase is (per project intent) the source of hand-labelled data for training the AI triage model. A row that reads "no score, LOW severity" is an unusable training example — the input field the model would consume is missing, but a label exists.

---

## 3. Defect 2 — the `0.33` default is a fossil of the old ladder

### 3.1 The old semantics (from §1 of the issue itself)

§1 of the issue documents the ladder being replaced:

> `>= 1.0 -> BREACH`, `>= 0.66 -> CRITICAL`, `>= 0.33 -> URGENT`, else `NEW`

Under those rules, `0.33` was the exact cut-point where an alert **entered** the elevated tier `URGENT`. Choosing `0.33` as the fallback meant *"if no score is provided, treat this alert as if it has just tipped into elevated urgency."* That was defensible under the old model.

### 3.2 The new semantics (from `case-priority.util.ts`, verified)

At [`backend/src/modules/shared/utils/case-priority.util.ts`](repos/case-management-system/backend/src/modules/shared/utils/case-priority.util.ts):

```typescript
export const FALLBACK_HIGH_THRESHOLD = 0.7;
export const FALLBACK_MEDIUM_THRESHOLD = 0.4;

async determinePriority(priorityScore: number, tenantId: string): Promise<Priority> {
  const { highThreshold, mediumThreshold } = await this.getThresholds(tenantId);
  if (priorityScore >= highThreshold)        return Priority.HIGH;
  else if (priorityScore >= mediumThreshold) return Priority.MEDIUM;
  else                                       return Priority.LOW;
}
```

Since `0.33 < 0.4`, the fallback `0.33` now buckets to **`LOW`**.

### 3.3 What that means operationally

Per §5 of the issue, priority *calibrates the SLA clock*. The MVP fallback SLA budgets are:

| Priority | Target    |
| -------- | --------- |
| HIGH     | 24h       |
| MEDIUM   | 72h       |
| LOW      | 168h (7d) |

So the default `0.33` used to mean **"URGENT — first rung above baseline"**; it now means **"LOW — slackest deadline (7 days)"**. Same literal, inverted semantics.

Sandy's characterisation — *"silently inverted from 'elevated urgency' to 'minimum severity, slackest deadline'"* — is exact.

### 3.4 Why nothing caught the inversion

The one test that exercises the default is `test/triage.service.spec.ts` L1315-1332 (**verified**), *"should handle manual triage with undefined priority score"*:

```typescript
casePriorityUtil.determinePriority.mockResolvedValue(Priority.LOW);
...
const dtoWithoutPriorityScore: ManualAlertUpdateDTO = {
  priority: Priority.LOW,
  alertType: CaseType.FRAUD,
  note: 'test note',
};

const result = await service.handleManualTriage(1, dtoWithoutPriorityScore, ...);
expect(casePriorityUtil.determinePriority).toHaveBeenCalledWith(0.33, 'tenant-123');
```

This test **mocks** `determinePriority` and only asserts that it was **called** with `0.33`. It:

- Does **not** verify which bucket `0.33` should land in — the mock returns whatever the test author wanted (`Priority.LOW`), disconnected from the real thresholds.
- Does **not** assert what value gets **persisted** as `priority_score`.
- Does **not** exercise the `alertService.updateAlert → alertRepository.updateAlert` path with the actual DTO shape to see that `priority_score` ends up NULL.

Sandy is right: the test "pins the fossil" (the literal `0.33`) without catching either defect. In fact, hard-coding the mock return to `Priority.LOW` accidentally documents the inversion as expected behaviour, which is worse than not testing it.

---

## 4. DTO-level context Sandy cites

At [`backend/src/modules/alert/dto/alert.dto.ts:29-39`](repos/case-management-system/backend/src/modules/alert/dto/alert.dto.ts#L29-L39) (**verified**):

```typescript
@ApiProperty({
  description: 'Priority score (0-1)',
  example: 0.85,
  required: false,
  type: 'number',
  minimum: 0,
  maximum: 1,
})
@IsOptional()
@IsNumber()
priorityScore?: number;
```

Two things Sandy points at implicitly:

- **`@IsOptional()` is why the DTO can arrive with no score.** Given the model says "priority is stamped from priorityScore at triage", allowing the field to be absent at the triage endpoint is arguably the design bug that made the fallback necessary in the first place.
- **Swagger declares `minimum: 0, maximum: 1` but the runtime validators do not enforce it.** The `@IsNumber()` decorator alone accepts any finite number — `1.5`, `-2`, `NaN`... all would bucket to whatever the thresholds resolve to (`1.5 >= 0.7 → HIGH`; `-2 < 0.4 → LOW`). The "0–1 contract" is documented but not enforced.

---

## 5. Sandy's proposed resolution

She offers three options with a clear preference, plus a hardening step and a test:

### 5.1 Options (from her comment)

1. **Persist the defaulted value.** Change L331 from `priority_score: updateAlertData.priorityScore` to `priority_score: priorityScore` (the local, already `?? 0.33`-defaulted). One-token change. Makes score and bucket agree by construction. **Fixes Defect 1**; leaves Defect 2 (`0.33 → LOW` inversion) intact.
2. **(Preferred)** **Make `priorityScore` required in `ManualAlertUpdateDTO`.** Drop `@IsOptional()`, drop the `?`, add `@IsNumber()` (already present). This is what the model actually implies — *"triage stamps priority from priorityScore"* means triage must supply the score. Removes the need for a default entirely. **Fixes both defects** by making the "no score" path impossible.
3. **If a default must remain,** choose it deliberately against the current thresholds and **document which bucket it lands in**. This is the fallback fix if requiring the score is not acceptable for some reason (e.g. UI backward-compat).

### 5.2 Hardening (independent of which option is picked)

- **Range validation on the DTO.** Add `@Min(0)` and `@Max(1)` to enforce the 0–1 contract that the Swagger declaration promises. Closes the currently-unvalidated 0–1 contract.

### 5.3 Regression test she proposes

> A regression test asserting **"manual triage with no score persists a non-NULL `priority_score` consistent with the stored `priority`"** would pin the invariant.

Key features of this proposed test that distinguish it from the current one:

- Do **not** mock `determinePriority` — use the real util so the thresholds and the default value actually collide as they would in production.
- Assert on the value **persisted** (i.e. the mock `alertRepository.updateAlert` should receive a non-`undefined` `priority_score`, and the value should match the bucket).
- Assert on the **relationship** (score → bucket) rather than the **literal** default value, so future default changes don't silently invert semantics again.

---

## 6. What Sandy is *not* saying

To bound the scope precisely — a few things her comment does **not** claim:

- She is **not** saying `priority` gets overwritten by the SLA job. That was the "original sin" the issue was built to remove, and Sandy's earlier ✔ IMPLEMENTED comment already confirmed the recalc job no longer writes `priority`. Defects 1 and 2 are about the **stamping** step, not the SLA recalc step.
- She is **not** asking for a schema change. Neither `Case` nor `Alert` needs a new column. The fixes are pure application-code + DTO-validation changes.
- She is **not** asking for tenant configuration of the default value. If a default is kept (option 3), she wants it chosen deliberately and documented — not made configurable.
- She is **not** claiming other bucket-and-persist sites are broken. She explicitly acknowledges that `case-creation.service.ts`, `case-creation-approval.service.ts`, and the AI triage path all use the same variable for bucket and persist — which is exactly what makes manual triage a clear outlier.

---

## 7. Recommended sequencing of a fix

Combining Sandy's preferred option with the hardening she names, in a two-PR shape that matches how #220 has been landed so far:

**PR 1 — Defect 1 (immediate fix, keeps `?` on the DTO).**

1. `triage.service.ts` L331 — change `updateAlertData.priorityScore` → `priorityScore` (the defaulted local). One-token change.
2. Optional in same PR: choose a deliberate default (e.g. `0.5` for MEDIUM, or `0.7` for HIGH) and add a comment naming the bucket. This closes Defect 2 within the "keep a default" option.
3. Add the regression test she proposes, wired to the real `CasePriorityUtil` (or at minimum, asserting the persisted `priority_score`).

**PR 2 — Defect 2 the correct way (make the score required).**

1. `alert.dto.ts` — remove `@IsOptional()` and the `?` on `priorityScore`; add `@Min(0)` and `@Max(1)`.
2. `triage.service.ts` L303 — remove the `?? 0.33` default; simplify to `const priorityScore = updateAlertDto.priorityScore;`.
3. Update the "undefined priority score" test to assert a validation error is returned by the controller, rather than a defaulted bucket.
4. Frontend impact check (out of scope of this document, but flag for anyone implementing): the manual-triage submit form must supply a score in every case, or the API will now 400. This is intended per the issue model, but is a UI change that needs coordinated release.

If only PR 1 ships, the audit-chain break (Defect 1) is fixed but the fossil default still buckets to LOW — acceptable stopgap, but not the settled state the issue asks for.

---

## 8. Verification summary

Every code claim above was checked against `origin/dev` @ `fa31dec6`:

| Claim                                                                                                  | File                                                                                                                                     | Lines        | Verified                    |
| ------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------------------- |
| `handleManualTriage` defaults `priorityScore` to `0.33` for bucketing but persists raw DTO field | [`triage.service.ts`](repos/case-management-system/backend/src/modules/triage/triage.service.ts)                                        | 303, 331     | ✅                          |
| `ManualAlertUpdateDTO.priorityScore` is `@IsOptional()`, no `@Min`/`@Max`                      | [`alert.dto.ts`](repos/case-management-system/backend/src/modules/alert/dto/alert.dto.ts)                                               | 29-39        | ✅                          |
| `createAlert` never sets `priority_score` at ingestion                                             | [`alert.repository.ts`](repos/case-management-system/backend/src/modules/repository/alert.repository.ts)                                | 16-40        | ✅                          |
| `updateAlert` passes `priority_score` straight through (undefined → no-op)                        | [`alert.repository.ts`](repos/case-management-system/backend/src/modules/repository/alert.repository.ts)                                | 141          | ✅                          |
| Fallback thresholds are`0.7 / 0.4`; `0.33 → LOW`                                                  | [`case-priority.util.ts`](repos/case-management-system/backend/src/modules/shared/utils/case-priority.util.ts)                          | 6-7, 36-45   | ✅                          |
| `case-creation.service.ts` bucket-and-persist uses same variable                                     | [`case-creation.service.ts`](repos/case-management-system/backend/src/modules/case/services/case-creation.service.ts)                   | 152-153, 209 | ✅                          |
| `case-creation-approval.service.ts` bucket-and-persist uses same variable                            | [`case-creation-approval.service.ts`](repos/case-management-system/backend/src/modules/case/services/case-creation-approval.service.ts) | 63-64, 78    | ✅                          |
| AI triage path persists`predictedPriorityScore` (same var used to bucket at L567)                    | [`triage.service.ts`](repos/case-management-system/backend/src/modules/triage/triage.service.ts)                                        | 564-567, 939 | ✅                          |
| "undefined priority score" test mocks`determinePriority` and only asserts the arg                    | [`triage.service.spec.ts`](repos/case-management-system/backend/test/triage.service.spec.ts)                                            | 1315-1332    | ✅                          |
| Old ladder`>= 0.33 → URGENT`                                                                        | Issue#220 body                                                                                                                           | §1          | ✅ (verified in issue text) |
| §5 SLA fallback budgets`24h / 72h / 168h`                                                           | Issue#220 body                                                                                                                           | §5          | ✅ (verified in issue text) |

Nothing in Sandy's comment is speculative or based on out-of-date code. Her diagnosis maps 1:1 to the current state of `dev`.

---

## 9. Sandy's other comments on #220 (context, not the main subject)

For completeness, so this file stands alone as a briefing:

### Commits on Jul 16, 2026

* #### [fix: align Lakehouse Catalog §11b join with gold_transactions raw column names](https://github.com/tazama-lf/biar/pull/130/changes/cdb443798678d94965b28250b2867e2c6d2c754e "fix: align Lakehouse Catalog §11b join with gold_transactions raw column names

  gold_transactions writes raw ISO field names (endtoendid, tenantid, txtp,
  amt, ccy, txsts) whereas gold_pacs002/pacs008 rename them to snake_case.
  §11b assumed the renamed convention, so the SELECT and INNER JOIN
  against alias t raised UNRESOLVED_COLUMN AnalysisException.

  Alias the raw names in the SELECT list and reference them directly in
  the join predicates for pacs002 and gold_account. Scope is one cell in
  JupyterHub/notebooks/Lakehouse_Catalog.ipynb; no ETL or other notebook
  touched.

  Fixes #129

  Signed-off-by: Ahmad Khalid &lt;ahmad.khalid@paysyslabs.com&gt;")

  **gold_transactions writes raw ISO field names (endtoendid, tenantid, txtp,
  amt, ccy, txsts) whereas gold_pacs002/pacs008 rename them to snake_case.
  §11b assumed the renamed convention, so the SELECT and INNER JOIN
  against alias t raised UNRESOLVED_COLUMN AnalysisException.
  **

- **Comment (SLA filter gap):** on the cases dashboard she can only filter by priority, not by SLA status — "I can't filter for high priority and breached cases". You have already committed to raising a follow-up issue for this.
- **Comment (supervisor priority-edit):** as supervisor she cannot see where to edit priority per §8; asked whether priority is an attribute of alert, case, or both. This is a UI-visibility question, not a model complaint.
- **Comment (✔ IMPLEMENTED audit):** her earlier verification confirms the schema and service side of #220 landed as designed (Priority enum shrunk, `sla_state` derived not stored, escalation table wired). This is the comment where she signs off on the *model*.
- **Comment (tenant-id on `sla_escalation_records`):** she noted the table lacked `tenant_id`. This was addressed on `dev` at commit `fa31dec6` via migration `20260716120000_add_tenant_id_to_sla_escalation_records`. This is closed.
- **Comment (this one — issuecomment-4972261159):** the AI-assisted code review producing the two defects analysed above.

The first four are separate threads; only the last is the technical defect report that needs a code response.
