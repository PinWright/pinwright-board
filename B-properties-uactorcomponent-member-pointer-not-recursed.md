---
id: B-properties-uactorcomponent-member-pointer-not-recursed
title: "Actor member UActorComponent pointers dumped as null instead of recursed"
status: DONE
severity: High
category: bug
tags: [properties, components, recursion]
---

# Actor member UActorComponent pointers dumped as null instead of recursed

Distinct from [B-asset-dump-instanced-subobjects-not-recursed](B-asset-dump-instanced-subobjects-not-recursed.md) (DONE — that fixed `TObjectPtr<>` instanced properties in data assets like Enhanced Input modifiers). The remaining gap: **native actor member-variable pointers to UActorComponent subclasses** (created via `CreateDefaultSubobject` in C++ or via SCS in a Blueprint) dump as `value: null` instead of recursing into the component's property state.

Result: component transforms, mesh references, collision settings, visibility, attach-parent info, and child component structure are completely missing from the actor's `properties.json`.

## Samples

- `Game/Blueprints/GamePlay/BP_IndustrialRobot/properties.json` — DefaultSceneRoot, Box, IndustrialRobot, SparksV2 all `value: null`
- `Game/Blueprints/Pawn/PW_Fly/properties.json` — ArrowComponent and nested subobjects null
- `Game/Blueprints/Data/AC_TilleArray/properties.json` — HSM, Source components null

## MCP verification

`call("blueprint.inspect", {assetPath: "/Game/Blueprints/GamePlay/BP_IndustrialRobot"})` returns the components with full property state; the asset dump drops everything except the field's existence.

## Fix

In `PropertyUtils.cpp` `FObjectProperty` branch: when the property is a `UActorComponent` subclass and `GetObjectPropertyValue_InContainer` returns null on a CDO, walk the owner's class chain looking for a `UBlueprintGeneratedClass` whose `SimpleConstructionScript->FindSCSNode(Property->GetFName())` resolves to an SCS node, then recurse on `Node->GetActualComponentTemplate(LeafBPGC)` (the ICH-aware archetype). The existing `IsOwnedComponentSubobjectForSerialization` ownership check still guards the non-null CDO case for native-CDS components; the new path only fires when the CDO field is null because the BPGC populates the slot at runtime construction, not at CDO build.

## History
- `#2-owned-component-recursion` `IN-REVIEW` implementer — `PropertyUtils.cpp` now expands owned/archetype component template pointers without broadening arbitrary external component references. Added utility coverage for owned vs external component pointers.
- `#1-component-pointer-null` `OPEN` reporter — distinct from the data-asset instanced-subobject fix; affects actor BP and component-bearing native classes. Loses transforms / meshes / collision / visibility. Sample paths in body.
- `#3-verify-still-null` `OPEN` tester — Returned: fresh `asset.dump` on `/Game/Blueprints/GamePlay/BP_IndustrialRobot` still shows `Box`, `DefaultSceneRoot`, `IndustrialRobot`, `SparksV2` as `value: null`; `/Game/Blueprints/Pawn/PW_Fly` still shows local `UCameraComponent*`, `UAudioComponent*`, `USceneCaptureComponent2D*`, `USpringArmComponent*` as `value: null`. Inherited `TObjectPtr<UArrowComponent>` from Character recurses fine, so the gap is specifically the raw `U*Component*` pointer form on BP-added owned components. Test: `call("asset.dump", {assetPath: "/Game/Blueprints/GamePlay/BP_IndustrialRobot"})` then read `properties.json`.
- `#4-scs-component-template-recursion` `IN-REVIEW` developer — `PropertyUtils.cpp` `FObjectProperty` branch now walks the BPGC chain via `SimpleConstructionScript->FindSCSNode(Property->GetFName())->GetActualComponentTemplate(LeafBPGC)` when `GetObjectPropertyValue_InContainer` returns null on a CDO for a `UActorComponent`-subclass property. This recovers BP-added component templates (Box, DefaultSceneRoot, CameraComponent, etc.) that the CDO doesn't carry directly because the SCS populates them at instance construction. Existing inherited/native-CDS recursion paths unchanged. Added regression test `FPropertyUtilsSCSAddedComponentTemplateExpandsOnCDOTest` (`Tests/Utility/TestPropertyComponentParentDefault.cpp`). `properties.json` aspect version bumped in `AssetDumpCache.cpp` (centralized sprint bump).
- `#5-verify-scs-recursion` `DONE` tester — Verified: fresh `asset.dump` on `/Game/Blueprints/GamePlay/BP_IndustrialRobot` now emits `Box`, `DefaultSceneRoot`, `IndustrialRobot`, `SparksV2` with full `value: { ... }` bodies (transform, attach, visibility, mesh refs) instead of `value: null`. Confirmed on second sample `/Game/Blueprints/Pawn/PW_Fly`: `UCameraComponent*`, `UAudioComponent*`, `USceneCaptureComponent2D*`, `USpringArmComponent*` raw pointer fields now recurse. Remaining `value: null` entries on PW_Fly are `APawn*` and `UUserWidget*` — runtime refs, out of scope.
