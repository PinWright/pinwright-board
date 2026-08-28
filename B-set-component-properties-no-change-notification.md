---
id: B-set-component-properties-no-change-notification
title: "actor.set_component_properties stores UPROPERTYs by reflection and fires no PostEditChangeProperty, so every property whose effect lives in that override is a silent no-op"
status: DONE
severity: High
category: bug
tags: [actor, components, property-write, derived-state, post-edit-change, water, exponential-height-fog, volumetric-fog, silent-noop, misleading-success]
encounters: 3
lastSeen: 2026-08-28T11:00:00+05:00
---

# The store is not the write

`actor.set_component_properties` applied every non-special-cased property through
`ApplyJsonValueToProperty` — a raw `FProperty` store into the component's memory — and
committed with `MarkRenderStateDirty()` + `UpdateComponentToWorld()` + `MarkPackageDirty()`
(`Handlers/Actor/ComponentHandler.cpp:320`, commit block `:328-333`). No
`FPropertyChangedEvent` was ever built, so **no class's `PostEditChangeProperty` override
ran**.

For a large class of engine properties that override *is* the property's behaviour. The
field is inert on its own. So the verb reported `applied`, the read-back returned the
requested value, the package went dirty — and nothing the property controls moved.

## The measurement

Assigning a `WaterBodyRiver`'s `WaterMaterial` to `None` and back through this verb, to
force the water Material Instance Dynamic to pick up a 2x `Scattering` change, rendered
**pixel-identical**: mean luma **0.4493** before and **0.4493** after. The same assignment
driven through Python `set_editor_property` rebuilt the MID (`WaterMID_0` → `WaterMID_3`)
and the identical change then moved mean luma to **0.4743**.

## Root cause, chain by chain (UE 5.8, `C:\UE_5.8`)

The missed work for the water case:

```
UWaterBodyComponent::PostEditChangeProperty   WaterBodyComponent.cpp:1368
  -> OnPostEditChangeProperty                                    :1244
  -> WaterMaterial branch                                        :1252
  -> UpdateMaterialInstances                                     :1019
  -> CreateOrUpdateWaterMID                                      :1057
  -> FWaterUtils::GetOrCreateTransientMID(WaterMID, ..., WaterMaterial)  :1062
```

`UpdateMaterialInstances` is also the **only** caller of
`MarkOwningWaterZoneForRebuild(EWaterZoneRebuildFlags::UpdateWaterMesh)` (`:1027`), so the
handler's `MarkRenderStateDirty()` cannot substitute for it: the water zone keeps rendering
the body from the stale MID no matter how often the component's own scene proxy is
recreated.

Why Python worked and the verb did not: `UObject.set_editor_property` routes through
`PropertyAccessUtil::SetPropertyValue_Object`, which builds a real change notify
(`PropertyAccessUtil.cpp:599`, `EPropertyChangeType::ValueSet`) and emits
`PostEditChangeChainProperty` (`:795`).

Why the omission is structural rather than a slip at one site: `ApplyJsonValueToProperty`
takes a `void* TargetContainer` (`Utils/PropertyImport.h:10`). It holds a memory address, not
a `UObject`, so it **cannot** notify even in principle. Only the call site can, and no call
site did.

## Scope — this is not a water bug

Engine classes whose `PostEditChangeProperty` is the only place the property's effect
happens, all reachable from this verb, all read out of UE 5.8 source:

