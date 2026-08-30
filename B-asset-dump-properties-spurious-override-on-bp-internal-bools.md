---
id: B-asset-dump-properties-spurious-override-on-bp-internal-bools
title: "asset.dump properties.json marks bHasScriptImplementedPaint/Tick as locally-overridden on every WidgetBP — every widget looks 'modified'"
status: DONE
severity: Medium
category: bug
tags: [asset-dump, properties, widget]
---

# asset.dump properties.json marks bHasScriptImplementedPaint/Tick as locally-overridden on every WidgetBP — every widget looks 'modified'

Inherited UMG bool flags (`bHasScriptImplementedPaint`, `bHasScriptImplementedTick`) appear with `is_overridden_locally: true` on every widget BP, yet hold the parent's default value (false).

Diff/audit consumers can't distinguish actual overrides from spurious ones — every widget looks "modified". 327/327 widget BPs under `App/App/UI/` show this.

**Repro:**
1. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/App/App/UI/LobbyAndMenu/HUD/W_HUD_None/properties.json`.
2. Observe: `bHasScriptImplementedPaint` / `bHasScriptImplementedTick` carry `is_overridden_locally: true` despite the value matching the parent.

**Fix (proposed):** In the `bOverridden` block of `ExportPropertyToJsonValueWithInheritance` — now `Utils/PropertyExport.cpp:1373-1387` — `bOverridden` defaults to `true` (`:1373`). When ParentContainer's class is the bool's owner but `Property->Identical()` (`:1381`) returns false on a UMG-internal compile-time field (set during BP compile), every child gets flagged. Need special handling for UMG-class internal flags or compare against ParentClass's CDO directly.

*(Applied by `#2`: the suppression is live at `PropertyExport.cpp:1384-1387`.)*

## History
- `#1-initial-repro` `OPEN` reporter — UMG internal bool flags (`bHasScriptImplementedPaint`, `bHasScriptImplementedTick`) carry `is_overridden_locally: true` on every widget BP regardless of actual override state. Sample path: `App/App/UI/LobbyAndMenu/HUD/W_HUD_None/properties.json`. 327/327 widget BPs under `App/App/UI/` exhibit it.
- `#2-suppress-userwidget-compiler-managed-flags` `IN-REVIEW` developer — Added `IsCompilerManagedUserWidgetFlag` static helper in `Utils/PropertyUtils.cpp` and forced `bOverridden = false` for `bHasScriptImplementedPaint`/`bHasScriptImplementedTick`/`bAutomaticallyRegisterInputOnConstruction` on `UUserWidget`. These three flags are unconditionally rewritten by `WidgetBlueprintCompiler` from BP graph contents, so the byte-compare against the parent CDO always reported them as "overridden" on every WBP. Added `Tests/Private/Utility/TestAssetDumpInheritance.cpp::FAssetDumpPropertiesSuppressesUserWidgetCompilerFlagsTest`.
- `#3-verify-fix` `DONE` tester — Verified: re-ran `asset.dump` on `/App/App/UI/LobbyAndMenu/HUD/W_HUD_None` and `/App/App/UI/LobbyAndMenu/HUD/W_HUD_CarChase`; grepped `bHasScriptImplementedPaint|bHasScriptImplementedTick|bAutomaticallyRegisterInputOnConstruction` in both fresh `properties.json` files — zero matches in either (previously all three flags appeared with `is_overridden_locally: true`). Spurious overrides are now suppressed for compiler-managed UUserWidget flags.
- `#4-repoint-citations-after-property-utils-split` `DONE` reporter — Citation maintenance only; **no behavioural claim changes and the status is untouched**. `Utils/PropertyUtils.cpp` was split into `PropertyExport.cpp` / `PropertyImport.cpp` / `PropertyInspection.cpp` / `PropertyDiff.cpp` (`PropertyUtils.h` survives only as a deprecated umbrella forwarder), so this ticket's two `PropertyUtils.cpp` citations were unresolvable paths, not stale line numbers. Both resolve into `PropertyExport.cpp`; verified against the code at plugin HEAD `ef8a1f1b`. Map: body `PropertyUtils.cpp:1677-1690` → `PropertyExport.cpp:1373-1387`, inside `ExportPropertyToJsonValueWithInheritance` (`:1324`) — `bool bOverridden = true;` at `:1373`, the `ParentClass->IsChildOf(OwnerClass)` guard at `:1377`, and `bOverridden = !Property->Identical(ChildPtr, ParentPtr, PPF_DeepComparison);` at `:1381`, exactly the mechanism the body describes. `#2`'s `IsCompilerManagedUserWidgetFlag` "in `Utils/PropertyUtils.cpp`" → `PropertyExport.cpp:1309-1322`, with the three flag names at `:1318-1320` and **two** call sites, not one: the `bOverridden` forcing at `:1384-1387` that `#2` describes, and a second in `ShouldEmitClassDumpProperty` at `:1434`, which drops the flags from the class-dump walk entirely. `#2` is left verbatim per the append-only rule and maps here. Second correction to `#2`, verified in the same pass: it names the test as `Tests/Private/Utility/TestAssetDumpInheritance.cpp`; the real path is `Source/PinWright/Private/Tests/Utility/TestAssetDumpInheritance.cpp` (`Private/Tests/`, not `Tests/Private/`), with `FAssetDumpPropertiesSuppressesUserWidgetCompilerFlagsTest` at `:826-897`. That directory inversion is not specific to this ticket and is worth grepping for board-wide.
