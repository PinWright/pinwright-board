---
id: B-inspect-class-short-name-fails
title: "`system.inspect.inspect_class` requires full `/Script/` path despite short-name promise"
status: DONE
severity: Medium
category: bug
tags: []
---

# `system.inspect.inspect_class` requires full `/Script/` path despite short-name promise

`system.inspect.inspect_class className` tool description states it accepts "short names without U/A prefix (e.g. 'Blueprint', 'StaticMesh', 'MyDataAsset') or full paths". In practice, any short form returns `CLASS_NOT_FOUND`. Only the full `/Script/Module.ClassName` form resolves.

**Repro (session):**
1. Tried `system.inspect.inspect_class className="ReplaySaveHandlerWidget"` → `CLASS_NOT_FOUND`
2. Tried `UReplaySaveHandlerWidget` (with U prefix) → `CLASS_NOT_FOUND`
3. Tried `DroneFlightRecorderComponent` (pre-existing class) → `CLASS_NOT_FOUND`
4. Tried `UDroneFlightRecorderComponent` → `CLASS_NOT_FOUND`
5. `/Script/App.DroneFlightRecorderComponent` → `SUCCESS`
6. `/Script/App.ReplaySaveHandlerWidget` → `SUCCESS`

Related to but distinct from `B-asset-list-short-class-ensure` (IN-REVIEW) — same short-name-resolution pattern but a different handler.

**Workaround:** Always use the full `/Script/<Module>.<ClassName>` path.

**Fix:** Apply the same resolution pattern used in `B-asset-list-short-class-ensure` (`UClass::TryFindTypeSlow<UClass>` / `ResolveUClass` helper) to `system.inspect.inspect_class` before the path lookup fails.

## History
- `#1-initial-repro` `OPEN` reporter — Found while verifying newly-compiled C++ classes were available to MCP before BP authoring. Five `CLASS_NOT_FOUND` errors across both new and pre-existing classes before discovering the full-path requirement.
- `#2-resolve-uclass-fix` `IN-REVIEW` developer — Replaced direct `FindObject<UClass>` + hardcoded `/Script/Engine.` fallback in `EnvironmentHandler.cpp` (lines 1500-1502) with a single `ResolveUClass(ClassName)` call. Closed as part of the broader `E-class-name-format-inconsistency` fix.
- `#3-verified-short-name` `DONE` tester — Verified via MCP: `inspect_class className="ReplaySaveHandlerWidget"` returned `classPath: "/Script/App.ReplaySaveHandlerWidget"` with `success:true`. `"DroneFlightRecorderComponent"` (pre-existing App class) resolved with `classPath: "/Script/App.DroneFlightRecorderComponent"`. `"UWidgetBlueprint"` resolved via U-prefix strip to `"/Script/UMGEditor.WidgetBlueprint"`.
