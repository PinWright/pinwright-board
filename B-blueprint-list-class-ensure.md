---
id: B-blueprint-list-class-ensure
title: "`blueprint.list` fires `FTopLevelAssetPath` ensure on short class names"
status: DONE
severity: Medium
category: bug
tags: [blueprint-list, class-filter, ensure]
---

# `blueprint.list` fires `FTopLevelAssetPath` ensure on short class names

Calling `blueprint.list` with a short class name (e.g. `class: "DroneRacingTrack"`) fires `Ensure condition failed: false` at `CoreUObject/Private/UObject/TopLevelAssetPath.cpp:141` with message
`Short asset name used to create FTopLevelAssetPath: "<ClassName>"`.

Same root cause as the already-fixed `B-asset-list-short-class-ensure`, but in a different handler (`EditorAutomationRpcGateway_BlueprintHandlers_List.cpp:159`). That file was missed in the prior sweep.

Stack observed:
```
FTopLevelAssetPath::TrySetPath          TopLevelAssetPath.cpp:141
FTopLevelAssetPath::FTopLevelAssetPath  TopLevelAssetPath.h:54
AutoHandler_22_                         EditorAutomationRpcGateway_BlueprintHandlers_List.cpp:159
FRpcDispatcher::ProcessRequest          RpcDispatcher.cpp:172
```

Repro: `blueprint.list` with `class="DroneRacingTrack"`. Call succeeds (returns empty `assets[]`) but PDS.log emits the engine ensure.

**Workaround:** use the full class path, e.g. `class="/Script/App.DroneRacingTrack"`.

**Fix:** apply the same short-name resolution as `B-asset-list-short-class-ensure`. Route through `Utils/ClassUtils.h`'s `ResolveUClass` (or equivalent) before constructing `FTopLevelAssetPath` at line 159 of `EditorAutomationRpcGateway_BlueprintHandlers_List.cpp`. The original fix for `asset.list` lives near `AssetManageHandler.cpp:718` — mirror that pattern.

## History
- `#1-initial-repro` `OPEN` reporter — Surfaced while using MCP to author BPs for the replay editor; called `blueprint.list` with `class="DroneRacingTrack"` to find the racing-track BP. Got `Ensure condition failed: false` at `TopLevelAssetPath.cpp:141` with the "Short asset name used to create FTopLevelAssetPath" message. Same bug class as `B-asset-list-short-class-ensure` (marked DONE), but the prior fix only covered `asset.list`; `blueprint.list` still constructs `FTopLevelAssetPath` directly from a short name.
- `#2-resolve-uclass-fix` `IN-REVIEW` developer — `blueprint.list` now routes short class names through `ResolveUClass` before constructing `FTopLevelAssetPath`, so the ensure no longer fires. Changed `EditorAutomationRpcGateway_BlueprintHandlers_List.cpp:152-174` (UE 5.1+ branch). Covered by `FBlueprintListShortClassNameNoEnsureTest` in `TestBlueprintHandlers.cpp`.
- `#3-verified-no-ensure` `DONE` tester — Verified: called `blueprint.list` with `class="DroneRacingTrack"` (short name). Response returned `success: true` with 1 entry (`B_Race_C_1`). Grepped `Saved/Logs/PDS.log` for "Short asset name used to create FTopLevelAssetPath" — zero matches. The ensure no longer fires and the short-name resolution works end-to-end.
