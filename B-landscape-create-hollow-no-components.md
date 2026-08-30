---
id: B-landscape-create-hollow-no-components
title: "landscape.create on UE 5.7 returns success but spawns a hollow landscape with zero ULandscapeComponents (sculpt no-ops, edit errors INVALID_LANDSCAPE)"
status: IN-REVIEW
severity: Critical
category: bug
tags: [landscape, ue5.7, silent-corruption, hollow, sculpt, edit, components]
---

# landscape.create builds a component-less landscape on UE 5.7

On UE 5.7, `landscape.create` returns `success:true` with a full geometry echo,
but the spawned `ALandscape` has **zero `ULandscapeComponent`s** — only its
`RootComponent0` (a bare `USceneComponent`). The actor has no heightmap surface:
`XYtoComponentMap` is empty and the bounding box is all-zero. Every downstream
shaping RPC is therefore dead on the actor:

- `landscape.sculpt` returns `success:true, modifiedVertices:0` — a **silent
  success-with-no-effect**: the brush finds no components under it, writes
  nothing, and reports no error.
- `landscape.edit` (raise/lower/flatten/set) errors `[INVALID_LANDSCAPE] Failed
  to get landscape extent` because the landscape has no extent.

This reproduces with the **documented default geometry** (`quadsPerComponent:63`,
`sectionsPerComponent:1`), so it is not the divisibility/invariant defect tracked
in `B-landscape-create-inconsistent-subsection-geometry` (that ticket is about
`ComponentSizeQuads != SubsectionSizeQuads*NumSubsections` when quads is not a
multiple of sections — a corrupt-fields case, not a hollow-actor case; 63/1 is
perfectly valid and still produces zero components here). The net effect: on the
5.7 host under test, `landscape.create` produces a non-functional landscape and
the entire sculpt/edit/flatten workflow is unusable, with no error surfaced at
create time.

## Root cause

`Handlers/Environment/LandscapeHandler.cpp`, the async create lambda's
version-gated heightmap-apply block. The UE 5.7 branch never instantiates the
component grid — it only calls `CreateDefaultLayer()` then writes through
`FLandscapeEditDataInterface::SetHeightData`, which only writes into
**already-existing** components via `XYtoComponentMap`. With no components,
`SetHeightData` is a no-op and the actor stays hollow:

```cpp
#if UE_VERSION_NEWER_THAN_OR_EQUAL(5, 7, 0)
  if (Landscape->GetLayersConst().Num() == 0) {
    Landscape->CreateDefaultLayer();
  }
  ULandscapeInfo* LandscapeInfo = Landscape->GetLandscapeInfo();
  if (LandscapeInfo && HeightArray.Num() > 0) {
    if (Landscape->GetRootComponent() && !Landscape->GetRootComponent()->IsRegistered()) {
      Landscape->RegisterAllComponents();   // no LandscapeComponents exist to register
    }
    FLandscapeEditDataInterface LandscapeEdit(LandscapeInfo);
    LandscapeEdit.SetHeightData(InMinX, InMinY, InMaxX, InMaxY, HeightArray.GetData(), 0, true);
    LandscapeEdit.Flush();
  }
#elif UE_VERSION_NEWER_THAN_OR_EQUAL(5, 5, 0)
  ... same SetHeightData-only path ...
#else
  Landscape->Import(Landscape->GetLandscapeGuid(), 0, 0, CaptComponentsX - 1, CaptComponentsY - 1,
                    CaptNumSubsections, CaptSubsectionSizeQuads, ImportHeightData, ...);
  Landscape->CreateDefaultLayer();
#endif
```

Only the legacy `#else` path calls `ALandscape::Import(...)`, which actually
**creates and registers the `ULandscapeComponent` grid** and populates
`XYtoComponentMap`. The 5.5+ and 5.7 branches skip `Import()` entirely and rely
on `SetHeightData` into a component set that was never built, so the landscape is
born empty.

## What it should do

- `landscape.create` must build the component grid on 5.7 (and 5.5+) — e.g. call
  the appropriate component-import path (the deprecated `Import()` still works and
  is what the legacy branch uses, or use `ULandscapeInfo`/`FLandscapeImportHelper`
  to materialize `ULandscapeComponent`s) before/instead of the bare
  `SetHeightData`, so `XYtoComponentMap` is populated and the actor has a real
  heightmap.
- `landscape.sculpt` must not report `success:true` when it modified zero
  vertices because the target has no components — either build the components at
  create time (preferred) or fail loudly (`INVALID_LANDSCAPE`) rather than
  silently returning `modifiedVertices:0` with `success:true`.

## Verbatim repro (live, UE 5.7, replayed for this ticket)

1. `landscape.create`
   ```json
   {"name":"ReplayTerrain","location":{"x":0,"y":0,"z":0},
    "componentsX":4,"componentsY":4,"quadsPerComponent":63,
    "sectionsPerComponent":1,"materialPath":"/Engine/EngineMaterials/WorldGridMaterial"}
   ```
   Response (reports success):
   ```json
   {"success":true,"landscapePath":".../PersistentLevel.Landscape_13","actorLabel":"ReplayTerrain",
    "componentsX":4,"componentsY":4,"quadsPerComponent":63,"subsectionSizeQuads":63,
    "numSubsections":1,"componentSizeQuads":63,"message":"Landscape created successfully"}
   ```

