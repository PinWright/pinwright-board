---
id: B-create-pose-library-noop-fake-success
title: "animation.authoring.create_pose_library returns success:true but creates no UPoseAsset (no-op stub)"
status: IN-REVIEW
severity: High
category: bug
tags: [animation, animation-authoring, pose-asset, pose-library, no-op, fake-success, silent-noop]
---

# `animation.authoring.create_pose_library` reports success but never creates the asset

`animation.authoring.create_pose_library` is documented (wiki + handler
summary) as "Create a pose library (pose asset)" with required `name` and
`skeletonPath`. The handler validates its inputs (loads + checks the
skeleton), then returns a **synthesized success envelope** carrying the
`assetPath` it *would* have created — but it never constructs, registers,
or saves a `UPoseAsset`. The result is a silent no-op: `success:true` plus
a real-looking `assetPath`, and nothing on disk or in the asset registry.

A programmatic caller that checks the `success` field (the normal contract)
proceeds as if the pose asset exists, then every downstream step
(`asset.exists`, `get_animation_info`, binding the pose to an AnimBP node)
fails with `ASSET_NOT_FOUND`. The "requires manual setup" hint is buried
*inside* the success result rather than surfaced as an error, so it does
not break the success check.

## What's wrong

`AnimationAuthoringHandler_AnimBlueprint.cpp:3071-3098` (the
`MCP_HAS_POSEASSET` branch, which is compiled in — the call returns the
success message, not `NOT_SUPPORTED`):

```cpp
USkeleton* Skeleton = AnimationAuthoringHelpers::LoadSkeletonFromPathAnim(SkeletonPath);
if (!Skeleton) { Ctx.SendError("SKELETON_NOT_FOUND", ...); return true; }

TSharedPtr<FJsonObject> Result = MakeShared<FJsonObject>();
Result->SetStringField(TEXT("assetPath"), Path / Name);   // synthesized, not created
Result->SetBoolField(TEXT("success"), true);
Result->SetStringField(TEXT("message"),
    FString::Printf(TEXT("Pose library '%s' creation requires manual setup"), *Name));
Ctx.SendSuccess(Result);
```

There is no `UPoseAssetFactory` / `IAssetTools::CreateAsset` call, no
`FAssetRegistryModule::AssetCreated`, and no package save. The handler
loads the skeleton purely to validate it, then fabricates a success
result.

## What it should do

Either:
1. **(preferred)** Actually create the asset: construct a `UPoseAsset`
   bound to the loaded skeleton (a fresh empty pose library is a valid
   asset — animators add poses afterward), register it with the asset
   registry, and save the package when `save` is true. Return
   `success:true` only after the asset is on disk / registered, so
   `asset.exists` on the returned `assetPath` is true. (Compare the
   already-working `create_ik_rig`, which does land a real
   `UIKRigDefinition` — `asset.exists` confirmed true in the same task.)
2. If creating an empty pose library truly cannot be supported headlessly,
   the call must **fail with a clear error** (e.g. `NOT_IMPLEMENTED` /
   `UNSUPPORTED`) instead of returning `success:true` with a synthesized
   `assetPath`. A no-op must never report success.

## Verbatim repro

Skeleton precondition (exists):

```
call("asset.exists", { "assetPath": "/Game/ExampleContent/IKRig/Mesh/DinoDragon/SK_DinoDragon_Skeleton" })
-> { "success": true, "exists": true, ... }
```

The no-op create:

```
call("animation.authoring.create_pose_library", {
  "name": "PA_DinoDragon_KeyPoses",
  "skeletonPath": "/Game/ExampleContent/IKRig/Mesh/DinoDragon/SK_DinoDragon_Skeleton",
  "path": "/Game/ExampleContent/IKRig/Animations",
  "save": true
})
-> { "assetPath": "/Game/ExampleContent/IKRig/Animations/PA_DinoDragon_KeyPoses",
     "success": true,
     "message": "Pose library 'PA_DinoDragon_KeyPoses' creation requires manual setup" }
```

The asset does not exist afterward:

