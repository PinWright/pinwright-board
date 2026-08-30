---
id: F-water-actor-authoring
title: "Water plugin actor RPCs (WaterBody, WaterZone, OceanActor)"
status: DONE
severity: Medium
category: feature
tags: [water, actor, spawn, content-authoring]
---

# Water Plugin Actor RPCs (WaterBody, WaterZone, OceanActor)

There is currently no `water.*` namespace despite Water being a stock UE 5.6
plugin shipped with the engine. Agents can flip `bWaterVolume` on a
`PhysicsVolume` (three references in `rpc-method-reference.generated.md` —
all on physics-volume property maps) but cannot create an actual rendered
water surface, ocean horizon, or buoyancy/post-process zone. This blocks
every water/ocean content pass.

Add a focused `water.*` namespace targeting the public Water plugin API:

- `water.spawn_water_body(type: "River"|"Lake"|"Ocean"|"Custom", location, splinePoints?)`
  — spawns the matching `AWaterBody*` subclass (`AWaterBodyRiver`,
  `AWaterBodyLake`, `AWaterBodyOcean`, `AWaterBodyCustom`). For
  spline-based types (River/Lake/Custom), accept optional `splinePoints`
  and reuse existing `spline.*` RPCs to set point positions after spawn
  rather than reimplementing spline editing inside the water handler.
- `water.spawn_water_zone(location, extent?)` — spawns `AWaterZone`,
  required for water rendering and underwater post-process to function.
- `water.set_water_body_material(actor, waterMaterial?, underwaterMaterial?, riverToLakeTransitionMaterial?, riverToOceanTransitionMaterial?)`
  — sets the materials on `UWaterBodyComponent`. Optional params let
  callers update one channel at a time.
- `water.set_water_body_underwater_post_process(actor, postProcessMaterial?, settings?)`
  — wires up the underwater post-process material/settings on the body's
  `UWaterBodyComponent`.

All handlers route through `UWaterBodyComponent` (and `AWaterZone`'s
component) for property mutation, following the existing actor-property
pattern. Class resolution uses the standard short-name → class-path lookup
so callers can pass `"WaterBodyOcean"` or full `/Script/Water.WaterBodyOcean`.

**Plugin gating:** The Water plugin is optional and can be disabled per
project. Follow the `TryAddConditionalModule()` pattern in `Build.cs` —
gate compilation on the Water module being available and, at runtime,
return a clear `WATER_PLUGIN_NOT_AVAILABLE` error if the module is not
loaded. The namespace should still appear in discovery (returning the
gate error) so agents get a useful signal instead of `METHOD_NOT_FOUND`.

**Out of scope:** Water mesh tessellation tuning, custom water body
weightmaps, landscape water-brush integration, gerstner wave parameter
authoring on `AWaterBodyOcean`. Those are follow-on tickets if requested.

**Fix:** Add `Source/PinWright/Private/Handlers/Water/WaterHandler.cpp`
with the four RPCs above. Conditionally add the `Water` module in
`PinWright.Build.cs` via `TryAddConditionalModule()`.
Editorial wiki overlay at `docs/wiki/water.md` documenting the spawn →
configure → `spline.set_points` workflow for river/lake/custom bodies.

## History
- `#1-initial-request` `OPEN` reporter — No `water.*` namespace exists; only `bWaterVolume` field on PhysicsVolume is reachable. Proposed four-RPC surface (`spawn_water_body`, `spawn_water_zone`, `set_water_body_material`, `set_water_body_underwater_post_process`) routing through `UWaterBodyComponent`, reusing `spline.*` for spline-body point authoring, with `TryAddConditionalModule` compile gate and a runtime `WATER_PLUGIN_NOT_AVAILABLE` error when the Water plugin is disabled.
- `#2-water-namespace-with-conditional-gate` `IN-REVIEW` developer — Added Handlers/Water/WaterHandler.cpp with 4 RPCs (spawn_water_body, spawn_water_zone, set_water_body_material, set_water_body_underwater_post_process) gated via __has_include("WaterBodyActor.h") + Build.cs TryAddConditionalModule("Water"). Dropped the optional splinePoints param per analysis — caller chains spline.set_spline_point_position. Added wiki overlay docs/wiki/water.md and regression test that proves registration survives the conditional gate.
- `#3-verify-namespace-and-gate` `DONE` tester — Verified: wiki.get path="water" renders overlay with all 4 method links; water.spawn_water_zone? returns full schema (name/location/extent). Live water.spawn_water_zone and water.spawn_water_body type=Ocean both reach the handler body (return SPAWN_FAILED, not WATER_PLUGIN_NOT_AVAILABLE or METHOD_NOT_FOUND) — proves MCP_HAS_WATER=1 compile gate took effect and namespace is registered. SPAWN_FAILED is environmental (L_Core UI map context, "No actor was spawned" in log) and outside the fix's scope.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place; every rewritten path was confirmed to exist at plugin HEAD `ef8a1f1b`. One file basename renamed by the same plugin commit is repointed with it (`EditorAutomationRpcGateway.Build.cs` → `PinWright.Build.cs`, `EditorAutomationRpcGateway_SCSHandlers` / `_BlueprintHandlers_List` → `PinWright_*`), verified present at HEAD. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
