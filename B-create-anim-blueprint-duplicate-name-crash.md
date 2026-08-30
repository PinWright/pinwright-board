---
id: B-create-anim-blueprint-duplicate-name-crash
title: "animation.authoring.create_anim_blueprint crashes the editor (Kismet2 check) when a UBlueprint of the same name is already loaded"
status: IN-REVIEW
severity: Critical
category: bug
tags: [animation, anim-blueprint, create, crash, assertion, duplicate-name, factory]
---

# create_anim_blueprint hits a fatal engine assertion on a name collision

`animation.authoring.create_anim_blueprint` calls
`UAnimBlueprintFactory::FactoryCreateNew(...)` directly with the raw
`FName(*Name)` and **never checks whether a `UBlueprint` of that name is
already present** under the target package. When one is, the engine's internal
`FKismetEditorUtilities::CreateBlueprint()` fires a hard `check()` and the
**entire editor crashes** (full fatal callstack + `CrashReportClientEditor`),
rather than returning a clean error.

The collision case that triggers it is mundane and easy for an agent to hit:
**create the same anim blueprint twice in one editor session** (or create over a
blueprint that is already loaded in memory — e.g. one this session already
created/compiled). The first call creates and loads the `UAnimBlueprint`; the
second call's `FindObject<UBlueprint>` finds the in-memory object and the
assertion fires.

## What's wrong

Handler (`AnimationAuthoringHandler_AnimBlueprint.cpp:521-536`) goes straight to
the factory with no pre-existence guard:

```cpp
FString PackagePath = Path / Name;
UPackage* Package = CreatePackage(*PackagePath);
...
UAnimBlueprintFactory* Factory = NewObject<UAnimBlueprintFactory>();
Factory->TargetSkeleton = Skeleton;
Factory->ParentClass = UAnimInstance::StaticClass();
UAnimBlueprint* NewAnimBP = Cast<UAnimBlueprint>(
    Factory->FactoryCreateNew(UAnimBlueprint::StaticClass(), Package,
                              FName(*Name), RF_Public | RF_Standalone,
                              nullptr, GWarn));   // <-- crashes here on collision
```

`FactoryCreateNew` → `FKismetEditorUtilities::CreateBlueprint()` asserts:

```
Assertion failed: FindObject<UBlueprint>(Outer, *NewBPName.ToString()) == 0
[File:...\Kismet2\Kismet2.cpp] [Line: 435]
```

Crash callstack (verbatim from `Saved/Logs/EAContentExamples57.log`):

```
FKismetEditorUtilities::CreateBlueprint()  Kismet2.cpp:435
UAnimBlueprintFactory::FactoryCreateNew()  AnimBlueprintFactory.cpp:466
AutoHandler_315_()  AnimationAuthoringHandler_AnimBlueprint.cpp:533
FRpcDispatcher::ProcessRequest()  RpcDispatcher.cpp:457
```

This is a tool bug: a crash brings down the editor and the MCP connection for
every subsequent call, far worse than a rejected input. A valid-looking,
idempotent-seeming "create this asset" request should never crash the host.

## Verbatim repro

1. `animation.authoring.create_anim_blueprint`
   `{ "name": "ABP_DinoDragon_Locomotion", "path": "/Game/ExampleContent/Animation", "skeletonPath": "/Game/ExampleContent/IKRig/Mesh/DinoDragon/SK_DinoDragon_Skeleton.SK_DinoDragon_Skeleton" }`
   → `{ "success": true, "message": "Animation Blueprint 'ABP_DinoDragon_Locomotion' created" }` (asset now loaded in memory).
2. Re-issue the **exact same call** (same name + path) in the same editor
   session → **socket closes / editor crashes**; log shows the
   `FindObject<UBlueprint>(Outer, ...) == 0` assertion at `Kismet2.cpp:435`.

Replay-confirmed twice on UE 5.7 (a warm session where the BP was already
loaded, then again on a freshly launched editor by creating-then-recreating in
the same session). Deterministic.

## Blast radius

