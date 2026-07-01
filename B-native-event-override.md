---
id: B-native-event-override
title: "`compile_bpir` cannot create BlueprintNativeEvent overrides"
status: DONE
severity: Critical
category: bug
tags: []
---

# `compile_bpir` cannot create BlueprintNativeEvent overrides

`entry function GetContentPanel()` on a child of `TrackHUDLayout` creates standalone function instead of `_Implementation` override. Errors: "function name already used", "different signature".

**Workaround:** Manual — Override Functions dropdown in editor.

**Fix:** Detect parent `BlueprintNativeEvent` declarations and create override graph. Or support `entry override Foo()` syntax.

## History
- `#1-initial-repro` `OPEN` reporter — GetContentPanel override on W_HUD_PhotoInspection_Route had to be done manually.
- `#2-parent-class-fallback` `IN-REVIEW` developer — Root cause: `TryResolveBlueprintOverride` relies on `SkeletonGeneratedClass` via `GetOverrideFunctionClass`, which is null for Widget BPs or freshly-created BPs. Three fixes: (1) ParentClass fallback in `TryResolveBlueprintOverride` — searches `Blueprint->ParentClass->FindFunctionByName()` when skeleton lookup fails. (2) Phase 0c skeleton-ensure in BpirCompiler — calls `RegenerateSkeletonOnly` before entry point setup. (3) Replace mode cleanup for Function-kind overrides placed as events. Test 52C2 validates override creation without skeleton.
- `#3-verified-override-clean` `DONE` tester — Verified: `entry override GetContentPanel()` on W_McpOverrideTest (child of TrackHUDLayout) compiled clean. blueprint_inspect shows GetContentPanel under Functions with its own graph (2 nodes). BP status UpToDate.
