# PR Review: TRS #126 — feat: add variable with data type node

**Repo:** tazama-lf/rule-studio  
**Branch:** `feat-paysys-variable-type-node` → `dev`  
**Author:** ReebaPaysys (Reeba Siddiqui)  
**Date Reviewed:** 2026-07-02  
**Label:** enhancement  
**Size:** +73 / -7 lines across 3 files  
**Commits:** 3 (`670a4c1b`, `8d9eb872`, `ed0b6c8b`)  
**State:** OPEN

---

## Table of Contents

- [Overview](#overview)
- [What Changed (Detailed)](#what-changed-detailed)
  - [1. CodeGenerator.ts — New Helper Functions](#1-codegeneratorts--new-helper-functions)
  - [2. CodeGenerator.ts — New Node Dispatch Branch](#2-codegeneratorts--new-node-dispatch-branch)
  - [3. CodeGenerator.ts — generateSetVariableWithTypeCode](#3-codegeneratorts--generatesetvariablewithtypecode)
  - [4. CodeGenerator.ts — escapeForInterpolatedTemplate Refactor](#4-codegeneratorts--escapeforinterpolatedtemplate-refactor)
  - [5. VariableManager.ts — SetVariableWithType Recognition](#5-variablemanagerts--setvariablewithtype-recognition)
  - [6. validation/schemas/index.ts — Schema Registration](#6-validationschemasindexts--schema-registration)
- [Code Quality Analysis](#code-quality-analysis)
  - [Strengths](#strengths)
  - [Issues and Observations](#issues-and-observations)
- [Security Assessment](#security-assessment)
- [Test Coverage](#test-coverage)
- [CodeRabbit and Bot Review Activity](#coderabbit-and-bot-review-activity)
  - [GitHub Advanced Security — CodeQL Finding](#github-advanced-security--codeql-finding)
  - [CodeRabbit Inline Comments](#coderabbit-inline-comments)
- [Resolution Status of Bot-Flagged Issues](#resolution-status-of-bot-flagged-issues)
- [Summary and Verdict](#summary-and-verdict)

---

## Overview

[↑ Back to top](#pr-review-trs-126--feat-add-variable-with-data-type-node)

This PR adds a new `SetVariableWithType` node type to the Rule Studio flow editor. The feature extends the existing `SetVariable` node with explicit TypeScript type casting via `as unknown as <dataType>` assertions, allowing users to declare typed variables in the code generator output.

Three files are changed:

| File | Nature of Change |
|------|-----------------|
| `frontend/src/utils/Flow/CodeGenerator.ts` | New dispatch branch + `generateSetVariableWithTypeCode` helper + two helper utilities extracted + escape fix backported to `generateSetVariableCode` |
| `frontend/src/utils/Flow/VariableManager.ts` | `SetVariableWithType` added alongside `SetVariable` in variable extraction and declaration-before-use checks |
| `frontend/src/validation/schemas/index.ts` | `SetVariableWithType` mapped to `setVariableSchema` in the node schema registry |

The PR went through **3 commits**: the initial feature (`670a4c1b`), a fix commit addressing CodeRabbit's first review (`8d9eb872`), and a second fix commit (`ed0b6c8b`). CodeRabbit's review was partially blocked by rate limits on the final commit.

---

## What Changed (Detailed)

[↑ Back to top](#pr-review-trs-126--feat-add-variable-with-data-type-node)

### 1. CodeGenerator.ts — New Helper Functions

[↑ Back to top](#pr-review-trs-126--feat-add-variable-with-data-type-node)

Two module-level utility functions were introduced:

```typescript
const escapeForInterpolatedTemplate = (str: string): string =>
  str.replace(/\\/g, '\\\\').replace(/`/g, '\\`');

const toTsType = (dataType: string): string =>
  (dataType === 'array' ? 'Array<unknown>' : dataType);
```

**`escapeForInterpolatedTemplate`** replaces the inline `.replace(/`/g, '\\`')` call that was scattered across `generateSetVariableCode`, `generateLogCode`, and `generateThrowErrorCode`. It adds correct backslash escaping (`\\` first, then `` ` ``) which was previously missing — the old code only escaped backticks and would have emitted a broken template literal for values containing a trailing backslash.

**`toTsType`** maps the UI's `dataType` field to a valid TypeScript type token. The only non-trivial mapping is `'array'` → `'Array<unknown>'`, since `array` is not a valid TS type keyword.

Both utilities carry explanatory JSDoc comments, which is appropriate given the non-obvious escaping order.

---

### 2. CodeGenerator.ts — New Node Dispatch Branch

[↑ Back to top](#pr-review-trs-126--feat-add-variable-with-data-type-node)

```typescript
if (nodeType === 'SetVariableWithType') {
  return generateSetVariableWithTypeCode(params, indent, generationMode);
}
```

Added in `generateNodeCode` adjacent to the existing `SetVariable` branch. The dispatch is straightforward and follows the established pattern used by all other node types in this function.

---

### 3. CodeGenerator.ts — generateSetVariableWithTypeCode

[↑ Back to top](#pr-review-trs-126--feat-add-variable-with-data-type-node)

This is the substantive addition. The function mirrors the structure of `generateSetVariableCode` but emits `as unknown as <type>` casts on the right-hand side. Full logic:

```typescript
const generateSetVariableWithTypeCode = (
  params: Record<string, string>,
  indent: string,
  mode: 'rule-builder' | 'test-case-generate' = 'test-case-generate'
): string => {
  const varName = params.name || params.variableName || 'variable';
  const declarationType = params.declarationType || 'var';
  const dataType = params.dataType || 'any';
  const originalValue = params.value || params.variableValue || '';

  const isVariableReference = /\{\{\s*.+?\s*\}\}/.test(originalValue);
  const varValue = stripVariableIndicators(originalValue, mode);

  if (!varValue || varValue.trim() === '' || dataType === 'undefined') {
    return `${indent}${declarationType} ${varName};`;
  }

  // ... type-specific branches ...
  return `${indent}${declarationType} ${varName} = ${valueStr};`;
};
```

**Value serialization branches:**

| Condition | Generated output |
|-----------|-----------------|
| Variable reference (`{{...}}`) | `<varValue> as unknown as <type>` |
| Empty value or `dataType === 'undefined'` | `<decl> <name>;` (declaration only) |
| `dataType === 'number'` + numeric literal | `<num> as unknown as number` |
| `dataType === 'boolean'` | `true/false as unknown as boolean` |
| `dataType === 'array'` + already-braced | `[...] as unknown as Array<unknown>` |
| `dataType === 'array'` + unbraced | `[<value>] as unknown as Array<unknown>` |
| `dataType === 'object'` + starts with `{` | raw `<value>` (no cast) |
| `dataType === 'object'` + contains `:` | `{<value>}` (wraps in braces) |
| `dataType === 'object'` + other | `<value> as unknown as object` |
| Numeric literal with `dataType === 'any'` | `<num> as unknown as any` |
| Contains `$` | `` `<escaped template>` `` |
| All other string values | `JSON.stringify(varValue) as unknown as <type>` |

The `JSON.stringify` in the final fallback (added in commit `8d9eb872`) correctly handles values containing embedded quotes, preventing broken TypeScript output like `"He said "hi""`.

---

### 4. CodeGenerator.ts — escapeForInterpolatedTemplate Refactor

[↑ Back to top](#pr-review-trs-126--feat-add-variable-with-data-type-node)

Three existing call sites were updated to use the new helper:

```diff
// generateSetVariableCode
- valueStr = `\`${varValue.replace(/`/g, '\\`')}\``;
+ valueStr = `\`${escapeForInterpolatedTemplate(varValue)}\``;

// generateLogCode
- messageStr = `\`${interpolatedMessage.replace(/`/g, '\\`')}\``;
+ messageStr = `\`${escapeForInterpolatedTemplate(interpolatedMessage)}\``;

// generateThrowErrorCode
- messageStr = `\`${interpolatedMessage.replace(/`/g, '\\`')}\``;
+ messageStr = `\`${escapeForInterpolatedTemplate(interpolatedMessage)}\``;
```

This is a correct and meaningful improvement — the old code would emit broken template literals for inputs like `path\to\file` (the trailing backslash would escape the closing backtick). The new function escapes backslashes first, then backticks, which is the correct order.

---

### 5. VariableManager.ts — SetVariableWithType Recognition

[↑ Back to top](#pr-review-trs-126--feat-add-variable-with-data-type-node)

Two locations in `VariableManager.ts` were updated to recognise `SetVariableWithType`:

**`extractVariablesFromNodes`** (line 23):
```diff
- if (nodeData.nodeType === 'SetVariable') {
+ if (nodeData.nodeType === 'SetVariable' || nodeData.nodeType === 'SetVariableWithType') {
```

**`isVariableDeclaredBefore`** (line 113):
```diff
- if (nodeData.nodeType !== 'SetVariable') return false;
+ if (nodeData.nodeType !== 'SetVariable' && nodeData.nodeType !== 'SetVariableWithType') return false;
```

Both changes are necessary for the new node to integrate correctly with the existing variable tracking system. Without these, variables declared via `SetVariableWithType` would be invisible to downstream reference checks and duplicate-name detection.

The CodeRabbit comment flagging this as missing was actually already addressed before the review fired — the PR's initial commit `670a4c1b` already included both VariableManager changes.

---

### 6. validation/schemas/index.ts — Schema Registration

[↑ Back to top](#pr-review-trs-126--feat-add-variable-with-data-type-node)

```diff
 export const nodeSchemas: Record<string, ObjectSchema<Record<string, unknown>>> = {
   SetVariable: setVariableSchema,
+  SetVariableWithType: setVariableSchema,
```

The new node type reuses the existing `setVariableSchema`. This ensures that `SetVariableWithType` nodes pass through the same Joi/Zod validation as `SetVariable` before reaching code generation — names, declaration types, and data types are validated at the boundary.

One caveat: `setVariableSchema` may not validate the `dataType` field's specific allowed values (e.g., it likely does not enforce `number | boolean | string | array | object | any | undefined`). If the schema does not enumerate valid `dataType` values, an unrecognised type like `'float'` would reach `generateSetVariableWithTypeCode` and produce `as unknown as float`, which is invalid TypeScript.

---

## Code Quality Analysis

[↑ Back to top](#pr-review-trs-126--feat-add-variable-with-data-type-node)

### Strengths

[↑ Back to top](#pr-review-trs-126--feat-add-variable-with-data-type-node)

- **Follows existing patterns** — the new function mirrors `generateSetVariableCode` closely, making it easy to read and maintain.
- **Correct helper extraction** — `escapeForInterpolatedTemplate` fixes a real latent bug in the old code (missing backslash escape) while reducing duplication across three call sites.
- **Defensive empty-value handling** — returns a declaration-only statement (`let x;`) rather than generating invalid code when value is empty or type is `undefined`.
- **Correct `JSON.stringify` for fallback** — the fix in commit `8d9eb872` properly uses `JSON.stringify` instead of naïve quote-wrapping for the string fallback path. This prevents broken output for values containing embedded quotes.
- **`toTsType` isolates the `array` special case cleanly** — keeping this transformation in one place avoids the `Array<unknown>` literal being scattered across the branching logic.
- **JSDoc comments on the two utilities** are accurate and explain non-obvious invariants (backslash-first escaping order, intentional `$` passthrough).

---

### Issues and Observations

[↑ Back to top](#pr-review-trs-126--feat-add-variable-with-data-type-node)

#### Issue 1 — Object branch: cast fallback generates invalid TypeScript

**Severity: Major (Functional Correctness)**  
**Status: Partially resolved in commit `8d9eb872` — but the third fallback arm is still problematic**

The `dataType === 'object'` branch has three arms:

```typescript
} else if (dataType === 'object') {
  const trimmedObj = varValue.trim();
  if (trimmedObj.startsWith('{')) {
    valueStr = trimmedObj;                          // ✅ correct: already an object literal
  } else if (trimmedObj.includes(':')) {
    valueStr = `{${varValue}}`;                     // ✅ correct: wrap as object literal
  } else {
    valueStr = `${varValue} as unknown as object`;  // ⚠️ problematic
  }
}
```

The third arm (`as unknown as object`) is reached when `varValue` is an object-shaped value that doesn't start with `{` and contains no `:`. In practice the only inputs that reach here are either bare identifiers (variable references already handled earlier), or malformed object literals. `foo as unknown as object` is technically valid TypeScript but semantically wrong — it does not create an object, it just casts whatever `foo` is. A user passing `myObj` would get `let x = myObj as unknown as object` instead of `let x = {myObj}`.

This edge case should either be documented as intentional (a reference passthrough) or handled more explicitly. As-is it silently produces potentially unexpected output.

#### Issue 2 — `setVariableSchema` does not validate `dataType` enum

**Severity: Minor (Data Integrity)**

Reusing `setVariableSchema` is the right call for the structural fields (name, declarationType, value), but the schema likely does not enumerate valid `dataType` values. An invalid `dataType` — e.g., `'float'`, `'int'`, `'Integer'`, a typo — will pass schema validation and reach code generation, producing `as unknown as float` which is invalid TypeScript.

The fix would be to either:
1. Extend `setVariableSchema` with a `dataType` enum field, or
2. Create a `setVariableWithTypeSchema` that extends `setVariableSchema` with the `dataType` constraint.

This is a defensive improvement rather than a blocking bug, since the UI presumably only surfaces valid type options.

#### Issue 3 — `generateSetVariableWithTypeCode` vs `generateSetVariableCode` divergence

**Severity: Minor (Code Duplication / Maintainability)**

The new function substantially duplicates the logic of `generateSetVariableCode`. The branching structure, variable extraction (`params.name || params.variableName`), `isVariableReference` check, and `stripVariableIndicators` call are all repeated. The differences are:
1. `generateSetVariableWithTypeCode` appends `as unknown as <type>` casts.
2. The string fallback uses `JSON.stringify` instead of naïve quote-wrapping.

This means any future change to `generateSetVariableCode` (e.g., a new `dataType` handling path) must also be replicated in `generateSetVariableWithTypeCode`. A unified private helper that accepts a `withType: boolean` flag — or a base function that `generateSetVariableWithTypeCode` calls and post-processes — would reduce drift risk.

This is not a blocker for this PR but is worth noting as a maintenance concern.

#### Issue 4 — Default `generationMode` mismatch

**Severity: Minor (Behavioural Inconsistency)**

```typescript
const generateSetVariableCode = (
  params, indent, generationMode: 'rule-builder' | 'test-case-generate' = 'rule-builder'
): string => { ... }

const generateSetVariableWithTypeCode = (
  params, indent, mode: 'rule-builder' | 'test-case-generate' = 'test-case-generate'
): string => { ... }
```

`generateSetVariableCode` defaults to `'rule-builder'` while `generateSetVariableWithTypeCode` defaults to `'test-case-generate'`. If the dispatch in `generateNodeCode` always passes an explicit `generationMode` this has no effect, but if any caller omits it (e.g., in a test or future utility context) the two functions will behave differently by default. The default should be aligned. Additionally, `generateSetVariableCode` calls its third parameter `generationMode` while `generateSetVariableWithTypeCode` calls it `mode` — minor naming inconsistency.

#### Issue 5 — CodeQL: Incomplete string escaping

**Severity: Informational (Flagged by GitHub Advanced Security)**

GitHub Advanced Security's CodeQL scanner raised an alert on `generateSetVariableCode`:

> `CodeQL / Incomplete string escaping or encoding — This does not escape backslash characters in the input.`

This finding targets the **original** `generateSetVariableCode` function where backtick escaping occurs. The new `escapeForInterpolatedTemplate` helper used in `generateSetVariableWithTypeCode` correctly handles backslashes. However, `generateSetVariableCode` still has code paths that don't use `escapeForInterpolatedTemplate` (the non-`$` value path uses a different branch), so it is possible CodeQL is alerting on residual unsafe escaping in that function.

The alert was not resolved in this PR. It is worth confirming whether the alert is on the same line that was refactored or on a different escaping path within `generateSetVariableCode`.

---

## Security Assessment

[↑ Back to top](#pr-review-trs-126--feat-add-variable-with-data-type-node)

| Concern | Assessment |
|---------|------------|
| Code injection via `varName` in generated TypeScript | `setVariableSchema` validates `varName` — relies on schema to prevent identifiers containing `;`, `//`, etc. If schema does not enforce identifier-safe characters, a malicious name could inject arbitrary code into generated output. Not introduced by this PR; pre-existing risk. |
| Code injection via `dataType` | An unvalidated `dataType` passes through directly into `as unknown as <dataType>`. A `dataType` containing `\n` or `//` would break generated code. Downstream injection is not possible since this affects generated TS, not runtime execution. Recommended: validate `dataType` against an enum as noted in Issue 2. |
| Template literal injection via `varValue` | Correctly handled via `escapeForInterpolatedTemplate` for the `$`-containing path. |
| Security improvement — `escapeForInterpolatedTemplate` | The new helper also fixes the backslash escape gap in `generateLogCode` and `generateThrowErrorCode`. Values like `C:\path\` would previously emit `` `C:\path\` `` in a template literal where `\`` breaks the literal. This is now fixed as a side effect of this PR. |

Overall: no new security surface introduced. The CodeQL alert on `generateSetVariableCode` should be investigated to confirm it is not a residual issue with the unchanged escaping paths in that function.

---

## Test Coverage

[↑ Back to top](#pr-review-trs-126--feat-add-variable-with-data-type-node)

The PR description checklist has all items unchecked:
- [ ] Locally
- [ ] Development Environment
- [ ] Not needed, changes very basic
- [ ] Husky successfully run
- [ ] Unit tests passing and Documentation done

No checkboxes are checked. This means **no testing method is documented as having been performed**. The author attached a screenshot in the PR comments showing the UI with the new node type rendered, which provides some evidence of manual local testing, but the checklist remains unchecked.

No unit tests were added for `generateSetVariableWithTypeCode`. Given the number of distinct branching paths (11 cases), the absence of tests is a gap — particularly for the edge cases around object literal detection, boolean normalisation, and the `isVariableReference` path.

For the variable manager changes, the functions `extractVariablesFromNodes` and `isVariableDeclaredBefore` should have test cases covering `SetVariableWithType` nodes to verify the `||` / `&&` additions work correctly.

---

## CodeRabbit and Bot Review Activity

[↑ Back to top](#pr-review-trs-126--feat-add-variable-with-data-type-node)

### GitHub Advanced Security — CodeQL Finding

[↑ Back to top](#pr-review-trs-126--feat-add-variable-with-data-type-node)

**Alert:** `CodeQL / Incomplete string escaping or encoding`  
**File:** `frontend/src/utils/Flow/CodeGenerator.ts`  
**Status:** Open — not resolved by this PR

CodeQL flagged that not all code paths in `CodeGenerator.ts` escape backslash characters. This is a static analysis finding with a security classification. The `escapeForInterpolatedTemplate` function added in this PR does handle backslashes correctly, but the alert may refer to other escaping paths not updated in this change.

---

### CodeRabbit Inline Comments

[↑ Back to top](#pr-review-trs-126--feat-add-variable-with-data-type-node)

CodeRabbit ran two full review passes and was rate-limited on the third:

**Pass 1 (commit `670a4c1b` — initial feature):** 4 actionable comments:

| # | Comment | Severity | Status |
|---|---------|----------|--------|
| CR-1 | Register `SetVariableWithType` in schema registry | Major | ✅ Already done in initial commit |
| CR-2 | Expose `SetVariableWithType` to variable extraction in `VariableManager.ts` | Major | ✅ Already done in initial commit |
| CR-3 | Object branch emits invalid TS for unbraced members like `foo: 1` | Major | ✅ Partially fixed in commit `8d9eb872` |
| CR-4 | Fallback string literals not escaped before wrapping in quotes | Minor | ✅ Fixed in commit `8d9eb872` using `JSON.stringify` |

**Pass 2 (commit `ed0b6c8b`):** Rate-limited — no actionable comments produced.

The developer marked CR-3 as resolved (`✅ Addressed in commit 8d9eb87`) and replied "Fixed." to CR-4.

---

## Resolution Status of Bot-Flagged Issues

[↑ Back to top](#pr-review-trs-126--feat-add-variable-with-data-type-node)

| # | Issue | Source | Status |
|---|-------|--------|--------|
| CR-1 | `SetVariableWithType` not in schema registry | CodeRabbit | ✅ Resolved (was already present in initial commit) |
| CR-2 | `SetVariableWithType` not extracted in VariableManager | CodeRabbit | ✅ Resolved (was already present in initial commit) |
| CR-3 | Object branch: unbraced `foo: 1` emits invalid TS | CodeRabbit | ⚠️ Partially resolved — the `foo: 1` case now wraps in `{...}`. Third arm (`as unknown as object` for bare values) remains and may produce unexpected output |
| CR-4 | String fallback not escaped before quote-wrapping | CodeRabbit | ✅ Resolved — `JSON.stringify` used in commit `8d9eb872` |
| CodeQL | Incomplete string escaping / backslash not escaped | GitHub Advanced Security | ❌ Open — not addressed in this PR |
| PR description | Testing checklist unchecked | N/A | ❌ No testing method documented |

---

## Summary and Verdict

[↑ Back to top](#pr-review-trs-126--feat-add-variable-with-data-type-node)

**Verdict: Changes Requested**

The core feature is sound. The dispatch, variable tracking, and schema registration are all correctly wired up. The `escapeForInterpolatedTemplate` refactor is a genuine improvement that fixes a latent bug in three unrelated call sites. The `JSON.stringify` fix in the string fallback is correct.

However, the following items require attention before this should be merged:

### Blocking Items

1. **Unchecked testing checklist** — the PR description has no testing method marked. At a minimum, the author should confirm local testing was done and check the appropriate box. The screenshot in comments helps but does not substitute for the checklist.

2. **CodeQL alert unresolved** — the GitHub Advanced Security CodeQL alert (`Incomplete string escaping or encoding`) is open. The PR should address or explicitly acknowledge this alert. It may be a false positive given the new helper, but it needs to be confirmed and closed.

### Non-Blocking but Recommended

3. **Object branch third arm** — the fallback `${varValue} as unknown as object` in the object path is semantically ambiguous. If this case is intentionally a reference passthrough, add an inline comment explaining it. If it is not intentional, handle it more explicitly (e.g., wrap in `{...}` as the `:` arm does).

4. **`dataType` validation in schema** — `setVariableSchema` should either enumerate valid `dataType` values or a new `setVariableWithTypeSchema` should be created. An invalid `dataType` currently reaches generated code unchecked.

5. **Default `mode` parameter alignment** — `generateSetVariableWithTypeCode` defaults to `'test-case-generate'` while `generateSetVariableCode` defaults to `'rule-builder'`. These should be aligned.

6. **No unit tests** — `generateSetVariableWithTypeCode` has 11 distinct branching paths. Unit tests covering the main cases (variable reference, empty value, each type branch) would provide confidence and prevent regressions from future refactors.

---

## GitHub Review Comment

The block below is ready to paste directly as a **Changes Requested** review comment on GitHub.

````markdown
**Changes Requested**

Good work on the core wiring — dispatch, `VariableManager` recognition, and schema registration are all correctly in place, and the `escapeForInterpolatedTemplate` extraction is a solid improvement that fixes a real backslash-escape bug across three existing call sites. A few things to address before this is ready to merge.

---

### Blocking

**1. Check the testing checklist**
The PR description checkboxes are all unchecked. Please go back and tick whichever ones apply — at minimum "Locally" given the screenshot you shared. This is required before merge.

**2. Resolve the open CodeQL alert**
There is an open GitHub Advanced Security alert on `CodeGenerator.ts`: `Incomplete string escaping or encoding`. Open the Security tab, find alert #49, and check which exact line it points to. The `escapeForInterpolatedTemplate` helper you added correctly escapes backslashes, but `generateSetVariableCode` still has a code path that does not go through it — the branch that handles values without `$` uses a plain `.replace(/'/g, "\\'")` with no backslash handling. If that is what CodeQL is pointing at, apply `escapeForInterpolatedTemplate` there too or open a separate fix. Either way, the alert must be dismissed or resolved before this merges — do not leave it open.

---

### Non-blocking (please address in this PR if possible)

**3. Object branch third arm will cast instead of construct**

In `generateSetVariableWithTypeCode`, the `dataType === 'object'` block has this fallback:

```typescript
} else {
  valueStr = `${varValue} as unknown as object`;
}
```

This arm is reached when the value has no `{` and no `:` — for example a bare word like `myVar`. The output `let x = myVar as unknown as object` does not construct an object, it casts a reference. That is almost certainly not what the user intends when they pick `object` as the type. Change this arm to wrap in braces, consistent with the `:` arm directly above it:

```typescript
} else {
  valueStr = `{${varValue}}`;
}
```

If there is a deliberate reason to cast rather than wrap here, add a comment in the code explaining it.

**4. `dataType` field is unvalidated — add an enum to the schema**

`SetVariableWithType` is registered against `setVariableSchema`, which validates `name`, `declarationType`, and `value` but does not constrain `dataType`. A value like `'float'` or `'Integer'` will pass validation and generate `as unknown as float`, which is invalid TypeScript and will break compilation silently. Add a `dataType` field to `setVariableSchema` (or create a `setVariableWithTypeSchema` that extends it) with an explicit allowlist:

```typescript
dataType: Joi.string()
  .valid('string', 'number', 'boolean', 'array', 'object', 'any', 'undefined')
  .required()
```

This keeps the schema the source of truth for what types the node supports and gives users a validation error at the UI boundary rather than a broken codegen output.

**5. Align the default `mode` parameter**

`generateSetVariableCode` declares its third parameter as `generationMode` with a default of `'rule-builder'`. `generateSetVariableWithTypeCode` declares it as `mode` with a default of `'test-case-generate'`. Change the new function to match:

```typescript
// change this:
const generateSetVariableWithTypeCode = (params, indent, mode = 'test-case-generate') => ...

// to this:
const generateSetVariableWithTypeCode = (params, indent, generationMode = 'rule-builder') => ...
```

Also rename the parameter from `mode` to `generationMode` throughout the function body so both functions are consistent and callers can treat them interchangeably.
````
