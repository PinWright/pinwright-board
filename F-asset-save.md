---
id: F-asset-save
title: "Add a generic single-asset save RPC (asset.save {assetPath})"
status: IN-REVIEW
severity: Medium
category: feature
tags: [asset, save, blueprint, persistence]
---

# Add a generic single-asset save RPC (`asset.save {assetPath}`)

There is no targeted way to persist ONE arbitrary asset over MCP. The only
save-registered RPCs are `editor.save_all` / `ui.save_all` (every dirty
package — blunt), `level.save` / `level.save_as` (levels), and `niagara.save`.
Graph/property mutators (`blueprint.graph.set_pin_default_values`,
`blueprint.set_default`, etc.) leave the asset dirty in memory with no per-asset
flush. The only targeted single-Blueprint save today is
`blueprint.compile {saveAfterCompile:true}`, which couples persistence to a
recompile.

Add `asset.save {assetPath}` that saves one loaded asset to disk.

**Persistence must be honest, not throttle-trusting.** Do NOT route through raw
`SaveLoadedAssetThrottled` (`Utils/AssetUtils.cpp:552`): it returns `true` on a
throttle-skip / transient / deferred no-write, which is exactly the false
`saved:true, sizeBytes:0` defect `B-niagara-save-no-disk-write` fixed. Route
through the established disk-presence wrapper `SaveAssetToDiskReportingPresence`
(`Utils/AssetUtils.cpp:387`) + `ShouldTreatAssetSaveAsSuccess`
(`Utils/AssetUtils.cpp:377`), which probe `IFileManager::FileSize` and only
report `saved:true` when the `.uasset` actually landed on disk — emitting
`package` / `sizeBytes` and `pendingFlush:true` otherwise, mirroring how
`niagara.save` already does it (`NiagaraCompileHandler.cpp:140`).

`SaveAssetToDiskReportingPresence` inherits the Blueprint integrity gate inside
`SaveLoadedAssetThrottled` (refuse-to-write on `ValidateBlueprintGraphIntegrity`
failure, `Utils/AssetUtils.cpp:588-602`). But that helper returns a bare `false`
for BOTH a gate block AND a genuine save failure and exposes no failure list, so
to **surface the verdict inline** (`integrityGate:"blocked"`, `integrityFailures`)
the handler must call `ValidateBlueprintGraphIntegrity` itself for `UBlueprint`
assets — exactly as `blueprint.compile` does (`BlueprintCompileHandler.cpp:51,66-78`).
That inline reporting currently lives only in the compile handler and is lost on
every other save path.

Decision (validate-on-save): the integrity gate stays on the save path, not
compile — save is the universal choke point that every mutation source flows
through. Paired with `E-blueprint-compile-compile-only` (removal of
`saveAfterCompile`); land this first so no targeted single-BP save is lost.

**Workaround:** `blueprint.compile {saveAfterCompile:true}` for BPs, or
`editor.save_all` (saves everything dirty).
**Fix:** New `asset.save {assetPath, force?}` handler under `Handlers/Asset/`;
`LoadObject<UObject>` by path → for `UBlueprint` run
`ValidateBlueprintGraphIntegrity` and on failure short-circuit with
`integrityGate:"blocked"` + `integrityFailures` (saved:false) → otherwise call
`SaveAssetToDiskReportingPresence(Asset, force, &package, &sizeBytes)` and return
`saved` (disk-gated) + `package` + `sizeBytes` + `pendingFlush` on a no-write.
`force` bypasses the save throttle (mark dirty + force).

## History
- `#1-no-targeted-asset-save` `OPEN` reporter — Verified by registration grep (`REGISTER_RPC_HANDLER("...save...")`): only `editor.save_all`, `ui.save_all`, `level.save`, `level.save_as`, `niagara.save` exist — no `asset.save` / `blueprint.save`. Mutators like `blueprint.graph.set_pin_default_values` don't save, so persisting a single edited Blueprint requires `blueprint.compile {saveAfterCompile:true}` or a blunt `editor.save_all`. Filed alongside `E-blueprint-compile-compile-only`.
- `#2-reword-disk-presence-route` `OPEN` developer — Reworded the **Fix:** off raw `SaveLoadedAssetThrottled` (which reports `true` on throttle-skip/transient/deferred no-write — the `B-niagara-save-no-disk-write` false-`saved` defect) onto the established disk-presence wrapper `SaveAssetToDiskReportingPresence` + `ShouldTreatAssetSaveAsSuccess` with `pendingFlush`, mirroring `niagara.save`; specified that the inline `integrityGate`/`integrityFailures` verdict requires a handler-level `ValidateBlueprintGraphIntegrity` call (as `blueprint.compile` does), not the helper's bare boolean; refreshed stale line cites (`SaveLoadedAssetThrottled` `:552`, integrity gate `:588-602`, wrapper `:387`/predicate `:377`).
- `#3-implement-asset-save` `IN-REVIEW` developer — Implemented `asset.save {assetPath, force?}` per the reworded Fix. New handler `Source/EditorAutomationRpcGateway/Private/Handlers/Asset/AssetSaveHandler.cpp`: `LoadObject<UObject>` by path (ASSET_NOT_FOUND on miss), for `UBlueprint` runs `ValidateBlueprintGraphIntegrity` and on failure short-circuits the save with `integrityGate:"blocked"` + `integrityFailures` (saved:false), otherwise routes through `SaveAssetToDiskReportingPresence` (disk-gated `saved` + `package` + `sizeBytes`, `pendingFlush` on a no-write); `force` marks dirty + bypasses the throttle. Regression test `Source/EditorAutomationRpcGateway/Private/Tests/Assets/TestAssetSaveHandler.cpp` (`EditorAutomationRpcGateway.asset.save`): asserts registration+category, INVALID_PARAMS on missing assetPath, ASSET_NOT_FOUND on a bad path, and a live integrity-gate path (real BP + planted stale `UK2Node_CreateDelegate`) returning `integrityGate:"blocked"` + non-empty `integrityFailures` + `saved:false` — would fail if the handler were missing or reverted to a route that drops the inline verdict. Not compiled/tested here (later phase).
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 2 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
