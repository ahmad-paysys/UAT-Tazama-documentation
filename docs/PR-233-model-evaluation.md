# Model Evaluation: PR Review Task (CMS #233)

**Date:** 2026-07-07
**Task:** Review [tazama-lf/case-management-system PR #233](https://github.com/tazama-lf/case-management-system/pull/233) — a large refactor (+2,709 / -2,212 across 71 files) — and produce a written review file, given the same prompt and `pull-requests.md` guidance.
**Harness:** VS Code + GitHub Copilot Chat, same tools available (Read, gh CLI, file writes).

## Result Summary

| Model | Outcome | Time |
|---|---|---|
| Claude | Completed successfully — 567-line structured review delivered | ~8 minutes |
| DeepSeek | Failed to produce output | >30 minutes before abandoned |
| GLM | Failed to produce output | Repeated errors, never recovered |

## Observations

**Claude** produced a complete review covering schema/migrations, backfill script, service wiring, alert repository, case lifecycle, and closure flows, with a table of contents and inline references — matching the format implied by `pull-requests.md`.

**DeepSeek** ([CMS-PR-233-deepseek.md](https://github.com/ahmad-paysys/UAT-Tazama-documentation/blob/main/pull-requests/CMS-PR-233-deepseek.md)) executed the reconnaissance steps — read `pull-requests.md`, ran `gh pr view`, `gh pr diff`, fetched reviews and comments — and then stalled at the write step. The transcript shows five separate attempts to invoke the file-write tool, each failing with "I need to provide both filePath and content parameters" style errors. The model could not correctly form the tool call for `create_file` and looped on retries until abandoned. No review content was ever generated.

**GLM** ([CMS-PR-233-GLM.md](https://github.com/ahmad-paysys/UAT-Tazama-documentation/blob/main/pull-requests/CMS-PR-233-GLM.md)) got further into gathering context (wrote the diff to `/tmp/pr233-diff.txt`, fetched reviews and comments) but then entered a "Recovered from a request error" loop while re-reading the diff file. Six retry cycles, all failed, no write attempted. The underlying error surfaced by Copilot was:

```
Reason: {"message":"litellm.APIError: Error building chunks for logging/streaming usage calculation","type":null,"param":null,"code":"500"}
```

This is a **500-level failure inside LiteLLM** (the proxy layer Copilot routes non-OpenAI models through), not inside GLM's own inference. LiteLLM streams the model's response back to the client in chunks and, in parallel, aggregates those chunks to compute token-usage totals for billing/logging. When the aggregator crashes mid-stream — typically triggered by an unusually large or malformed chunk sequence (which a 3,000-line diff read easily produces) — the whole request is torn down and returned to the client as a generic 500. GLM's model likely produced valid tokens; the proxy's bookkeeping is what failed. Retries hit the same code path and fail identically, which matches the observed loop.

## Takeaway

For this class of task — long-context reasoning over a large diff followed by structured long-form output — Claude was the only model in the trial that completed. The other two failed on different axes: DeepSeek on tool-call construction, GLM on request-error recovery during large-file reads.
