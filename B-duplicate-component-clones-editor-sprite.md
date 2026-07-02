---
id: B-duplicate-component-clones-editor-sprite
title: "actor.duplicate_component clones the engine's editor-only light sprite, producing redundant BillboardComponents"
status: IN-REVIEW
severity: Medium
category: bug
tags: [duplicate-copies-editor-sprite]
encounters: 1
lastSeen: 2026-07-02T09:33:28.1850131+03:00
---

# actor.duplicate_component clones the engine's editor-only light sprite, producing redundant BillboardComponents

`actor.duplicate_component` with the default `duplicateChildren:true` recursively
copies **every** attach-child of the source component, including the transient,
engine-auto-managed **editor-only `UBillboardComponent` sprite** that
`ULightComponentBase::OnRegister` auto-creates for any light component in the
editor. That sprite is not a user-authored child — the engine regenerates a fresh
one whenever a light component registers — so copying it is both unwanted and
redundant: the newly-duplicated light auto-creates its **own** sprite on register,
and duplicate_component *additionally* clones the source's sprite. Each duplicated
light therefore ends up with **two** overlapping billboards (one auto-created, one
copied) instead of one, and the copied one is a persistent instance component
serialized into the level.

Downstream effects: on a scene-dressing task that adds one point light and
duplicates it 3x, the actor accumulates 12 components (1 mesh + 4 lights + 4
auto-sprites + 3 copied sprites) instead of the expected 9, which spilled
`actor.get_components` to a file and forced the caller to filter to the 4 real
`PointLightComponent`s. On repeated duplication the redundant editor sprites
accumulate without bound.

## What it should do

When recursing children for `duplicateChildren`, `DuplicateActorComponent` should
**skip engine-auto-managed editor-only visualization components** (transient /
`bIsEditorOnly` sprite billboards such as the light/audio sprite the engine
auto-creates on register). The duplicated component regenerates its own sprite on
`RegisterComponent()`, so the copy is always redundant. A user-authored
`UBillboardComponent` deliberately attached by the caller is a different case, but
the engine's auto sprite (created in `ULightComponentBase::OnRegister`) is not
part of the authored component graph and must not be cloned.

## Guilty source

`Plugins/PinWright/Source/PinWright/Private/Handlers/Actor/ComponentDuplicateHandler.cpp:198-221`
— the child-recursion loop copies all attach-children with no editor-only/transient filter:

```cpp
if (Options.bDuplicateChildren)
{
    if (USceneComponent* SourceScene = Cast<USceneComponent>(SourceComponent))
    {
        USceneComponent* NewSceneParent = Cast<USceneComponent>(NewComponent);
        for (USceneComponent* SourceChild : SourceScene->GetAttachChildren())
        {
            if (!SourceChild || SourceChild->GetOwner() != Actor)
            {
                continue;
            }
            // no skip for SourceChild->IsEditorOnly() / bIsEditorOnly / transient auto-sprites
            FString ChildError;
            FActorComponentDuplicateOptions ChildOptions = Options;
            ChildOptions.DesiredName.Reset();
            ChildOptions.TargetParent = NewSceneParent;
            UActorComponent* NewChild = DuplicateActorComponent(Actor, SourceChild, ChildOptions, DuplicateContext, ChildError);
            ...
```

The new light's auto-sprite is created at `NewComponent->RegisterComponent()`
(`ComponentDuplicateHandler.cpp:187-190`), so both sprites exist after the call.

## Verbatim repro (replayed at HEAD via mcp__pinwright__call)

1. `actor.spawn` `{meshPath:"/Engine/BasicShapes/Cylinder", actorName:"ReplayCandelabra_A"}` → ok.
2. `actor.add_component` `{actorName:"ReplayCandelabra_A", componentType:"PointLightComponent", componentName:"ReplaySource", properties:{Intensity:350, AttenuationRadius:240}}` → ok.
3. `actor.get_components` `{actorName:"ReplayCandelabra_A"}` → **3** components: `StaticMeshComponent0`, `ReplaySource` (PointLightComponent), and an auto-created `BillboardComponent_0` (`/Script/Engine.BillboardComponent`). Confirms the engine attaches a sprite to the light.
4. `actor.duplicate_component` `{actorName:"ReplayCandelabra_A", sourceName:"ReplaySource", newName:"ReplayDup1", copyProperties:true}` → ok, and the response `mapping` explicitly shows the sprite was cloned:
   `"mapping":{"ReplaySource":"ReplayDup1","BillboardComponent_0":"BillboardComponent_0_Copy_0"}`
5. `actor.get_components` → **6** components: `StaticMeshComponent0`, `ReplaySource`, `BillboardComponent_0`, `ReplayDup1` (the duplicate light), `BillboardComponent_1` (the duplicate's own **auto-created** sprite), and `BillboardComponent_0_Copy_0` (the **redundant copied** sprite). One duplicated light → two billboards.

## Workaround

Pass `duplicateChildren:false` when duplicating a light (or any sprite-bearing
component) — the new component still gets its own auto-sprite on register, and no
redundant copy is made. Or ignore/filter the extra `BillboardComponent_*_Copy_*`
entries in reads.

severity rationale: impact=redundant persistent editor-only component written on every light/sprite-bearing duplicate (a wrong write that accumulates and spills reads, not pure cosmetics) × reach=targeted duplicate verb (not every-session) -> Medium

## History
- `#1-initial-repro` `OPEN` reporter — Replayed at HEAD: adding one PointLightComponent auto-creates `BillboardComponent_0` (editor sprite); `actor.duplicate_component` with default `duplicateChildren:true` clones it (`mapping` shows `BillboardComponent_0 -> BillboardComponent_0_Copy_0`) while the new light also auto-creates its own `BillboardComponent_1`, leaving two billboards per duplicated light. Child-recursion loop at `ComponentDuplicateHandler.cpp:198-221` has no editor-only/transient filter. Fix: skip engine-auto-managed editor-only sprite children when recursing.
- `#2-fix` `IN-REVIEW` developer — Verified against plugin + engine source: `USceneComponent::CreateSpriteComponent` (engine `SceneComponent.cpp:879-909`) creates the sprite as `NewObject<UBillboardComponent>(GetOwner(), ... RF_Transient|RF_TextExportTransient)`, `SetIsVisualizationComponent(true)`, `SetupAttachment(this)`, only when `!GetWorld()->IsGameWorld()` — so it IS an attach-child owned by the actor that passes the recursion's sole `GetOwner()!=Actor` guard. Fixed by adding an editor-only/visualization skip to the child-recursion loop in `ComponentDuplicateHandler.cpp` (`if (SourceChild->IsVisualizationComponent() || SourceChild->IsEditorOnly()) continue;`), mirroring the engine's own idiom (`DataprepEditingOperations.cpp`: "Skip components which are either editor only or for visualization"). Files: `Plugins/PinWright/Source/PinWright/Private/Handlers/Actor/ComponentDuplicateHandler.cpp`. Regression test `PinWright.actor.duplicate_component.SkipsEditorOnlyLightSpriteChild` (in `Private/Tests/World/TestActorDuplicateComponentHandler.cpp`) builds an in-code fixture (bare `AActor` + `UPointLightComponent`), asserts the source light auto-created exactly one visualization billboard child (fixture sanity → FAIL if absent, not skip), then asserts the duplicate has exactly one sprite child (its own auto-sprite, no clone) and the response `mapping` carries no `Billboard*` entry — both fail if the skip is reverted. Compiles clean.
