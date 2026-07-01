---
id: B-asset-dump-bpir-stub-on-graphless-classes
title: "asset.dump writes bpir.txt stubs for BP classes that structurally cannot have an event graph"
status: DONE
severity: Low
category: bug
tags: [asset-dump, bpir, sidecar, noise, style-classes]
---

# `asset.dump` writes `bpir.txt` stubs for structurally-graphless BP classes

After the `B-bpir-stub-no-empty-marker` fix (DONE) `BuildBpirText`
(`Source/EditorAutomationRpcGateway/Private/Utils/AssetDumpBuilder.cpp:97-170`)
emits a header + `# (empty: no event/function bodies)` marker whenever a
Blueprint's auto-created EventGraph has zero nodes. That fix correctly
disambiguates "decompile succeeded with empty body" from "decompile
failed silently" for **real Blueprint subclasses that can have an event
graph but happen to be empty**.

The fix is too broad, though: it also emits the same stub for Blueprint
assets whose parent class is a **pure-data UObject** (style assets,
SaveGame, etc.). Those subclasses never have user-authored event logic
— the auto-created `EventGraph` page is a UE editor artifact and is
guaranteed to be empty for every asset of that family. Writing a 79-byte
sidecar for each of them adds no signal: there is nothing to decompile,
no failure mode to disambiguate, just per-asset noise in the cache.

**Counts (global sweep, `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/`):**

- `find . -name "bpir.txt"` → 2224 total
- `find . -name "bpir.txt" -size 79c` → 500 stub files (22%)
- In the UI subtree alone (`App/App/UI`): 492 / 829 = 59% stubs.

**Stub `parentClass` tally (500 files):**

| count | parentClass | structurally-graphless? |
|------:|-------------|:-----------------------:|
| 349 | `/Script/CommonUI.CommonTextStyle` | yes |
| 127 | `/Script/CommonUI.CommonButtonStyle` | yes |
|   6 | `/Script/Engine.SaveGame` | yes |
|   3 | `/Script/UMG.UserWidget` | no (UMG BPs can have event graphs) |
|   2 | `/Script/CoreUObject.Object` | yes (raw UObject data class) |
|   1 | `/Script/CommonUI.CommonTextScrollStyle` | yes |
|   1 | `/Script/Engine.CameraShakeBase` | yes |
|   1 | `/Script/Engine.ActorComponent` | no (components can have event graphs) |
|   1 | `/Script/PDSGame.LyraHUDLayout` | no |
|   1 | `/Script/PDSGame.LyraButtonBase` | no |
|   1 | `/Script/PDSGame.LyraActivatableWidget` | no |
|   1 | `/Script/App.WebUIHUDLayout` | no |
|   1 | `/Script/App.TrackLoaderUtilWidget` | no |
|   1 | `/Script/App.CircularMinimapWidget` | no |
|   ~ | derived style/data classes | yes |

Roughly 480 of the 500 stubs (~96%) are on classes that can never have
an event graph; ~20 are on real BP types (UserWidget, ActorComponent,
HUD layouts, etc.) that happen to be empty in this project but could
legitimately get user code added later — those stubs are the intended
beneficiaries of `B-bpir-stub-no-empty-marker`'s marker.

**Repro:**
1. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/App/App/UI/LobbyAndMenu/Editor/ButtonStyle-LeftBarButton/bpir.txt`
   (79 bytes) — `parentClass: "/Script/CommonUI.CommonButtonStyle"`.
2. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/App/App/UI/B_AxisListData/bpir.txt`
   (79 bytes) — `parentClass: "/Script/CoreUObject.Object"`.
3. Both contain only:
   ```
   # ==== Graph: EventGraph (ubergraph) ====
   # (empty: no event/function bodies)

   ```
4. Compare to a real BP with an empty graph, e.g.
   `App/App/UI/HUD/W_RaceHUD/...` (UserWidget subclass) — same stub,
   but here it serves the documented "yes, dump succeeded" purpose.

**Distinction to preserve:**

- **Elide** `bpir.txt` entirely when the Blueprint's effective parent
  class is a structurally-graphless UClass — i.e. NOT derived from
  `AActor`, `UActorComponent`, `UUserWidget`, `UBlueprintFunctionLibrary`,
  `UAnimInstance`, `UGameInstance`, `UGameMode`, `UPlayerController`, or
  any other type the editor authors event graphs onto. Style classes
  (`UCommonTextStyle`, `UCommonButtonStyle`, ...), `USaveGame`, plain
  `UObject` data classes fall here.
- **Keep** the existing stub-with-marker for real graph-bearing classes
  whose EventGraph happens to be empty — the marker is the success
  signal that disambiguates a successful empty decompile from a silent
  failure (the original `B-bpir-stub-no-empty-marker` rationale).

**Workaround:** Cache consumers that walk `bpir.txt` can filter on
`meta.json.parentClass` to ignore style-class stubs. Filename-based
detection (`-size 79c`) is brittle since the marker text could change.

**Fix (proposed):** In `Handlers/Asset/AssetDumpHandler.cpp` BP/WBP/ABP
branches (lines 480, 536, 544; level BPs at 610 are graph-bearing by
definition), before calling `AssetDumpBuilder::BuildBpirText`, probe the
Blueprint's `ParentClass`:

- If `ParentClass` derives from a known graph-bearing root
  (`AActor`, `UActorComponent`, `UUserWidget`, `UBlueprintFunctionLibrary`,
  `UAnimInstance`, `UGameInstance`, `UGameMode`, `UPlayerController`,
  `ULevelScriptActor`, ...): keep current behavior (always emit `bpir.txt`,
  marker covers empty case).
