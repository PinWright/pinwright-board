---
id: B-asset-dump-properties-skipped-on-stub-class
title: "asset.dump skips properties.json when generated class fails to load (writes meta+bpir but no properties)"
status: DONE
severity: Medium
category: bug
tags: [asset-dump, properties, blueprint]
---

# `asset.dump` skips `properties.json` when generated class fails to load

For exactly one Blueprint in a 30,335-asset sweep, `asset.dump` wrote
`meta.json` and `bpir.txt` but did not produce a `properties.json` at
all. The `meta.json` for that asset also has empty `parentClass` and
`className` set to the raw `"Blueprint"` instead of the resolved
`{Name}_C`. Every other Blueprint in the same folder, dumped in the
same sweep, produced the resolved `_C` class name and a populated
`parentClass`.

The pattern strongly suggests `GetGeneratedClass()` returned null for
this asset (so the meta writer fell back to raw type strings and
`parentClass` was left empty), and the properties branch then bailed
out silently rather than writing an empty `{}` placeholder. Result:
inconsistent shape across the cache — one folder has `bpir.txt` but
no `properties.json`, breaking any downstream tool that assumes both
exist for every dumped Blueprint.

**Repro:**
1. Run `asset.dump_folder("/App/App/LevelBlueprints", { "recursive": true })`.
2. Inspect `.editor-automation/asset-dumps/App/App/LevelBlueprints/B_SearchMode_Mine/`.
3. Observe: `meta.json` (`className: "Blueprint"`, `parentClass: ""`) +
   `bpir.txt`, but no `properties.json`.

**Workaround:** Re-run `asset.dump` directly on the asset; if the class
load now succeeds, the dump fills in. Otherwise the asset is
silently broken in the cache.

**Fix (proposed):** In the per-asset dump dispatch, ensure the
properties writer always emits `properties.json` — either with the CDO's
properties when the class loads, or with `{}` plus a sidecar
`aspect_diagnostic.json` recording the load failure. Never leave a
folder with only meta+bpir; keeps cache shape predictable.

## History
- `#1-initial-repro` `OPEN` reporter — `asset.dump_folder("/App")` produced `B_SearchMode_Mine/` with `meta.json` (className="Blueprint", parentClass="") + `bpir.txt` but no `properties.json`. Indicates generated-class load failure causes the properties branch to silently skip rather than emit `{}` + diagnostic. Sole occurrence in a 30,335-asset sweep.
- `#2-always-emit-properties-placeholder` `IN-REVIEW` developer — Extracted `BuildBlueprintPropertiesAspect_Internal` helper in `Source/EditorAutomationRpcGateway/Private/Handlers/Asset/AssetDumpHandler.cpp` (mirrors `BuildWidgetTreeAspect_Internal`) and exposed it via `Source/EditorAutomationRpcGateway/Private/Test/AssetDumpHandlerInternal.h` with `EDITORAUTOMATIONRPCGATEWAY_API`. Both BP and WBP branches of `BuildAllFilesForAsset` now route through the helper, which always emits a `properties.json` entry — real CDO diff when `GeneratedClass` loads, an empty `{}` placeholder plus a `RecordAspectDiagnostic` entry under `DumpFileNames::Properties` when it does not. No new `aspect_diagnostic.json` channel introduced; reuses the existing `OutFileErrors` → RPC `skipped[]` surface. Added regression test `FAssetDumpBuilderPropertiesAlwaysEmittedTest` in `Source/EditorAutomationRpcGatewayTests/Private/Utility/TestAssetDumpBuilder.cpp` covering both `UBlueprint` and `UWidgetBlueprint` stub-class cases.
- `#3-verify-always-emit-properties-placeholder` `DONE` tester — Verified: `asset.dump` on `/App/App/LevelBlueprints/B_SearchMode_Mine.B_SearchMode_Mine` now returns Files=[meta.json, properties.json, bpir.txt] with Skipped=[properties.json — "Blueprint has no GeneratedClass; CDO unavailable."]. On-disk `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/App/App/LevelBlueprints/B_SearchMode_Mine/properties.json` exists as the `{}` placeholder. Cache shape is now consistent across all dumped Blueprints.
