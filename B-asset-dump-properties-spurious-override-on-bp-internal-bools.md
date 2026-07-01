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

**Fix (proposed):** In `PropertyUtils.cpp:1677-1690`, default `bOverridden = true`. When ParentContainer's class is the bool's owner but `Property->Identical()` returns false on a UMG-internal compile-time field (set during BP compile), every child gets flagged. Need special handling for UMG-class internal flags or compare against ParentClass's CDO directly.

## History
- `#1-initial-repro` `OPEN` reporter — UMG internal bool flags (`bHasScriptImplementedPaint`, `bHasScriptImplementedTick`) carry `is_overridden_locally: true` on every widget BP regardless of actual override state. Sample path: `App/App/UI/LobbyAndMenu/HUD/W_HUD_None/properties.json`. 327/327 widget BPs under `App/App/UI/` exhibit it.
- `#2-suppress-userwidget-compiler-managed-flags` `IN-REVIEW` developer — Added `IsCompilerManagedUserWidgetFlag` static helper in `Utils/PropertyUtils.cpp` and forced `bOverridden = false` for `bHasScriptImplementedPaint`/`bHasScriptImplementedTick`/`bAutomaticallyRegisterInputOnConstruction` on `UUserWidget`. These three flags are unconditionally rewritten by `WidgetBlueprintCompiler` from BP graph contents, so the byte-compare against the parent CDO always reported them as "overridden" on every WBP. Added `Tests/Private/Utility/TestAssetDumpInheritance.cpp::FAssetDumpPropertiesSuppressesUserWidgetCompilerFlagsTest`.
- `#3-verify-fix` `DONE` tester — Verified: re-ran `asset.dump` on `/App/App/UI/LobbyAndMenu/HUD/W_HUD_None` and `/App/App/UI/LobbyAndMenu/HUD/W_HUD_CarChase`; grepped `bHasScriptImplementedPaint|bHasScriptImplementedTick|bAutomaticallyRegisterInputOnConstruction` in both fresh `properties.json` files — zero matches in either (previously all three flags appeared with `is_overridden_locally: true`). Spurious overrides are now suppressed for compiler-managed UUserWidget flags.
