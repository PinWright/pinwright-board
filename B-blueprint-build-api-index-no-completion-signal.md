---
id: B-blueprint-build-api-index-no-completion-signal
title: "blueprint.build_api_index returned {started:true} with no completion signal"
status: DONE
severity: Medium
category: bug
tags: [async, jobs, blueprint, api-index, no-completion-signal]
---

# blueprint.build_api_index returned {started:true} with no completion signal

`blueprint.build_api_index` triggered a full blueprint API scan (used to populate the search index for `blueprint.search_api`) and returned `{started:true}`. The scan walks all blueprint assets in the registry and can take many seconds on large projects; callers had no signal when the index was ready to query.

**Fix:** Handler now calls `Ctx.StartJob()` and wraps the index-build task in an `AsyncTask`. On completion, `FJobRegistry::CompleteJob` is called with `{assetCount, indexedCount}` so callers know the scan is done before issuing `blueprint.search_api` queries.

**Files:** `Source/EditorAutomationRpcGateway/Private/Handlers/Blueprint/BuildApiIndexHandler.cpp`.

## History
- `#1-no-completion-signal` `OPEN` reporter — API index build returned immediately. Subsequent `blueprint.search_api` calls on a not-yet-built index returned empty or stale results.
- `#2-asynctask-wrap` `IN-REVIEW` developer — Migrated to `Ctx.StartJob()`. `AsyncTask` wraps the scan; `CompleteJob` fires with `{assetCount, indexedCount}`.
- `#3-verified-completion-payload` `DONE` tester — Verified: kicked `blueprint.build_api_index classFilter:["UWorld"]` → ticket `j_20260427T024546_823da697`. `system.job_status` ~0.5 s later returned `status:"completed"` with `result:{message:"API index built successfully", classCount:1, functionCount:2, totalClasses:3484, totalFunctions:30399, indexPath:"…/Saved/AI/ApiIndex.json"}`. Completion payload exposes both filter result and total scan stats.
