---
id: B-physics-asset-create-helper-no-disk-write
title: "Shared physics-asset creation helper only marks new assets dirty, while its two RPC callers report successful creation with no disk outcome"
status: IN-REVIEW
severity: High
category: bug
tags: [physics-asset, asset-utils, create, persistence, no-disk-write, false-success]
---

# Created PhysicsAssets are resident-only despite green RPC results

## What's wrong

The scoped root cause is `Utils/AssetUtils.cpp:627-668`:
`McpCreatePhysicsAssetFromSkeletalMeshHeadless` creates and populates a `UPhysicsAsset`, then calls
`McpSafeAssetSave`, which only marks dirty and registers the object. It returns the pointer without
any persistence outcome.

Its direct production consumers live in other scanners' areas and were read only to confirm the
failure path: `skeleton.create_physics_asset` (`Handlers/Animation/PhysicsAssetHandler.cpp:424-441`)
and `physics.setup_physics_simulation` (`Handlers/Physics/PhysicsHandler.cpp:276-300`) both publish
successful creation from that pointer with no `saved`/`pendingFlush` state. The latter can also
assign the dirty-only asset to a mesh. Restarting before a separate save loses the PhysicsAsset
and can leave the disk mesh without the reported assignment.

## What it should do

Return a structured creation/save outcome from the shared helper, persist with
`SaveAssetToDiskReportingPresence` when requested, and require both callers to include the
standard save report for the PhysicsAsset and any assigned mesh.

## Fix

`McpCreatePhysicsAssetFromSkeletalMeshHeadless` now returns `FPhysicsAssetCreateResult`, including
the created path, package, on-disk size, `EAssetSaveState`, and pending-flush verdict. It still
registers and marks the asset dirty, then uses `SaveAssetToDiskReportingPresence` when `save:true`
is requested; `save` is a new optional parameter defaulting to `true` on both RPC callers.

Both callers now emit the standard save report for the new PhysicsAsset. When the generated asset
is assigned to a SkeletalMesh, `skeletalMeshSave` carries the same measured report for the mesh
package. The new handler-harness test
`PinWright.physics.setup_physics_simulation.PersistenceWritesDiskAndRegistry` verifies the
created PhysicsAsset is both on disk and discoverable through the registry. Additional
behavioural coverage exercises
`PinWright.skeleton.create_physics_asset.PersistenceWritesDiskAndRegistry` and
`PinWright.physics.setup_physics_simulation.AssignedAssetSurvivesReload`, including durable
mesh assignment readback after reload. Wiki overlays in
`Plugins/PinWright/Docs/wiki-src/skeleton.md` and `physics.md` document the new save contract.
Changed files include `Source/PinWright/Private/Dispatch/SafePoint.cpp`,
`Source/PinWright/Private/Tests/Infra/TestHandlerTickSafetyRatchet.cpp`, and
`Source/PinWright/Private/Tests/Gameplay/TestPhysicsAssetFactoryModalHang.cpp` (its headless
coverage now opts into `save:false`), alongside the shared helper, both callers, and persistence
test listed above.
No editor, build, MCP, or automation run was performed per the source-only task boundary.

## Workaround

Immediately `asset.save` the returned PhysicsAsset and, when assigned, the SkeletalMesh.

## History
- `#1-pattern-scan` `OPEN` reporter — Root cause is in the assigned Utils area; outside-area callers were inspected only to confirm reachability. No editor, build, test, or RPC run.
- `#2-structured-physics-asset-save` `IN-REVIEW` developer — Shared helper and both callers now report measured durable-save outcomes; assigned meshes have a nested save report; added disk/registry handler coverage and wiki contract notes.
