# CMS #169 — Sandy's Feedback Analysis (comment `#issuecomment-4961156073`)

**Repository:** [tazama-lf/case-management-system](https://github.com/tazama-lf/case-management-system)
**Issue:** [#169 — Display the subruleref for the ERFUP rule](https://github.com/tazama-lf/case-management-system/issues/169)
**Comment under analysis:** [issuecomment-4961156073](https://github.com/tazama-lf/case-management-system/issues/169#issuecomment-4961156073) — posted by **Sandy-at-Tazama**, 2026-07-13
**Prepared:** 2026-07-17 (verified against `case-management-system` @ `fa31dec6` and `biar` @ `978c503f`)

---

## 0. Executive summary

The feature request (#169) asks that the EFRuP rule's `subRuleRef` (values `override` / `block` / `none`) be surfaced above the triggered-typologies list on the alert view. Sandy's comment confirms the feature was implemented — the SQL, backend mapping, and frontend banner all exist and behave as specified — but flags a **residual defect**: the entire mechanism is pinned to the exact literal `'EFRuP@1.0.0'` at multiple locations across two repositories. If the EFRuP rule is ever bumped to `EFRuP@2.0.0` (or any other version), the feature silently disappears at every layer, and no error is raised.

Her diagnosis is accurate on every code-referenced point. Verifying it adversarially — trying to find a rescue path, a shared constant, a prefix-match, or any hedging that would prevent the silent-failure mode she describes — turned up **no such rescue path in either repository**. In fact the situation is slightly *worse* than her comment suggests: she cites three hardcoded literals, but there is a **fourth** in `AlertsDetailModal.tsx` that exhibits the same defect.

The header-level `blockReason` field she flags as a related residual (currently derived from `transaction_status`, not from the EFRuP subRuleRef) is a separate biar-side concern tracked as biar #94; it is mentioned here for context but is not the subject of the residual risk in her comment.

---

## 1. What #169 required and what shipped

The issue body is a one-liner plus a screenshot: *"display the subruleref for the ERFUP rule which could be 'override' or 'block' or 'none' above the list of triggered typologies."* No SLA, no schema — a pure UI/data-plumbing request.

The shipped fix has three cooperating layers:

1. **SQL (CMS backend).** The rules CTE in `alerts-lakehouse.service.ts` normally filters weight-0 rules out of the aggregation. EFRuP carries `rule_weight = 0` (confirmed as an invariant, see §5.1). The CTE has an explicit exception to keep EFRuP rows despite their zero weight.
2. **Mapping (CMS backend).** The same service extracts an `flowProcessorRule` from the rules array by rule id, and exposes `flowProcessorData = flowProcessorRule?.rule_sub_ref` on each typology.
3. **Rendering (CMS frontend).** `AlertNavigatorTab.tsx` finds the first typology that carries a `flowProcessorData` and renders it in a dedicated banner directly above the triggered-typologies section.

There is also a parallel path in the biar automation-orchestrator that materialises `gold/alerts.efrup_subruleref` for downstream metrics — same defect pattern, different repo.

---

## 2. Claim-by-claim verification

Every claim below was checked against the current `dev` HEAD in each repo. File paths, line numbers, and quoted code are all direct reads, not paraphrase.

### 2.1 The SQL CTE keeps weight-0 EFRuP rows explicitly

**Claim:** [`alerts-lakehouse.service.ts`](repos/case-management-system/backend/src/modules/gold-lakehouse/alerts-lakehouse.service.ts#L60) line 60 has `anr.rule_weight > 0 OR anr.rule_id = 'EFRuP@1.0.0'` inside a rules CTE.

**Verified.** The `rules_agg` CTE at L38-69 aggregates per-typology rules into a struct array. Its WHERE clause (L58-63):

```sql
WHERE anr.alert_id  = ${safeAlertId}
  AND anr.tenant_id = '${safeTenantId}'
  AND (
    anr.rule_weight > 0
    OR anr.rule_id = 'EFRuP@1.0.0'
  )
```

Semantics as Sandy describes: any weight-0 rule row is dropped, **except** rows with `rule_id = 'EFRuP@1.0.0'` which are kept regardless of weight.

### 2.2 The mapping extracts EFRuP by exact id and filters triggered rules by weight

**Claim:** L172-173, 184 of the same file:

```typescript
const flowProcessorRule = rulesData.find((r) => r.rule_id === 'EFRuP@1.0.0');
const triggeredRulesData = rulesData.filter((r) => (r.rule_weight ?? 0) > 0);
...
const flowProcessorData = flowProcessorRule?.rule_sub_ref ?? undefined;
```

**Verified.** Direct read of [alerts-lakehouse.service.ts:172-184](repos/case-management-system/backend/src/modules/gold-lakehouse/alerts-lakehouse.service.ts#L172-L184). Two important properties of this code:

- `triggeredRulesData` uses **no rule-id blacklist**. EFRuP is excluded from the triggered list solely because its weight is 0. Sandy's characterisation — *"implicitly (its weight is 0), not by rule ID"* — is exact.
- `flowProcessorData` is optional (`?? undefined`), and the frontend banner is conditionally rendered on it (§2.4). This is what makes the silent-failure mode possible.

### 2.3 The frontend picks the first typology carrying `flowProcessorData`

**Claim:** [`AlertNavigatorTab.tsx`](repos/case-management-system/frontend/src/features/cases/components/view/visualizations/alertnavigator/AlertNavigatorTab.tsx#L134-L136) L134-136 picks the first matching typology.

**Verified.** L134-136:

```typescript
const flowProcessorData = data.typologies?.find(
  (typology) => typology.flowProcessorData,
)?.flowProcessorData;
```

`Array.prototype.find` returns the first match — so the banner reflects the EFRuP subRuleRef from whichever typology it appears on first. That is fine given the current data model (EFRuP is a shared flow-processor rule that reports the same subRuleRef across typologies), but if that ever changes, the "first-wins" choice is silently non-deterministic. Not raised by Sandy; noted here for completeness.

### 2.4 The banner renders above the triggered-typologies list

**Claim:** L239-242 render the EFRuP banner above the triggered-typologies section.

**Verified.** L233-247 of `AlertNavigatorTab.tsx`:

```tsx
<div className="rounded-lg border border-gray-200 bg-white p-5">
  <div className="mb-4 grid grid-cols-3 items-center">
    <h4 className="text-sm font-semibold text-gray-900">Triggered Typologies</h4>

    {flowProcessorData && (
      <div className="justify-self-center text-sm font-semibold">
        <span className="text-gray-900">EFRuP:</span>{' '}
        <span className="text-red-600">{flowProcessorData}</span>
      </div>
    )}
  </div>

  <div className="space-y-3">
    {data.typologies && data.typologies.length > 0 ? ...
```

The EFRuP banner sits in the header grid **above** the typologies list (L247+). The `{flowProcessorData && ...}` guard is the silent-failure surface: when `flowProcessorData` is `undefined`, the banner simply does not render — no error, no placeholder.

### 2.5 The header `blockReason` renders separately

**Claim:** L220-226 render `block_or_override_status` as `Block Status` at header level.

**Verified.** L220-228:

```tsx
{data.alertMetadata.blockReason && (
  <div className="col-span-2">
    <div className="text-xs font-medium text-gray-500 uppercase mb-1">Block Status</div>
    <div className="text-sm text-gray-900">
      {data.alertMetadata.status} - {data.alertMetadata.blockReason}
    </div>
  </div>
)}
```

This is the field Sandy flags in her ⚠️ point 3 (biar #94): the value being rendered here is derived upstream from `transaction_status`, not from the EFRuP `subRuleRef`. That's an independent concern from the version-pin residual and is out of scope for the fix this document is analysing.

### 2.6 The biar ETL filters EFRuP by exact id

**Claim:** [`ALertsETL.py`](repos/biar/automation-orchestrator/Table_ETLs/ALertsETL.py#L225) L225 filters `r.id = 'EFRuP@1.0.0'`.

**Verified.** L211-234 populate `efrup_subruleref` via:

```python
.withColumn(
"efrup_subruleref",
F.expr("""
    element_at(
        flatten(
            transform(
                filter(
                    alert_data_obj.tadpResult.typologyResult,
                    t -> t is not null
                ),
                t -> transform(
                    filter(
                        t.ruleResults,
                        r -> r is not null
                            AND r.id = 'EFRuP@1.0.0'
                    ),
                    r -> r.subRuleRef
                )
            )
        ),
        1
    )
""")
)
```

`element_at(flatten([]), 1)` on Spark returns `NULL`. For a hypothetical `EFRuP@2.0.0`-triggered alert, the inner `filter` drops every rule row (the literal doesn't match), `transform` yields empty arrays, `flatten` returns `[]`, and `element_at(..., 1)` returns `NULL`. The column ends up `NULL` with no error. Sandy's "goes NULL for v2 alerts" is exact.

### 2.7 The core adversarial claim — v2 EFRuP would be dropped entirely

**Claim:** A hypothetical `EFRuP@2.0.0` row would be dropped from the CTE result because:
- Its weight is 0, so `anr.rule_weight > 0` fails.
- Its `rule_id` is `'EFRuP@2.0.0'`, so `= 'EFRuP@1.0.0'` fails.

**Verified — and no rescue path exists.** Read the full CTE (L38-69). There is no other branch in the WHERE clause. There is no fallback JOIN, no `COALESCE`, no default row, no case-insensitive comparison. If both OR branches evaluate false, the row is not aggregated into the `rules` struct array for that typology. Downstream, `flowProcessorRule` is `undefined`, `flowProcessorData` is `undefined`, the frontend banner does not render — every layer fails together, silently.

This is the load-bearing claim in her comment. It stands.

### 2.8 Excluding EFRuP from `triggeredRulesData` is by weight, not by id

**Claim:** L173 filters on `(r.rule_weight ?? 0) > 0` — no rule-id blacklist.

**Verified.** There is no rule-id filter anywhere in the mapping. EFRuP's exclusion from the triggered-rules table is *entirely* a consequence of its weight being 0. This is important because it means the "rules table" surface is *not* pinned to `EFRuP@1.0.0` — a v2 EFRuP row would erroneously be *included* in the triggered rules list if it were ever to carry a non-zero weight. Sandy doesn't mention this reverse direction, but it is worth flagging as part of the same version-pin concern.

---

## 3. What Sandy missed — a fourth version-pinned literal

Sandy identifies **three** exact-match literals. There are actually **four**. The fourth is in a different frontend component and was not cited in her comment.

**File:** [`frontend/src/features/alerts/components/AlertsDetailModal.tsx`](repos/case-management-system/frontend/src/features/alerts/components/AlertsDetailModal.tsx#L166-L187) L166-L187.

```typescript
const EFRUP_RULE_ID = 'EFRuP@1.0.0';
const extractFlowProcessorData = (
  alert: AlertWithAlertedTypologies,
): string | undefined => {
  const tadpResult = isRecord(alert.alert_data)
    ? alert.alert_data.tadpResult
    : undefined;
  const typologyResult = isRecord(tadpResult) ? tadpResult.typologyResult : undefined;
  if (!Array.isArray(typologyResult)) {
    return undefined;
  }
  for (const typology of typologyResult) {
    if (!isRecord(typology)) continue;
    const ruleResults = Array.isArray(typology.ruleResults)
      ? typology.ruleResults.filter(isRecord)
      : [];
    const flowProcessorRule = ruleResults.find(
      (rule) => rule.id === EFRUP_RULE_ID,
    );
    const subRuleRef = flowProcessorRule?.subRuleRef;
    if (typeof subRuleRef === 'string' && subRuleRef.trim()) {
      return subRuleRef;
    }
  }
  return undefined;
};
```

This is a parallel implementation in the alerts-list modal (not the alert-navigator tab covered by claims 2.3–2.4). Same defect pattern: exact-literal comparison, `undefined` fall-through, silent failure. Notably it's the only site that already extracts the id into a named constant (`EFRUP_RULE_ID`), which makes it *slightly* easier to migrate to prefix-matching or a shared config — but the literal is still `'EFRuP@1.0.0'`.

This makes Sandy's argument **stronger**, not weaker: there are four version-pinned surfaces (three in CMS, one in biar), not three, and any fix that only touches the three she mentions would leave the alerts-modal path silently broken on v2.

---

## 4. Test coverage — why no test catches this

### 4.1 Fixtures pin the literal instead of the invariant

`backend/test/alerts-lakehouse.service.spec.ts:111` explicitly stubs an EFRuP rule with `rule_id: 'EFRuP@1.0.0'` and `rule_weight: 0`. `frontend/src/features/alerts/components/__tests__/AlertsDetailModal.test.tsx:411,427` do the same in JS fixtures. Every test asserts against the literal `'EFRuP@1.0.0'`, so a code change that moved to `'EFRuP@2.0.0'` would silently pass without banner or subruleref appearing in production alerts — until a real v2 alert flowed through and no one noticed the missing UI.

### 4.2 No prefix-match anywhere in either codebase

Grep across `case-management-system/backend`, `case-management-system/frontend`, and `biar/automation-orchestrator` finds **zero** uses of a prefix match or `startsWith`/`LIKE 'EFRuP@%'`/`^EFRuP@` regex against the rule id. The four exact-match literals are the only mechanism.

### 4.3 EFRuP `rule_weight = 0` is an invariant with no enforcement

The exception at L60 of the SQL CTE only makes sense if EFRuP is *always* weight 0. Nothing in the schema or migrations enforces this — it's a data-plane convention. If a future EFRuP release ships with a non-zero weight, the row is picked up by the *first* branch of the CTE (`rule_weight > 0`) and included in the triggered-rules list, which is not what any UI code was written to handle. Not something Sandy raises, but worth being explicit about as part of the same fragility.

### 4.4 No comment or ADR explaining the pin

The commit that introduced the exact-match literal (`aa2ba81`, 2026-06-23, *"feat: fixed the high priority feature requests"*) has no message text explaining why `1.0.0` was hardcoded rather than prefix-matched. There is no comment near any of the four literals justifying the choice. Adopting Sandy's proposed prefix-match doesn't break any documented intent — there is no documented intent.

---

## 5. Sandy's recommended fix and where it applies

She proposes: prefix-match (`LIKE 'EFRuP@%'` on the SQL side, `startsWith('EFRuP@')` on the TS side) or a shared config constant. Applied consistently, this would remove the version pin from all four sites:

| # | Site | Repo | Current | Proposed |
| --- | --- | --- | --- | --- |
| 1 | `alerts-lakehouse.service.ts` L60 (SQL CTE) | CMS backend | `anr.rule_id = 'EFRuP@1.0.0'` | `anr.rule_id LIKE 'EFRuP@%'` |
| 2 | `alerts-lakehouse.service.ts` L172 (TS `.find`) | CMS backend | `r.rule_id === 'EFRuP@1.0.0'` | `r.rule_id.startsWith('EFRuP@')` |
| 3 | `AlertsDetailModal.tsx` L166 (`EFRUP_RULE_ID` const + L186 comparison) | CMS frontend | `EFRUP_RULE_ID = 'EFRuP@1.0.0'` + `rule.id === EFRUP_RULE_ID` | `rule.id.startsWith('EFRuP@')` |
| 4 | `ALertsETL.py` L225 (Spark SQL filter) | biar automation-orchestrator | `r.id = 'EFRuP@1.0.0'` | `r.id LIKE 'EFRuP@%'` |

Site 3 was not mentioned in Sandy's comment (see §3). Any PR that addresses sites 1, 2, and 4 without site 3 leaves the alerts-modal `subRuleRef` broken on any future EFRuP version, so a complete fix must touch all four.

A shared constants file (e.g. `EFRUP_RULE_ID_PREFIX = 'EFRuP@'`) would be the natural home if the team wants a single source of truth across TS and Python — though a plain prefix-match at each site is also acceptable for a fix this small.

Two secondary considerations worth pairing with the fix:

- **Enforce the weight-0 invariant.** If EFRuP is architecturally weight-0, a schema check or upstream validation would make the invariant explicit and defend against the reverse failure noted in §2.8.
- **Update tests to assert the invariant, not the literal.** Fixtures should still use a concrete rule id (`'EFRuP@1.0.0'` is fine as sample data), but assertions should target the behaviour ("EFRuP subRuleRef appears in the banner regardless of the version suffix"), and at least one test case should use a non-`1.0.0` version to prove the prefix-match holds.

---

## 6. What Sandy is *not* saying

To bound the scope precisely — a few things her comment does **not** claim:

- She is **not** saying the current fix is wrong on `EFRuP@1.0.0`. Every layer works correctly for the current version; her ✅ points confirm the fix behaves as #169 asks.
- She is **not** asking for the `blockReason` header field to be re-plumbed to use the EFRuP `subRuleRef`. That's a separate biar-side concern she flags as biar #94 and explicitly notes is not the subject of this comment.
- She is **not** asking for a schema change. All four proposed fix sites are code-only edits.
- She is **not** asking for the rule-id filter to be removed entirely. The `triggeredRulesData` weight filter should stay — EFRuP must continue to be excluded from the triggered-rules list. Her critique is about *how* the version match is expressed, not whether EFRuP should be special-cased.

---

## 7. Verification summary

Every code claim above was checked against `case-management-system` @ `fa31dec6` and `biar` @ `978c503f`:

| Claim | File | Lines | Verified |
| --- | --- | --- | --- |
| SQL CTE keeps EFRuP@1.0.0 rows via explicit OR branch | [`alerts-lakehouse.service.ts`](repos/case-management-system/backend/src/modules/gold-lakehouse/alerts-lakehouse.service.ts) | 58-63 | ✅ |
| Backend mapping extracts EFRuP by exact id, exposes `rule_sub_ref` as `flowProcessorData` | [`alerts-lakehouse.service.ts`](repos/case-management-system/backend/src/modules/gold-lakehouse/alerts-lakehouse.service.ts) | 172, 184 | ✅ |
| `triggeredRulesData` filters by weight only, no rule-id blacklist | [`alerts-lakehouse.service.ts`](repos/case-management-system/backend/src/modules/gold-lakehouse/alerts-lakehouse.service.ts) | 173 | ✅ |
| Frontend picks first typology with `flowProcessorData` (via `.find`) | [`AlertNavigatorTab.tsx`](repos/case-management-system/frontend/src/features/cases/components/view/visualizations/alertnavigator/AlertNavigatorTab.tsx) | 134-136 | ✅ |
| Frontend banner renders above triggered-typologies list, guarded by `{flowProcessorData && ...}` | [`AlertNavigatorTab.tsx`](repos/case-management-system/frontend/src/features/cases/components/view/visualizations/alertnavigator/AlertNavigatorTab.tsx) | 233-247 | ✅ |
| Header `blockReason` (`block_or_override_status`) rendered as `Block Status` | [`AlertNavigatorTab.tsx`](repos/case-management-system/frontend/src/features/cases/components/view/visualizations/alertnavigator/AlertNavigatorTab.tsx) | 220-228 | ✅ |
| biar ETL filters by exact `EFRuP@1.0.0` id, `element_at` returns NULL on empty match | [`ALertsETL.py`](repos/biar/automation-orchestrator/Table_ETLs/ALertsETL.py) | 211-234 | ✅ |
| **Undisclosed fourth literal in `AlertsDetailModal.tsx`** | [`AlertsDetailModal.tsx`](repos/case-management-system/frontend/src/features/alerts/components/AlertsDetailModal.tsx) | 166, 186 | ✅ (Sandy did not cite this) |
| No prefix-match / shared constant anywhere in either repo | Grep across `case-management-system` + `biar/automation-orchestrator` | — | ✅ |
| EFRuP `rule_weight = 0` invariant is a data-plane convention (no schema enforcement) | Migrations + fixtures | — | ✅ (fixture at `alerts-lakehouse.service.spec.ts:111`; no schema check) |

**Adversarial verdict.** Every load-bearing claim in Sandy's comment stands. The one gap in her write-up is a *missing* site (`AlertsDetailModal.tsx`) which strengthens her thesis rather than undermining it — a fix that addresses only the three sites she cites will leave a fourth silently broken on any future EFRuP version. Nothing in her comment is inaccurate, and no rescue path exists in either codebase that would prevent the silent-failure mode she describes on `EFRuP@2.0.0`.