| component | property | work skipped | source |
|---|---|---|---|
| `UWaterBodyComponent` | `WaterMaterial`, `WaterInfoMaterial`, `WaterStaticMeshMaterial`, `UnderwaterPostProcessMaterial` | MID regeneration + water-zone mesh rebuild | `WaterBodyComponent.cpp:1252-1261` |
| `UWaterBodyComponent` | `StaticMeshSettings`, `WaterZoneOverride`, `TargetWaveMaskDepth`, `LayerWeightmapSettings` | static-mesh components, zone reassignment, GPU wave data, weightmap | `:1262-1284` |
| `UShapeComponent` (Box/Sphere/Capsule) | `BoxExtent`, `SphereRadius`, `CapsuleHalfHeight`, … | `UpdateBodySetup()` — necessary but **NOT sufficient**, see "Shape extents: only half closed" below | `ShapeComponent.cpp:157-162` |
| `UPrimitiveComponent` | `LDMaxDrawDistance`, `bAllowCullDistanceVolume`, `bNeverDistanceCull` | `SetCachedMaxDrawDistance` — `CachedMaxDrawDistance` is what the renderer culls on | `PrimitiveComponent.cpp:1552-1561`, `:1601-1629` |
| `UPrimitiveComponent` | `bCanEverAffectNavigation` | `HandleCanEverAffectNavigationChange()` | `:1636-1639` |
| `UPrimitiveComponent` | **any** property | `IStreamingManager::Get().NotifyPrimitiveUpdated(this)` | `:1641` |
| `USkyLightComponent` | `SourceType`, `Cubemap`, `SourceCubemapAngle`, `CubemapResolution`, `LowerHemisphereColor` | `SetCaptureIsDirty()` / `SanitizeCubemapSize()` — the sky is never re-captured | `SkyLightComponent.cpp:570-579` |
| `UReflectionCaptureComponent` | `Cubemap`, `SourceCubemapAngle`, `ReflectionSourceType`, `bRuntimeCapture` | `MarkDirtyForRecapture()` | `ReflectionCaptureComponent.cpp:1108-1118` |
| `UChildActorComponent` | `ChildActorClass` | `DestroyChildActor(); CreateChildActor();` — **the old child actor stays in the level** | `ChildActorComponent.cpp:246-271` |
| `ULightComponent` | `Intensity`, `SpecularScale`, `DiffuseScale`, `LightFunctionMaterial` | clamping + `InvalidateLightingCache` (built lighting stays valid over a changed light) | `LightComponent.cpp:775-800` |
| `UStaticMeshComponent` | `OverrideMaterials`, `OverriddenLightMapRes`, `MaterialCacheTileCount` | HLOD dirty, `StreamingTextureData.Empty()`, material-cache texture recreate | `StaticMeshComponent.cpp:1994-2011` |
| `USplineMeshComponent` | `SplineParams` | `SetEndTangent` + HLOD cluster dirty | `SplineMeshComponent.cpp:1270-1288` |
| `USceneComponent` | `RelativeLocation/Rotation/Scale3D` | `FNavigationSystem::UpdateComponentData` (transform itself is covered by the handler's `UpdateComponentToWorld`) | `SceneComponent.cpp:693-697` |
| `UActorComponent` | **any** property | `ConsolidatedPostEditChange` → `UAssetUserData::PostEditChangeOwner` on every entry | `ActorComponent.cpp:1493-1497`, `:1477-1490` |

Same shape, not enumerated exhaustively: `UExponentialHeightFogComponent`,
`USkyAtmosphereComponent`, `UVolumetricCloudComponent`, `ULocalFogVolumeComponent`,
`UHeterogeneousVolumeComponent`, `USceneCaptureComponent`, `UPlanarReflectionComponent`,
`UAudioComponent`, `UBrushComponent`, `UPhysicsConstraintComponent`, `URadialForceComponent`,
`UCameraComponent`, `UVectorFieldComponent`, `ULevelInstanceComponent`,
`UCharacterMovementComponent` — every `::PostEditChangeProperty` override under
`Engine/Private/Components/` is reachable from this verb.

Explicitly **not** at risk, checked rather than assumed: `UDecalComponent` has no
`PostEditChangeProperty` override at all, and `USceneComponent::bVisible` is covered, because
`USceneComponent::OnVisibilityChanged` is `MarkRenderStateDirty()`
(`SceneComponent.cpp:3574-3586`) which the handler already calls.

## Shape extents: only half closed, and the two instruments disagree

**Correction to `#1`.** The first pass said a raw extent write leaves collision stale
*because `UpdateBodySetup()` is skipped*. The skip is real but the mechanism was wrong, and
the corrected version is worse rather than better:

`UShapeComponent::GetBodySetup()` calls `UpdateBodySetup()` **on the way past**
(`ShapeComponent.cpp:100-104`) — the same self-repairing-accessor pattern as
`UStaticMeshComponent::GetStaticMesh`. So `ShapeBodySetup->AggGeom` is not durably stale;
any reader repairs it. What is stale is the **live physics body**. The shapes in the Chaos
scene were built from `AggGeom` at `CreatePhysicsState()` time and nothing re-reads it. The
engine's own setter closes that separately:

```cpp
void UBoxComponent::SetBoxExtent(FVector NewBoxExtent, bool bUpdateOverlaps)  // BoxComponent.cpp:28-47
{
    BoxExtent = NewBoxExtent;
    UpdateBounds();
    MarkRenderStateDirty();
    UpdateBodySetup();
    if (bPhysicsStateCreated)
    {
        BodyInstance.UpdateBodyScale(GetComponentTransform().GetScale3D(), true);
        if (bUpdateOverlaps && IsCollisionEnabled() && GetOwner()) { UpdateOverlaps(); }
    }
}
```

The Details panel reaches the same end state by a different route: its `PreEditChange`
creates an `FComponentReregisterContext`, so `ConsolidatedPostEditChange` tears the
component down and back up and the physics state is rebuilt from the fresh `AggGeom`. **This
fix deliberately does not call `PreEditChange`** (that is the whole point — no flush, no
construction-script rerun), so it gets `UpdateBodySetup()` and nothing else.

Consequence, and it is a **measurement hazard, not just a rendering one**: after an extent
write through this verb,

- a **screenshot** shows the NEW extent — `UBoxComponent::CreateSceneProxy`
  (`BoxComponent.cpp:129`) reads `BoxExtent` and the handler's `MarkRenderStateDirty()`
  rebuilds the proxy;
- a **trace or overlap query** hits the OLD geometry — `UBoxComponent` declares no query
  override, so scene queries go through the standard `FBodyInstance` path against shapes
  created from the previous `AggGeom`.

Two instruments confidently disagreeing about the same actor. True before this fix and
still true after it. Remedy for callers today: use the typed shape setter, or force a
physics-state rebuild. A follow-up should decide whether this verb should call
`RecreatePhysicsState()` when a notified property is one the engine's own setter follows
with a body update — that is a real design question (cost, and which properties qualify),
not a one-liner, so it is **not** attempted here.

## Fix

`Utils/PropertyChangeNotify.h` (new) — `PinWright::NotifyPropertyChanged(UObject*, FProperty*)`
builds `FPropertyChangedEvent(Property, EPropertyChangeType::ValueSet)` and calls the virtual
`PostEditChangeProperty`. `ComponentHandler.cpp` calls it after each successful generic write
and reports a `notified[]` array beside `applied[]`.

Three decisions are load-bearing and are argued in the header:

- **Non-chain form only.** `UInstancedStaticMeshComponent::PostEditChangeChainProperty`
  dereferences `PropertyChain.GetActiveMemberNode()` with no null check
  (`InstancedStaticMesh.cpp:5638`), so a hand-built empty chain crashes the editor for any
  property outside its three known branches. `FPropertyChangedEvent(Property, ValueSet)` sets
  `MemberProperty = Property` (`UnrealType.h:6977-6985`), the shape every name-matched engine
  branch reads.
- **No `PreEditChange`.** It unregisters the component and calls `FlushRenderingCommands`
  (`ActorComponent.cpp:1327`, `:1336-1339`) — the cost `EnvironmentDirtyUtils.h:14-16` rejected,
  correctly. It is also the only thing that populates `EditReregisterContexts`, which is the
  sole trigger for `ConsolidatedPostEditChange` rerunning the owner's construction scripts
  (`:1437-1446`). Notifying alone therefore costs no flush, no re-registration, and cannot
  destroy the target out from under the loop. (The loop guards `IsValid(TargetComponent)`
  anyway.)
- **Engine-setter paths are not notified.** `Mobility`, `SimulatePhysics`, `StaticMesh` and
  `SkinnedAsset` already route through typed setters, which are supersets of the notification
  (`B-set-component-properties-staticmesh-shadow-corruption`). They appear in `applied`, never
  in `notified`, and a test asserts that absence.

## Deliberately NOT fixed here — the remaining doors into the same state

`rpc-design.md` §5b: fixing one entry point is not fixing the defect. These call the same
generic store with no notification and are **not** touched by this change. Each needs its own
judgement (some are pre-registration or pre-compile, where the defect does not apply):

- `actor.add_component` — `ComponentHandler.cpp:145`. Lower risk: `RegisterComponent()` runs
  *after* the writes and rebuilds most derived state. Not verified.
- `actor.set_blueprint_variables` — `ActorPropertyHandler.cpp:362`
- `water.set_water_body_underwater_post_process` — `WaterHandler.cpp:473`
- `environment.spawn_sky_atmosphere` / `spawn_volumetric_cloud` / `spawn_reflection_capture` —
  `EnvironmentSpawnHelpers.h:73`
- `blueprint.scs.set_property` — `PinWright_SCSHandlers.cpp:1460`, `:1500`, `:1514`
- `widget.set` / `widget.wrap` / `widget.import_xml` — `WidgetSetHandler.cpp:265`, `:283`, `:312`;
  `WidgetWrapHandler.cpp:53`, `:59`, `:75`; `WidgetXmlImportHandler.cpp:244`, `:272`
- `vehicle.*` — `ChaosVehicleHandler.cpp:62`, `:356`
- `blueprint.set_default` / `blueprint.add_variable` — `BlueprintPropertyHandler.cpp:489`, `:535`, `:200`
- `state_tree.add_*` — `StateTreeAuthoringHandler.cpp:157`, `:169`
- `behavior_tree.set_node_properties` / `attach_*` — `BehaviorTreeHandler.cpp:358`
- `animation.authoring.*` — `AnimGraphConstructionUtils.cpp:515`;
  `AnimationAuthoringHandler_AnimBlueprint.cpp:1950`, `:2877`;
  `AnimationAuthoringHandler_Sequence.cpp:810`
- `property.set` and the whole `container.*` family — `UtilityPropertyHandler.cpp:1129` and
  siblings. **These are the worst of the list**: they call the *bare* `UObject::PostEditChange()`,
  which builds an empty event, so `MemberPropertyName == NAME_None` and every
  `GET_MEMBER_NAME_CHECKED` branch is skipped — the same failure
  `B-landscape-set-material-stale-mics` documents. They look notified and are not.

## Related

- `B-property-set-container-empty-change-event` (OPEN) — **split out of this ticket's
  "deliberately NOT fixed" list**, because it is the camouflaged variant with a different
  fix and a wider blast radius: `property.set`, `property.reset` and 11 `container.*` verbs
  DO call a notification, but the bare `PostEditChange()` form, whose empty event matches no
  named branch. Must not be closed by association when this one closes.
- `B-set-component-properties-staticmesh-shadow-corruption` (DONE) — same handler. Its fix
  routed the three engine-wide **shadow-copy** properties to their typed setters and
  deliberately rejected `PostEditChangeProperty` for them, correctly: for those three the
  setter is a superset. That enumeration was complete for the shadow-copy class and says
  nothing about this one — the properties here have no reachable typed setter, and the
  notification is their entire effect.
- `B-landscape-set-material-stale-mics` (IN-REVIEW) — identical shape one domain over: a bare
  `PostEditChange()` whose empty event misses the name-matched branch.
- `B-spline-point-scale-water-derived` (IN-REVIEW) — the *converse* water defect (§5a): a value
  the engine re-derives. Notifying does not help there; refusing does.
- `rpc-design.md` §5d — the lesson, added with this fix.
- `water.set_water_body_material` remains the correct verb for water materials:
  `UWaterBodyComponent::SetWaterMaterial` calls `UpdateMaterialInstances()` itself
  (`WaterBodyComponent.cpp:282-289`).

## History
- `#1-water-mid-never-rebuilt` `OPEN` reporter — Measured on the production map: setting a `WaterBodyRiver`'s `WaterMaterial` to None and back through `actor.set_component_properties`, to force the water MID to pick up a 2x `Scattering` change, rendered pixel-identical (mean luma 0.4493 before and after); the same assignment through Python `set_editor_property` rebuilt the MID (`WaterMID_0` → `WaterMID_3`) and moved mean luma to 0.4743. Root cause traced to `ComponentHandler.cpp:320` storing through `ApplyJsonValueToProperty` and committing with `MarkRenderStateDirty()` + `UpdateComponentToWorld()` + `MarkPackageDirty()` (`:328-333`) without ever building an `FPropertyChangedEvent`, so `UWaterBodyComponent::PostEditChangeProperty` (`WaterBodyComponent.cpp:1368`) and its `WaterMaterial` → `UpdateMaterialInstances` → `CreateOrUpdateWaterMID` chain never ran. `MarkRenderStateDirty()` cannot substitute: `UpdateMaterialInstances` is also the only caller of `MarkOwningWaterZoneForRebuild` (`:1027`). The omission is structural — `ApplyJsonValueToProperty` takes a `void*` (`PropertyImport.h:10`) and cannot notify even in principle — so the audit above enumerates the whole reachable class from UE 5.8 source rather than the one property in the report. Not reproduced live in-session: the shared editor was in use by other agents, so the mechanism is read out of engine source at the lines cited and corroborated by the frame measurement supplied with the report.
- `#2-notify-on-the-generic-path` `IN-REVIEW` developer — Added `Utils/PropertyChangeNotify.h` (`PinWright::NotifyPropertyChanged`), which builds `FPropertyChangedEvent(Property, EPropertyChangeType::ValueSet)` and calls the virtual `PostEditChangeProperty`; `ComponentHandler.cpp` now calls it after each successful `ApplyJsonValueToProperty` and emits a `notified[]` array beside `applied[]`, because "the value is in the field" and "the class's change hook ran" fail independently and only the second one moves a derived-state property. Non-chain form only (the chain form crashes on ISM, `InstancedStaticMesh.cpp:5638`); no `PreEditChange` (it would flush rendering commands and, via `EditReregisterContexts`, rerun the owner's construction scripts — `ActorComponent.cpp:1327`, `:1336-1339`, `:1437-1446`); engine-setter paths deliberately excluded. Tests in `Tests/Actor/TestSetComponentPropertiesNotifies.cpp`: `ReflectionWriteRunsEngineNotification` writes `LDMaxDrawDistance` on a Movable probe and asserts `CachedMaxDrawDistance` followed it — a copy only `UPrimitiveComponent::PostEditChangeProperty` performs (`PrimitiveComponent.cpp:1552-1555`, `:1628`), so it fails on the pre-fix handler for the right reason, needs no RHI and no Water plugin; `EngineSetterPathIsNotNotified` asserts `Mobility` lands in `applied` and never in `notified`. **Not yet linked or suite-run.** Both touched translation units compile clean under `-SingleFile` (`ComponentHandler.cpp`, `TestSetComponentPropertiesNotifies.cpp`, UE 5.8, Result: Succeeded), which writes nothing into `Binaries/`; a full link and the automation suite were not run because a shared editor for this host project was live throughout with roughly a dozen agents attached, and linking requires that editor to exit. The remaining doors into the same state are enumerated above as deliberately-not-fixed and are what a tester should check are still separately tracked before this closes.
- `#3-shape-extent-correction-and-split` `IN-REVIEW` developer — Two corrections to `#1`, both from re-reading engine source rather than new measurement. (a) The `UShapeComponent` row overstated the fix: `UShapeComponent::GetBodySetup()` calls `UpdateBodySetup()` on the way past (`ShapeComponent.cpp:100-104`), so `AggGeom` self-repairs and is not durably stale; the real gap is the live physics body, which `UBoxComponent::SetBoxExtent` rebuilds via `BodyInstance.UpdateBodyScale(..., true)` when `bPhysicsStateCreated` (`BoxComponent.cpp:28-47`) and which the Details panel rebuilds via the `PreEditChange` reregister this fix deliberately avoids. Net: a shape extent written through this verb renders new and traces old — a screenshot and an overlap query disagree about the same actor, before AND after this fix. Written up under "Shape extents: only half closed" with the follow-up question (should a notified property whose engine setter follows with a body update also trigger `RecreatePhysicsState()`) left open rather than guessed at. (b) The `property.set` / `container.*` entry in the deliberately-not-fixed list has been split into its own ticket `B-property-set-container-empty-change-event`, because burying a camouflaged defect with a different fix inside another ticket is how it gets closed by association. No code change in this entry; the commit from `#2` is unchanged and still unlinked.
- `#4-fix-verified-live-on-exponential-height-fog` `IN-REVIEW` reporter — **Live confirmation of `#2`'s fix on a component class this ticket only inferred from source**, plus a correction to a downstream project's workaround. Measured 2026-08-27 on host project EAContentExamples58, level `/Game/Maps/Atlantis`, while tuning underwater fog depth. (a) The running editor build **does** carry `#2`: every `actor.set_component_properties` response in this session returned a `notified[]` array beside `applied[]` (e.g. `{"applied":["FogDensity"],"notified":["FogDensity"]}`), so the change is linked and live on this host — `#2` had recorded that a full link was never run. (b) `UExponentialHeightFogComponent` appears in this ticket only in the "same shape, not enumerated exhaustively" list, never measured. It is now measured, and the notification reaches the renderer: `FogDensity` 0.06 -> 0.6 through `actor.set_component_properties` on `ExponentialHeightFog_0` / `HeightFogComponent0` moved a fixed-pose, pinned-exposure frame (`render.capture_open_level`, location `{x:-9000,y:0,z:260}`, rotation `{pitch:-4,yaw:0,roll:0}`, fov 60, 768x768, `exposure {mode:"fixed", ev100:0}`) from `meanLuminance` **0.333654** to **0.275127** — against a no-change control pair shot at the same pose immediately before, which differed by **0.0017**. A 0.0585 signal against a 0.0017 floor is 35x, and **no `lighting.setup_volumetric_fog` flush was called between the write and the capture.** Six further density/emissive edits in the same session behaved identically. (c) The correction: that host project's own defect log had generalised an earlier `property.set` measurement into "fog component writes do not reach the renderer, call `lighting.setup_volumetric_fog` as a flush after **every** fog write", and was applying that workaround to `actor.set_component_properties` as well. The flush is unnecessary on this verb — the two verbs differ exactly as this ticket and `B-property-set-container-empty-change-event` predict, and conflating them costs a spurious RPC per edit and, worse, hides which verb is actually broken. Nothing here changes the ticket's own status or its deliberately-not-fixed list; it is evidence that the shipped fix does what `#2` claimed, on a second component class, under a pixel measurement with a control.
- `#5-verified-fixed-by-derived-state` `DONE` verifier — 2026-08-28. Verified on the editor rebuilt at
  `b79ba53e` by watching a **derived** value that only the engine's change hook can write, not by
  reading the property back (a read-back returns the reflection store and cannot fail) and not by
  trusting the new `notified[]` array (that is the diagnostic, not the behaviour). Probe was a
  throwaway `/Engine/BasicShapes/Cube` StaticMeshActor `PWVERIFY_NOTIFY_01` spawned into
  `/Game/Maps/Atlantis` and deleted afterwards — no shipping actor was written to, and the level was
  not saved. Made Movable first (`{"applied":["Mobility"]}` — note **no** `notified` entry, which is
  the documented engine-setter exclusion, observed rather than assumed) so
  `UPrimitiveComponent::PostEditChangeProperty` takes the direct
  `if (!bAllowCullDistanceVolume || Mobility != Static)` branch and the cull-volume path cannot
  contribute.
  The observable is `CachedMaxDrawDistance`: in the editor it is written **only** by
  `SetCachedMaxDrawDistance` inside `PostEditChangeProperty`, reached only through
  `if (PropertyThatChanged)` + `PropertyName == GET_MEMBER_NAME_CHECKED(UPrimitiveComponent,
  LDMaxDrawDistance)` (`PrimitiveComponent.cpp:1541-1560`, `:1600-1628`) — so a reflection store with
  no event, or with an event carrying a null `Property`, leaves it behind.
  Sequence, every step through `actor.set_component_properties` on `StaticMeshComponent0`, each read
  back with `actor.get_component_property`:
  `CachedMaxDrawDistance` **0** (baseline) -> write `LDMaxDrawDistance: 5000` -> **5000** -> write
  `LDMaxDrawDistance: 1234` -> **1234**. Two independent writes, so not a one-off.
  **Name-scoping control, to rule out the plugin simply copying the field itself:** forced
  `CachedMaxDrawDistance: 99` directly (`applied` + `notified`), then wrote `MinDrawDistance: 10`
  (also `applied` + `notified`) — `CachedMaxDrawDistance` stayed **99**, because `MinDrawDistance`'s
  branch only calls `MarkRenderStateDirty()` and never touches the cached distance. Then
  `LDMaxDrawDistance: 2222` -> **2222** again. So the event carries the *specific* property name and
  the engine picks the branch: a blanket recompute or a plugin-side copy would have restored 1234 at
  the `MinDrawDistance` step, and an empty/absent event would never have moved it at all.
  Together with `#4`'s independent pixel measurement on `UExponentialHeightFogComponent`, the fix is
  confirmed on two component classes by two different kinds of instrument.
  **Not closed by this, and stated so it is not buried:** (a) `B-property-set-container-empty-change-event`
  is verified separately, same day — this ticket's `notified[]` is not evidence for that verb;
  (b) the § *Shape extents: only half closed* finding is still true after the fix (a notified extent
  write rebuilds the render proxy but not the live physics body, so a screenshot and an overlap query
  still disagree) and it has **no ticket of its own** — I searched the board and found none. It is a
  distinct defect from the missing notification and should be filed rather than left inside a DONE
  ticket. (c) The § *Deliberately NOT fixed* list of other entry points is untouched and untested
  here; it is scoped to other verbs by construction.
- `#6-shape-extent-finding-now-tracked` `DONE` reporter — Cross-link only; status unchanged. The § *Shape extents: only half closed* finding that `#5` flagged as having no ticket of its own is now filed as **`B-shape-extent-stale-physics`** (OPEN, High). It was **reproduced live** rather than carried over from source reading: `actor.set_component_properties {BoxExtent:{400,400,400}}` on a throwaway `TriggerBox` returned `applied`+`notified`, `actor.get_bounding_box` and a fixed-pose capture both showed the new 800 cm box, and `spatial.raycast` returned the baseline-identical hit at the old 40-extent face — with a destroy/recreate-physics-state control that immediately moved the same ray to the new face, proving the ray was never the problem. A second write also established the worse direction this ticket did not have: **shrinking** leaves the body larger than the drawing, so a ray 150 cm outside the visible box still blocks. The follow-up design question this ticket left open (should a notified property whose engine setter follows with a body update trigger a body rebuild) travels with it, along with a third option — route shape extents to their typed setters the way `StaticMesh`/`SkinnedAsset` already are. Nothing here reopens this ticket: its own contract, the named notification, is verified and holds.
