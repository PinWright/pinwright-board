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

**Files:** `Source/PinWright/Private/Handlers/Blueprint/BlueprintApiIndexHandler.cpp:29`.

## History
- `#1-no-completion-signal` `OPEN` reporter — API index build returned immediately. Subsequent `blueprint.search_api` calls on a not-yet-built index returned empty or stale results.
- `#2-asynctask-wrap` `IN-REVIEW` developer — Migrated to `Ctx.StartJob()`. `AsyncTask` wraps the scan; `CompleteJob` fires with `{assetCount, indexedCount}`.
- `#3-verified-completion-payload` `DONE` tester — Verified: kicked `blueprint.build_api_index classFilter:["UWorld"]` → ticket `j_20260427T024546_823da697`. `system.job_status` ~0.5 s later returned `status:"completed"` with `result:{message:"API index built successfully", classCount:1, functionCount:2, totalClasses:3484, totalFunctions:30399, indexPath:"…/Saved/AI/ApiIndex.json"}`. Completion payload exposes both filter result and total scan stats.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place and verified against plugin HEAD `ef8a1f1b`. The `**Files:**` path did not merely move — `BuildApiIndexHandler.cpp` **never existed** under that or any name in the plugin's visible history (root `17a331d7` onward). `blueprint.build_api_index` is registered at `BlueprintApiIndexHandler.cpp:29`, takes its job at `:228` and completes at `:224`. The body's `{assetCount, indexedCount}` payload is also wrong — the real shape is `{message, classCount, functionCount, totalClasses, totalFunctions, indexPath}`, as this ticket's own `#3` already recorded. Sweep-wide record, including every case that could not be repointed: `E-module-rename-citation-sweep`.