Scoped to `animation.authoring.create_anim_blueprint`. The sibling
`animation.create_animation_bp` (`AnimationHandler.cpp:403-406`) routes through
`FAssetToolsModule::Get().CreateAsset(Name, SavePath, UAnimBlueprint::StaticClass(), Factory)`
instead of raw `FactoryCreateNew`, and AssetTools handles the duplicate-name
case without the assertion — so that entry does not crash. The other
`animation.authoring.*` creators that call `Factory->FactoryCreateNew` for
non-Blueprint asset types (sequence/montage/composite/blendspace) go through
different engine factories that do not hit the `CreateBlueprint` check; this
ticket is specifically about the AnimBlueprint path.

**Workaround:** Never call `create_anim_blueprint` with a name that already
exists in the session; use a fresh name, or use `animation.create_animation_bp`
which de-dupes the name safely.

**Fix:** Before constructing the factory, detect a collision and return a clean
error instead of crashing. Either (a) check
`FindObject<UBlueprint>(Package, *Name)` / `StaticFindObject` (and/or
`FindPackage` + asset-registry existence) and `SendError("ASSET_EXISTS", ...)`
when found, mirroring how `level.structure.create_level` returns
`[LEVEL_ALREADY_EXISTS]` for an in-memory `UWorld`
(see `B-create-level-saved-true-no-umap`); or (b) route creation through
`FAssetToolsModule::Get().CreateAsset(...)` like the sibling
`animation.create_animation_bp` so duplicate names are resolved by AssetTools
rather than the raw factory.

## History
- `#2-fix` `IN-REVIEW` developer — Added a pre-existence guard to the `animation.authoring.create_anim_blueprint` handler before the raw `UAnimBlueprintFactory::FactoryCreateNew` call, mirroring the AIHandler.cpp duplicate-name guard for the raw UBlueprintFactory path. The handler now checks `UEditorAssetLibrary::DoesAssetExist(AssetObjectPath) || LoadObject<UBlueprint>(...)` before creating the package, and `FindObject<UBlueprint>(Package, *Name)` after — returning `SendError("ASSET_EXISTS", ...)` on a collision instead of crashing inside `FKismetEditorUtilities::CreateBlueprint`'s fatal `check(FindObject<UBlueprint>(Outer, ...) == 0)`. Fix option (a) from the ticket. File: `Source/EditorAutomationRpcGateway/Private/Handlers/Animation/AnimationAuthoringHandler_AnimBlueprint.cpp` (note: this clone's tree nests the handler under `Handlers/Animation/`, not the `Handlers/` path the original report cited). Regression test `FAnimAuthoringCreateAnimBlueprintDuplicateNameNoCrashTest` (`EditorAutomationRpcGateway.anim.authoring.CreateAnimBlueprintDuplicateNameNoCrash`) added in `Source/EditorAutomationRpcGateway/Private/Tests/Assets/TestAnimGraphHandlers.cpp`: dispatches the create RPC twice with the same name+path through `FRpcDispatcher`; the first call must succeed, the second must return `bSuccess=false` with `ASSET_EXISTS` rather than crashing the host (reverting the guard crashes the suite or drops the error code, failing the test).
- `#1-initial-repro` `OPEN` reporter — `animation.authoring.create_anim_blueprint` calls `UAnimBlueprintFactory::FactoryCreateNew` with the raw `FName(*Name)` and no pre-existence check (`AnimationAuthoringHandler_AnimBlueprint.cpp:530-536`). When a `UBlueprint` of that name is already loaded in memory (e.g. creating the same anim BP twice in one session), the engine fires `check(FindObject<UBlueprint>(Outer, *NewBPName.ToString()) == 0)` at `Kismet2.cpp:435` and the whole editor crashes (fatal callstack + CrashReportClientEditor; MCP connection lost). Replay-confirmed twice on UE 5.7 — once in a warm session, once on a freshly relaunched editor by create-then-recreate in the same session; deterministic. Sibling `animation.create_animation_bp` routes through `FAssetToolsModule::CreateAsset` and does not crash. Proposed fix: guard the collision (return `ASSET_EXISTS`, mirroring `level.structure.create_level`'s `[LEVEL_ALREADY_EXISTS]`) or route through AssetTools::CreateAsset.
- `#3-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 2 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