- Otherwise (pure-data parent like `UObject`, `USaveGame`,
  `UCommonTextStyle`, `UCommonButtonStyle`, etc.): skip the file entirely
  *iff* `BuildBpirText` would have produced a header-only-with-empty-marker
  output (i.e. no ubergraphs/functions/macros have nodes). If the
  Blueprint somehow does carry a function or macro graph with nodes
  even on a data-only parent, still emit.

Alternative shape: build the BPIR text first, then drop the file when
its sole content is the empty marker AND the parent class is in the
graphless allowlist. Either implementation keeps the elision class-aware
so the marker still appears for the cases it was added for.

Test surface: extend `TestBpirEmptyGraphMarker.cpp` (or add a new test
next to it) with a fixture BP whose `ParentClass = UObject` / a style
class, asserting `AddStringFile(BpirTxt, ...)` is skipped; and keep the
existing test that empty-graph UserWidget BPs still emit the marker.

## History
- `#1-initial-repro` `OPEN` reporter — Filed from asset-dump cache audit. 500/2224 (22%) of `bpir.txt` files globally, 492/829 (59%) in the UI subtree, are 79-byte stubs containing only the graph header + `# (empty: no event/function bodies)` marker added by `B-bpir-stub-no-empty-marker`. Tally of `meta.json.parentClass` across the 500 stubs: 349 `CommonUI.CommonTextStyle`, 127 `CommonUI.CommonButtonStyle`, 6 `Engine.SaveGame`, 2 `CoreUObject.Object`, 1 `CommonUI.CommonTextScrollStyle`, 1 `Engine.CameraShakeBase`, plus derived style classes — ~96% on classes that structurally cannot host an event graph. The remaining ~20 stubs are on real graph-bearing types (UserWidget, ActorComponent, LyraHUDLayout, etc.) where the marker is the intended success signal and the file must stay. Recommend a class-aware elision in `AssetDumpHandler.cpp`: when `Blueprint->ParentClass` is not a known graph-bearing root and `BuildBpirText` would only produce a header + empty marker, skip `bpir.txt` entirely; otherwise keep current marker behavior so `B-bpir-stub-no-empty-marker`'s decompile-failure disambiguation remains intact for the case it was designed for. Repro paths: `App/App/UI/LobbyAndMenu/Editor/ButtonStyle-LeftBarButton/bpir.txt` (CommonButtonStyle), `App/App/UI/B_AxisListData/bpir.txt` (CoreUObject.Object).
- `#2-class-aware-bpir-elide` `IN-REVIEW` developer — Added `AssetDumpBuilder::ShouldEmitBpirText` helper alongside `BuildBpirText` (`Private/Utils/AssetDumpBuilder.{h,cpp}`): returns false only when (a) every ubergraph/function/macro graph is empty AND (b) `ParentClass` is not derived from any of the 7 graph-bearing roots (AActor, UActorComponent, UUserWidget, UBlueprintFunctionLibrary, UAnimInstance, UGameInstance, UGameModeBase). Null parent is fail-open. Wrapped the 3 BPIR emission sites in `AssetDumpHandler.cpp` (WBP/ABP/generic-BP branches; level BP path untouched) so the empty-marker contract from `B-bpir-stub-no-empty-marker` stays intact for graph-bearing parents while ~500 style-class / SaveGame / pure-UObject stubs are elided. Extended `TestBpirEmptyGraphMarker.cpp` with three new tests pinning the three regimes (graphless+empty → elide; UserWidget+empty → keep marker; graphless+function-graph-with-nodes → keep).
- `#3-skip-mcp-unavailable` `SKIP` tester — MCP server on port 19880 is not listening (UnrealEditor is running but plugin module isn't bound), so cannot drive a live `asset.dump` to verify runtime behavior. Source review confirms the fix is implemented per spec: `ShouldEmitBpirText` exists at `AssetDumpBuilder.cpp:243` with the documented graphless-elide / graph-bearing-keep logic over 6 roots (UGameModeBase intentionally omitted as AActor-subsumed, per inline comment); all 3 BPIR emission sites in `AssetDumpHandler.cpp` (WBP@552, ABP@615, BP@626) gated by `if (ShouldEmitBpirText(...))`; level BP path untouched; three new automation tests in `TestBpirEmptyGraphMarker.cpp` pin the three regimes. Status left IN-REVIEW for a tester with a live MCP session to re-dump `App/App/UI/LobbyAndMenu/Editor/ButtonStyle-LeftBarButton` and `App/App/UI/B_AxisListData` and confirm `bpir.txt` is absent.
- `#4-verify-live-dump` `DONE` tester — Verified live via `asset.dump`: (a) graphless `/App/App/UI/B_AxisListData` (`parentClass`: `/Script/CoreUObject.Object`) → `writtenPaths` = `[meta.json, properties.json]`, no `bpir.txt` on disk, `meta.json.sidecarsEmitted = ["properties.json"]`; (b) graphless `/App/App/UI/LobbyAndMenu/Popups/Buttons/ButtonStyle-PopupButton` (`parentClass`: `/Script/CommonUI.CommonButtonStyle`, used as substitute since original `ButtonStyle-LeftBarButton` no longer exists) → same outcome, no `bpir.txt`; (c) graph-bearing `/App/App/UI/LobbyAndMenu/TrackMenu/W_WidgetUtils` (UserWidget) → `bpir.txt` still emitted with real Construct/Tick/GetIsActive node bodies, confirming the marker contract for graph-bearing parents remains intact.