```
call("asset.exists", { "assetPath": "/Game/ExampleContent/IKRig/Animations/PA_DinoDragon_KeyPoses" })
-> { "success": true, "exists": false, ... }

call("animation.authoring.get_animation_info", { "assetPath": "/Game/ExampleContent/IKRig/Animations/PA_DinoDragon_KeyPoses" })
-> [ASSET_NOT_FOUND] Could not load asset: /Game/ExampleContent/IKRig/Animations/PA_DinoDragon_KeyPoses
```

Editor is healthy (skeleton load + asset queries succeed; this is a
fake-success no-op, not a crash or hang). Replayed and confirmed
deterministically.

## History
- `#1-initial-repro` `OPEN` reporter — `animation.authoring.create_pose_library`
  returns `success:true` with a synthesized `assetPath` and message "Pose
  library '<name>' creation requires manual setup" but never creates,
  registers, or saves a `UPoseAsset`. Confirmed by replay: after the
  success result, `asset.exists` on the returned path is `false` and
  `get_animation_info` returns `ASSET_NOT_FOUND`. Root cause:
  `AnimationAuthoringHandler_AnimBlueprint.cpp:3090-3094` builds a success
  envelope with `assetPath = Path / Name` and the "requires manual setup"
  message immediately after loading/validating the skeleton — there is no
  factory/create/save call in the `MCP_HAS_POSEASSET` branch. The "manual
  setup" caveat lives inside the success result, so the standard `success`
  check passes and downstream steps fail with ASSET_NOT_FOUND. Fix: either
  actually create + save the UPoseAsset (so `asset.exists` on the returned
  path is true) or fail with a clear error code instead of fake-succeeding.
  This is the silent-success-with-no-effect family (cf.
  `B-create-morph-target-empty-not-persisted`,
  `B-misc-create-bookmark-silent-noop`). No existing board ticket covers
  `create_pose_library` — the IK-rig tickets
  (`F-ik-rig-retargeter-family-not-compiled`,
  `E-ik-rig-family-wiki-advertises-compiled-out-workflow`) are a separate
  capability/docs family about `create_ik_rig`/`add_ik_chain`. Severity
  High: it is the central deliverable of a reasonable pose-authoring task
  and fails silently rather than honestly.
- `#2-fix` `IN-REVIEW` developer — Implemented option 1 (create-real). The
  `MCP_HAS_POSEASSET` branch of `animation.authoring.create_pose_library`
  no longer fabricates a success envelope: after validating the skeleton it
  now `CreatePackage` + `NewObject<UPoseAsset>(RF_Public|RF_Standalone|RF_Transactional)`,
  `SetSkeleton(Skeleton)`, then `AnimationAuthoringHelpers::SaveAnimAsset`
  (MarkPackageDirty + `FAssetRegistryModule::AssetCreated`) so the asset is
  registered and `asset.exists` on the returned path is true. The returned
  `assetPath` is now the real `PoseAsset->GetPathName()` and the message
  reflects actual creation. `UPoseAssetFactory` is unusable headlessly (its
  `ConfigureProperties` pops a modal asset picker and `FactoryCreateNew`
  needs a `SourceAnimation`), so the handler replicates the factory's
  internal NewObject+SetSkeleton, mirroring the sibling `create_ik_rig` /
  `create_control_rig` in the same file. File:
  `Source/EditorAutomationRpcGateway/Private/Handlers/Animation/AnimationAuthoringHandler_AnimBlueprint.cpp`
  (the `MCP_HAS_POSEASSET` branch around the `create_pose_library`
  registration). Regression test added:
  `EditorAutomationRpcGateway.animation.authoring.create_pose_library.CreatesRealAsset`
  in `Source/EditorAutomationRpcGateway/Private/Tests/Gameplay/TestAnimationHandlers.cpp`
  — invokes the production handler with a transient skeleton, asserts
  success, then `StaticLoadObject(UPoseAsset::StaticClass(), …, returnedAssetPath)`
  returns a non-null asset whose `GetSkeleton()` equals the requested
  skeleton. Reverting the fix makes the load return null and the test fails.
  (Gated by `MCP_TEST_HAS_POSEASSET` = `__has_include("Animation/PoseAsset.h")`,
  the same guard the production branch uses.)
