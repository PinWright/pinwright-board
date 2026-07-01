---
id: E-asset-dump-meta-add-kind-and-blueprint-type
title: "asset.dump meta.json needs `kind` + `blueprintType` fields — parentClass alone misclassifies MacroLibrary and non-UserWidget-named widget BPs"
status: DONE
severity: Medium
category: ergonomic
tags: [asset-dump, meta, schema, blueprint, widget, classification]
---

# asset.dump meta.json needs `kind` + `blueprintType` fields — parentClass alone misclassifies MacroLibrary and non-UserWidget-named widget BPs

`meta.json` currently exposes `className` (`<AssetName>_C` for BP-class entries) and `parentClass` (UE path of the parent C++/BP class). With `assetType` removed in schema v4 (see `B-asset-dump-meta-assettype-degenerate`), consumers must classify Blueprint kind by inspecting `parentClass`. That string is **not** a sufficient discriminator — two distinct miscategorization patterns show up in the live dump under `C:\Unity\unreal-fpv-pluginwork\.editor-automation\asset-dumps\`.

**Pattern 1 — MacroLibrary BPs are indistinguishable from Actor BPs.**

`App\Blueprints\Data\ML_A_MacroLibrary\meta.json`:

```json
{
  "assetPath": "/App/Blueprints/Data/ML_A_MacroLibrary.ML_A_MacroLibrary",
  "assetType": "Blueprint",
  "className": "ML_A_MacroLibrary_C",
  "parentClass": "/Script/Engine.Actor",
  ...
}
```

`parentClass` and the `_C` className shape are identical to a real Actor BP. The only behavioral fingerprints are sidecar presence (no `scs.json`, empty `properties.json`) — consumers that classify off `meta.json` alone will route this as a spawnable actor.

**Pattern 2 — UserWidget-derived BPs whose C++ parent is not literally `UserWidget` look like Actor/UObject BPs.**

`App\App\UI\LobbyAndMenu\Settings\W_ControllerAxesPanel\meta.json`:

```json
{
  "assetPath": "/App/App/UI/LobbyAndMenu/Settings/W_ControllerAxesPanel.W_ControllerAxesPanel",
  "assetType": "WidgetBlueprint",
  "className": "W_ControllerAxesPanel_C",
  "parentClass": "/Script/PDSGame.LyraControllerAxisEditor",
  ...
}
```

`tree.xml` is present (14 KB+) and `bpir.txt` is non-trivial (15.6 KB), proving the asset is a widget — but `parentClass` points at a project C++ class that doesn't carry `UserWidget` in its name. A scan of the live dump finds **268 widget BPs** (assets that have a sibling `tree.xml`) whose `parentClass` does not contain `UserWidget` / `CommonUserWidget` / `CommonActivatableWidget`. Top parent classes:

| Count | parentClass |
|------:|-------------|
| 66 | `/Script/PDSGame.LyraHUDLayout` |
| 49 | `/Script/PDSGame.LyraActivatableWidget` |
| 35 | `/Script/PDSGame.LyraButtonBase` |
| 30 | `/Script/Blutility.EditorUtilityWidget` |
| 9 | `/App/App/UI/LobbyAndMenu/Popups/B_AccountApiWidgetBase.B_AccountApiWidgetBase_C` (BP chain) |
| 6 | `/Script/App.TrackHUDLayout` |
| 4 | `/Script/PDSGame.LyraSettingScreen` |
| 4 | `/Script/App.MarkerWidget` |
| 3 | `/Script/PDSGame.MaterialProgressBar` |
| 3 | `/Script/GameSettings.GameSettingDetailExtension` |
| 2 | `/Script/App.WebUIHUDLayout` |
| 1 | `/Script/PDSGame.LyraSafeZoneEditor` |
| ... | (long tail of project-specific bases) |

Consumers that maintain a denylist of "known UserWidget bases" must enumerate every project-side widget base class — a list that drifts every time someone adds a new C++ widget parent. The C++ MRO is the only authoritative answer, and the dumping side already has it loaded.

## Proposed schema additions

Add two top-level fields to `meta.json` so consumers can classify without ascending the parent chain:

- **`kind`** — coarse runtime classification, one of: `"Widget"`, `"Actor"`, `"Component"`, `"AnimInstance"`, `"Interface"`, `"FunctionLibrary"`, `"MacroLibrary"`, `"Object"`. Derived from `UBlueprint::BlueprintType` (for the macro/interface/function-library cases) combined with `UClass::IsChildOf` checks against `UUserWidget`, `AActor`, `UActorComponent`, `UAnimInstance` for the rest. For non-Blueprint assets, fill `kind` from the matching `IsChildOf` walk on `Asset->GetClass()`.
- **`blueprintType`** — exact `UBlueprint::BlueprintType` enum string for BP-class entries only: `"Normal"`, `"MacroLibrary"`, `"Interface"`, `"FunctionLibrary"`, `"Const"`. Omitted on non-Blueprint assets.

Existing fields (`className`, `parentClass`) stay — they're still useful for ancestor lookups and reflection joins. The new fields are additive.

## Implementation pointer

`Source\EditorAutomationRpcGateway\Private\Utils\AssetDumpBuilder.cpp::BuildMetaJson` (lines 42–93) already branches on `Cast<UWidgetBlueprint>` / `Cast<UBlueprint>` / else. Extend each branch:

- `UWidgetBlueprint`: `kind = "Widget"`, `blueprintType = BlueprintTypeToString(WBP->BlueprintType)`.
- `UBlueprint`:
  - `blueprintType = BlueprintTypeToString(BP->BlueprintType)`.
  - if `BP->BlueprintType == BPTYPE_MacroLibrary` → `kind = "MacroLibrary"`.
  - elif `BP->BlueprintType == BPTYPE_Interface` → `kind = "Interface"`.
  - elif `BP->BlueprintType == BPTYPE_FunctionLibrary` → `kind = "FunctionLibrary"`.
  - else (Normal/Const): walk `BP->GeneratedClass` (or `BP->ParentClass` fallback when GeneratedClass is null — see `B-asset-dump-bp-meta-degrades-when-genclass-null`) with `IsChildOf(UUserWidget::StaticClass())` / `AActor` / `UActorComponent` / `UAnimInstance` and emit `kind` accordingly; fall back to `"Object"`.
- Else (native asset): same `IsChildOf` walk on `Asset->GetClass()`.

Bump `dumpSchemaVersion` (currently 5). Update `docs/wiki/asset.md` schema list and add a test in `Tests/Private/Utility/TestAssetDumpBuilder.cpp` covering a MacroLibrary BP and a non-UserWidget-named widget BP (both reproducible from the dump cache above).

**Fix:** Add `kind` (always) and `blueprintType` (BP-class only) to `BuildMetaJson` derived from `UBlueprint::BlueprintType` + `UClass::IsChildOf` on the generated class. Consumers drop their parentClass denylists.

## History
- `#1-initial-repro` `OPEN` reporter — `meta.json` schema v5 exposes only `className` + `parentClass`, which fails to disambiguate (a) MacroLibrary BPs vs Actor BPs (`App\Blueprints\Data\ML_A_MacroLibrary\meta.json` reports `parentClass: "/Script/Engine.Actor"` identical to a real Actor BP) and (b) UserWidget-derived BPs whose C++ parent doesn't carry `UserWidget` in its name (`W_ControllerAxesPanel` → `/Script/PDSGame.LyraControllerAxisEditor`, with `tree.xml` present proving widget). Live-dump scan finds 268 widget BPs (asset has sibling `tree.xml`) whose parentClass doesn't contain `UserWidget`/`CommonUserWidget`/`CommonActivatableWidget`; top parents include `LyraHUDLayout` (66), `LyraActivatableWidget` (49), `LyraButtonBase` (35), `EditorUtilityWidget` (30), `TrackHUDLayout`, `WebUIHUDLayout`, `LyraSettingScreen`, `MarkerWidget`. Proposed addition: top-level `kind` (Widget/Actor/Component/AnimInstance/Interface/FunctionLibrary/MacroLibrary/Object) plus `blueprintType` (BP-class only, raw `UBlueprint::BlueprintType` enum string). Derive `kind` from `BP->BlueprintType` for the library/interface cases and from `IsChildOf(UUserWidget::StaticClass())` etc. on `GeneratedClass` for the Normal/Const case; implementation hook is `Utils\AssetDumpBuilder.cpp::BuildMetaJson` lines 42–93.
- `#2-kind-and-blueprint-type-added` `IN-REVIEW` developer — Extended `AssetDumpBuilder::BuildMetaJson` in `Utils/AssetDumpBuilder.cpp` to emit `kind` on every asset and `blueprintType` on Blueprint assets. Added a file-local `ResolveRuntimeKind(UClass*)` helper that walks `IsChildOf` against `UUserWidget` / `AActor` / `UActorComponent` / `UAnimInstance` and falls back to `"Object"`. `UWidgetBlueprint` is hardcoded to `kind = "Widget"` regardless of C++ parent (fixes the 268 widget BPs whose `parentClass` doesn't contain `UserWidget`). For `UBlueprint` the kind derives from `BlueprintType` for MacroLibrary / Interface / FunctionLibrary, otherwise walks `GeneratedClass` with `ParentClass` fallback when GeneratedClass is null. `blueprintType` reuses `AssetDumpHandler::BlueprintTypeToStatusString` — declared in `AssetDumpHandlerInternal.h` and called from the builder via that internal include to avoid duplicating the switch. Schema bumped to v6. `Tests/Utility/TestAssetDumpBuilder.cpp::MetaJsonShape` updated to expect `kind == "Component"` (USceneComponent is a UActorComponent child).
- `#3-skip-editor-offline` `SKIP` tester — Editor MCP endpoint at 127.0.0.1:19880 refused connection on two consecutive POSTs, so the fix could not be exercised against a live `asset.dump` of `ML_A_MacroLibrary` / `W_ControllerAxesPanel`. Cached dump under `.editor-automation/asset-dumps/App/Blueprints/Data/ML_A_MacroLibrary/meta.json` is still schema v3 (pre-fix), confirming no live dump has been re-run since the change. Source inspection of `AssetDumpBuilder.cpp::BuildMetaJson` (lines 67–149) shows the contract is implemented as described (`kind`, `blueprintType`, `UWidgetBlueprint→"Widget"`, `BPTYPE_MacroLibrary→"MacroLibrary"`, schema v6), but behavior verification requires an editor instance.
- `#4-verify-fix` `DONE` tester — Verified: live `asset.dump` on both repro assets returns the new fields at schema v6. `ML_A_MacroLibrary/meta.json` now has `kind: "MacroLibrary"`, `blueprintType: "MacroLibrary"` (disambiguates from Actor BPs despite `parentClass: "/Script/Engine.Actor"`). `W_ControllerAxesPanel/meta.json` now has `kind: "Widget"`, `blueprintType: "Normal"` (correctly classified despite `parentClass: "/Script/PDSGame.LyraControllerAxisEditor"` lacking "UserWidget"). Both meta.json files also gained `propertiesStatus` and `sidecarsEmitted`; `dumpSchemaVersion: 6` confirmed on both.