2. `actor.get_components` `{"actorName":"ReplayTerrain"}` → only `RootComponent0`
   (`/Script/Engine.SceneComponent`), `"count":1` — **zero LandscapeComponents**.
   `actor.describe` with `componentClass:"LandscapeComponent"` → `"components":[]`.

3. `actor.get_bounding_box` `{"actorName":"ReplayTerrain"}` →
   `{"origin":[0,0,0],"extent":[0,0,0]}` (hollow).

4. `landscape.sculpt`
   `{"landscapeName":"ReplayTerrain","location":{"x":8000,"y":8000,"z":0},"toolMode":"Raise","brushRadius":4000,"strength":0.5}`
   → `{"success":true,"toolMode":"Raise","modifiedVertices":0,"message":"Landscape sculpted"}`
   (silent no-op; also reproduced with a brush centered at origin, radius 8000, strength 1).

5. `landscape.edit`
   `{"landscapeName":"ReplayTerrain","operation":"raise","region":{"minX":10,"minY":10,"maxX":50,"maxY":50}}`
   → error `[INVALID_LANDSCAPE] Failed to get landscape extent`.

## History
- `#1-initial-repro` `OPEN` reporter — Replayed `landscape.create` with the documented default geometry (componentsX/Y=4, quadsPerComponent=63, sectionsPerComponent=1) against the live UE 5.7 host. Create returned `success:true` with a full geometry echo, but `actor.get_components` shows only `RootComponent0` (SceneComponent), `count:1`, and `actor.describe componentClass=LandscapeComponent` returns `components:[]` — zero ULandscapeComponents; `actor.get_bounding_box` is all-zero. Consequently `landscape.sculpt` returns `success:true, modifiedVertices:0` (silent no-op, reproduced twice) and `landscape.edit raise` errors `[INVALID_LANDSCAPE] Failed to get landscape extent`. Root cause: LandscapeHandler.cpp's `UE_VERSION_NEWER_THAN_OR_EQUAL(5,7,0)` create branch (and the 5.5+ branch) only does `CreateDefaultLayer()` + `FLandscapeEditDataInterface::SetHeightData`, which writes into existing components via XYtoComponentMap and never builds the component grid; only the legacy `#else` branch calls `ALandscape::Import(...)` which creates/registers the components. So on 5.7 the landscape spawns hollow and the whole sculpt/edit/flatten workflow is dead. Distinct from B-landscape-create-inconsistent-subsection-geometry (that is the corrupt-geometry-fields case for non-divisible quads/sections; this repro uses the valid 63/1 default and still yields zero components).
- `#2-fix` `IN-REVIEW` developer — Root-cause fix: routed `landscape.create` through `ALandscapeProxy::Import` on EVERY engine version instead of the 5.5+/5.7 `SetHeightData`-only path. `Import` is the only path that instantiates and registers the `ULandscapeComponent` grid (loops `NewObject<ULandscapeComponent>` + `Init`, populating `LandscapeComponents` and `ULandscapeInfo::XYtoComponentMap`) and sets `ComponentSizeQuads`/`NumSubsections`/`SubsectionSizeQuads`; the old branches only did `CreateDefaultLayer()` + `FLandscapeEditDataInterface::SetHeightData`, which writes through `XYtoComponentMap` into already-existing components and never creates any — a guaranteed no-op against the empty map, leaving the actor hollow while create reported `success:true`. Verified against engine source that `ALandscapeProxy::Import` carries NO `UE_DEPRECATED` on UE 5.7 (LandscapeProxy.h:1397-1400) so the unified call is clean on 5.3-5.7; the existing `PRAGMA_DISABLE_DEPRECATION_WARNINGS` wrap is retained only to keep older engines warning-clean. Also fixed a latent extent bug in the call: passed the vertex extents `InMaxX`/`InMaxY` (= ComponentsX*ComponentSizeQuads) instead of the legacy branch's incorrect `ComponentsX-1`/`ComponentsY-1` (component counts), matching the heightmap array dimensions. Files: `Source/EditorAutomationRpcGateway/Private/Handlers/Environment/LandscapeHandler.cpp` (collapsed the 3-way version-gated apply block into one Import-based path). Test: added `FLandscapeCreateBuildsComponentGridTest` ("EditorAutomationRpcGateway.landscape.create.BuildsComponentGrid") to `Source/EditorAutomationRpcGateway/Private/Tests/World/TestEnvironmentHandlers.cpp` — drives the real `landscape.create` handler through its `AsyncTask(GameThread)` path to completion (shared-capture + game-thread pump), finds the spawned `ALandscape`, and asserts `LandscapeComponents.Num() > 0` (== 4 for the 2x2 grid), `ULandscapeInfo::XYtoComponentMap.Num() > 0`, and that `GetLandscapeExtent()` succeeds with a non-degenerate region. All three assertions fail under the reverted SetHeightData-only path even though create still reports success. Not yet compiled/tested (later phase).
- `#3-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 2 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
