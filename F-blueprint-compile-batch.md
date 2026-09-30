---
id: F-blueprint-compile-batch
title: "No batch Blueprint compile: blueprint.compile takes one path, so a post-refactor sweep costs N calls and the errored-Blueprint scan stays private"
status: OPEN
severity: Medium
category: feature
tags: [blueprint, compile, batch, diagnostics, gap-analysis-2026-09-30]
encounters: 1
---

# No batch Blueprint compile

`blueprint.compile` takes a single `path` (`Source/PinWright/Private/Handlers/Blueprint/BlueprintCompileHandler.cpp:17-60`). `blueprintCandidates`/`candidates` are for path resolution, not batching. The shared helper `CompileBlueprintWithDiagnostics` (`BlueprintHandlerUtils.cpp:465-536`) returns `errors[{message}]` with no node GUID or graph, has no warnings-as-errors, and runs a GC on every call. A loaded `BS_Error` scan already exists, `CollectBlueprintsWithErrors` (`PIEHandler.cpp:138-151`), but it only surfaces as `editor.play`'s `blueprintsWithErrors`. pinwright.com/compare rates "Compile / batch compile" partial. ue-mcp `compile_all` (`BlueprintHandlers_Graph.cpp:2616`) is a thin yes: an explicit list, counts only, saves by default, no reinstancing guard. Epic compiles one Blueprint with per-node `ErrorMsg` and `warnings_as_errors`.

**Fix:** a new verb `blueprint.compile_batch` in `BlueprintCompileHandler.cpp`.
- Params: `assets[]` or `folder` (exactly one), `recursive`, `namePattern`, `onlyStatus: all|error|dirty` (`error` covers the errored-Blueprint listing, so no separate verb), `allowReinstancing`, `warningsAsErrors`, `limit`/`offset`. Conventions follow `geometry.audit_static_meshes` (`MeshAuditHandler.cpp:372`, `NO_ASSETS_MATCHED`).
- Runs as `Ctx.StartJob` with progress per asset (rpc-design §9a).
- Row: `{path, statusBefore, status, errors[{message, nodeGuid?, graph?}], warnings, reinstanced}`. `nodeGuid` comes from the message's `FUObjectToken`. Add it and `warningsAsErrors` to the shared helper so single `blueprint.compile` gains them too.
- A live-instance refusal (`LIVE_INSTANCES_WOULD_BE_REINSTANCED`) is a row status, not a call error. Totals `compiled + failed + refused + unloadable` must equal `matched`.
- One GC at the end. No save; the caller uses `asset.save`.
- Model the loop on `CompileAllBlueprintsCommandlet.cpp`: `LOAD_DisableCompileOnLoad` at `:153`, a fresh `FCompilerResultsLog` per Blueprint at `:317`, `TrimMemory` at `:164`. Do not use `QueueForCompilation`: it has no per-Blueprint log.
- UE APIs: `FKismetEditorUtilities::CompileBlueprint` (`KismetEditorUtilities.h:169`), `EBlueprintCompileOptions` (`:33-58`), `FCompilerResultsLog` (`CompilerResultsLog.h:106/121/130`). `UBlueprint::Status` is transient (`Blueprint.h:508`). No 5.3-5.7 drift seen.

**Acceptance:**
- A folder with one broken, one clean and one live-instanced Blueprint returns three rows with the correct statuses. Totals sum, and no package is saved.
- Progress frames arrive per asset.
- An empty match returns `NO_ASSETS_MATCHED`.
- Error rows carry `nodeGuid` for node-level errors.

Effort M. Risk medium: mass reinstancing and long runs, bounded by the guard and `limit`.

## History
- `#1-gap-analysis` `OPEN` reporter — Filed from the 2026-09-30 competitor gap analysis (compare row "Compile / batch compile": PinWright partial, ue-mcp/Epic yes). Single-path compile only; diagnostics carry no node location; the errored-Blueprint scan is private. Evidence and design above.
