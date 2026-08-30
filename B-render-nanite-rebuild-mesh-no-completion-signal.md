---
id: B-render-nanite-rebuild-mesh-no-completion-signal
title: "render.nanite_rebuild_mesh returned {started:true} with no completion signal"
status: DONE
severity: Medium
category: bug
tags: [async, jobs, render, nanite, no-completion-signal]
---

# render.nanite_rebuild_mesh returned {started:true} with no completion signal

`render.nanite_rebuild_mesh` (also exposed as `asset.nanite_rebuild_mesh`) triggered Nanite mesh streaming data rebuild and returned `{started:true}`. Nanite rebuilds run on background threads; callers had no way to know when the static mesh was ready for use.

**Fix:** Handler now calls `Ctx.StartJob()` and wraps the Nanite build call in an `AsyncTask` that awaits the background task completion before calling `FJobRegistry::CompleteJob(JobId, true, "Nanite rebuild complete")`.

**Files:** `Source/PinWright/Private/Handlers/Render/RenderHandler.cpp:1869`.

## History
- `#1-no-completion-signal` `OPEN` reporter — Nanite rebuild started with no completion event. Subsequent rendering queries on the mesh could return stale data.
- `#2-asynctask-wrap` `IN-REVIEW` developer — Migrated to `Ctx.StartJob()`. `AsyncTask` wraps the Nanite rebuild; `CompleteJob` fires on task return.
- `#3-skip-mutates-mesh` `SKIP` tester — Live test would enable Nanite on a real static mesh asset (mutating project state). Schema verified registered with required `assetPath` param. The `Ctx.StartJob()` + `AsyncTask` pattern is the same shape as other migrated handlers verified end-to-end this session (e.g. `level.save`, `editor.save_all`, `blueprint.build_api_index`).
- `#4-accepted-without-recheck` `DONE` tester — Accepted by user decision without further live verification; prior SKIP entry documents why the static-mesh mutation test was not re-run.
- `#5-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place and verified against plugin HEAD `ef8a1f1b`. `NaniteRebuildMeshHandler.cpp` never existed; the verb is `Handlers/Render/RenderHandler.cpp:1869`, job at `:1922`, completion at `:1918`, failure at `:1895`. The body's “also exposed as `asset.nanite_rebuild_mesh`” is no longer an alias — that is now a separate, richer registration at `Handlers/Asset/AssetWorkflowHandler.cpp:1249`. Sweep-wide record, including every case that could not be repointed: `E-module-rename-citation-sweep`.
