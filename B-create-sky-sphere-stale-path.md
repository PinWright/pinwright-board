---
id: B-create-sky-sphere-stale-path
title: "`environment.build.create_sky_sphere` always fails — hardcoded SkySphere blueprint path is stale in UE 5.7"
status: IN-REVIEW
severity: Medium
category: bug
tags: [environment, sky, lighting, stale-asset-path, ue57]
---

# `environment.build.create_sky_sphere` always fails — hardcoded SkySphere blueprint path is stale

`environment.build.create_sky_sphere` (documented as "Create a sky sphere actor
in the level", `RPC_NO_PARAMS`) returns `[CREATION_FAILED] Failed to create sky
sphere` on every call. The handler in
`Source/EditorAutomationRpcGateway/Private/Handlers/Environment/EnvironmentHandler.cpp`
(around line 297) loads the sky sphere class from a hardcoded engine path that no
longer exists in modern UE:

```cpp
UClass* SkySphereClass = LoadClass<AActor>(
    nullptr, TEXT("/Script/Engine.Blueprint'/Engine/Maps/Templates/SkySphere.SkySphere_C'"));
if (SkySphereClass) { /* spawn */ }
// else falls through to CREATION_FAILED
```

On UE 5.7 that asset path does not resolve, so `LoadClass` returns null and the
handler emits `CREATION_FAILED`. Confirmed via the gateway's own `asset.exists`:

- `asset.exists assetPath="/Engine/Maps/Templates/SkySphere"` → `{"exists":false}`
- `asset.exists assetPath="/Engine/EngineSky/BP_Sky_Sphere"` → `{"exists":true}`

The shipped sky sphere blueprint moved to `/Engine/EngineSky/BP_Sky_Sphere`
(class `BP_Sky_Sphere_C`). Because the create helper never produces an actor, the
sibling `environment.build.set_time_of_day` is also collaterally dead: it scans
level actors for a class whose name `Contains("SkySphere")` and calls
`SetTimeOfDay` via reflection — but no sky sphere was ever created, so it returns
`[SET_TIME_FAILED] Sky sphere not found or time function not available`. (Even a
hand-spawned `/Engine/EngineSky/BP_Sky_Sphere` would not satisfy the
`Contains("SkySphere")` class-name match — its class is `BP_Sky_Sphere_C` — and
its time control is the `TimeOfDay` BP variable rather than a `SetTimeOfDay`
function, so that path needs revisiting too, but the primary defect is the dead
`create_sky_sphere` path.)

Distinct from `F-sky-cloud-reflection-actors` (DONE): that ticket added the modern
`environment.spawn_sky_atmosphere` / `spawn_volumetric_cloud` /
`spawn_reflection_capture` RPCs and only mentioned `create_sky_sphere` in passing.
It did not touch the legacy `create_sky_sphere` load path, which remains broken.

**Repro (verbatim):**
- `call("environment.build.create_sky_sphere", {})`
  → `[CREATION_FAILED] Failed to create sky sphere`  (reproduced on replay, deterministic)
- `call("asset.exists", {"assetPath":"/Engine/Maps/Templates/SkySphere"})`
  → `{"success":true,"exists":false,...}`
- `call("asset.exists", {"assetPath":"/Engine/EngineSky/BP_Sky_Sphere"})`
  → `{"success":true,"exists":true,...}`

**Impact:** A documented, parameter-free environment helper is 100% non-functional
on UE 5.7, and it silently takes down `set_time_of_day` with it. Agents building an
outdoor lighting rig must abandon the helper entirely.

**Workaround:** Spawn the blueprint directly via
`actor.spawn_from_blueprint` with `blueprintPath="/Engine/EngineSky/BP_Sky_Sphere"`.

**Fix:** Update the load path to the current engine sky sphere
(`/Engine/EngineSky/BP_Sky_Sphere.BP_Sky_Sphere_C`), preferably with a fallback
chain that tries the legacy `/Engine/Maps/Templates/SkySphere.SkySphere_C` (older
engines) then `/Engine/EngineSky/BP_Sky_Sphere.BP_Sky_Sphere_C` so the handler
works across UE 5.3–5.7. While in the file, align `set_time_of_day`'s sky-sphere
discovery (class-name `Contains("SkySphere")` + `SetTimeOfDay` function) with the
actor the fixed `create_sky_sphere` actually spawns, so the two legacy helpers
remain a usable pair.

## History
- `#1-initial-repro` `OPEN` reporter — `environment.build.create_sky_sphere` (RPC_NO_PARAMS) deterministically returns `[CREATION_FAILED] Failed to create sky sphere`; reproduced on direct replay. Root cause: `EnvironmentHandler.cpp` (~line 297) `LoadClass<AActor>(nullptr, "/Script/Engine.Blueprint'/Engine/Maps/Templates/SkySphere.SkySphere_C'")` returns null on UE 5.7 — that path no longer exists. Confirmed via gateway `asset.exists`: `/Engine/Maps/Templates/SkySphere` → exists:false, `/Engine/EngineSky/BP_Sky_Sphere` → exists:true. Collateral: `environment.build.set_time_of_day` then always returns `[SET_TIME_FAILED]` because no sky sphere actor is ever created. Distinct from DONE `F-sky-cloud-reflection-actors` (that added the modern spawn_* RPCs, never fixed this load path). Workaround: `actor.spawn_from_blueprint` with `/Engine/EngineSky/BP_Sky_Sphere`.
- `#2-fix-fallback-chain` `IN-REVIEW` developer — Fixed in `Source/EditorAutomationRpcGateway/Private/Handlers/Environment/EnvironmentHandler.cpp`. Added a static `LoadSkySphereClass()` resolver that tries the legacy `/Engine/Maps/Templates/SkySphere.SkySphere_C` first, then the modern `/Engine/EngineSky/BP_Sky_Sphere.BP_Sky_Sphere_C`, so `create_sky_sphere` resolves a class on UE 5.3–5.7 instead of falling straight to CREATION_FAILED. Aligned `set_time_of_day`'s discovery with the actor `create_sky_sphere` now spawns: added `IsSkySphereActor()` (matches both `SkySphere` and the modern `BP_Sky_Sphere`/`Sky_Sphere` class names) and, when no `SetTimeOfDay` UFunction exists, set the time via the modern sphere's BP variable and rerun its construction script. Correction to the ticket body's parenthetical: the modern `BP_Sky_Sphere` does NOT expose a `TimeOfDay` variable — its time-of-day control is the `Sun height` double (live `system.inspect.inspect_class` of `BP_Sky_Sphere_C` shows `Sun height`, no `Time of Day`), so the fallback maps the 0..24 time onto `Sun height` (-cos: midnight→-1, noon→+1) rather than a nonexistent variable. Regression test: `environment.build.create_sky_sphere.ResolvesEngineClass` in `Source/EditorAutomationRpcGateway/Private/Tests/World/TestEnvironmentHandlers.cpp` drives the real handler with capture and asserts it succeeds and does not return `CREATION_FAILED` (it fails if the resolver is reverted to the lone stale path). Did not compile/run tests — deferred to the test phase.
