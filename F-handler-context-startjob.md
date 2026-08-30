---
id: F-handler-context-startjob
title: "FHandlerContext::StartJob — one-call async job dispatch helper"
status: DONE
severity: Medium
category: feature
tags: [async, jobs, handler-context, infrastructure, api]
---

# FHandlerContext::StartJob — one-call async job dispatch helper

Migrating a handler from synchronous to async required boilerplate: call `FJobRegistry::RegisterJob`, send the initial `{jobId, started:true}` response via `Ctx.SendSuccess`, then bind the completion delegate separately. Three repeated steps, each requiring direct access to `FPluginState`.

`Ctx.StartJob()` combines all three: it registers the job in `FJobRegistry`, immediately sends `{jobId, started:true}` through the HTTP response slot (closing the request), and returns the `jobId` string for use in the callback closure. Each migrated handler now uses a single `FString JobId = Ctx.StartJob()` call followed by an `OnDelegate.BindLambda([JobId](...){ FJobRegistry::Get().CompleteJob(JobId, ...); })`.

**Files:** `Source/PinWright/Private/Handlers/HandlerContext.h`, `Private/Handlers/HandlerContext.cpp`.

## History
- `#1-async-migration-boilerplate` `OPEN` reporter — Migrating a handler to async requires three repeated steps (RegisterJob, SendSuccess, bind delegate) each needing direct `FPluginState` access; need a single-call helper on `FHandlerContext` to eliminate the boilerplate.
- `#2-startjob-helper-added` `IN-REVIEW` developer — `StartJob(OptionalInitialPayload)` added to `FHandlerContext`. Calls `FPluginState::Get().GetJobRegistry().RegisterJob(GetMethod())`, emits `{jobId, started:true}` via `SendSuccess`, returns the assigned `jobId`. All 20 migrated handlers use this pattern.
- `#3-verified-startjob-shape` `DONE` tester — Verified: every migrated handler exercised this session (`editor.save_all`, `editor.screenshot`, `level.save`, `level.build_navigation`, `level.build_lighting`, `level.build_level_lighting`, `level.build_all`, `lighting.build_lighting`, `navigation.rebuild_navigation`, `performance.optimize_shaders`, `performance.run_benchmark`, `blueprint.build_api_index`, `editor.open_level`, `asset.dump_folder`, `system.run_tests`) returned the canonical kickoff JSON `{status:"running", ticket_id:"j_<ts>_<hex>", monitor_path:".editor-automation/jobs.jsonl", method:<name>, started_at:<iso>}` plus per-method context fields. StartJob shape consistent across all 15 verified handlers.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place; every rewritten path was confirmed to exist at plugin HEAD `ef8a1f1b`. Path(s) here that move by more than the prefix in this ticket, taken from the plugin's rename history rather than the prefix rule: `Source/EditorAutomationRpcGateway/Public/Handlers/HandlerContext.h` → `Source/PinWright/Private/Handlers/HandlerContext.h`. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
