---
id: B-physics-asset-create-helper-no-disk-write
title: "Shared physics-asset creation helper only marks new assets dirty, while its two RPC callers report successful creation with no disk outcome"
status: OPEN
severity: High
category: bug
tags: [physics-asset, asset-utils, create, persistence, no-disk-write, false-success]
---

# Created PhysicsAssets are resident-only despite green RPC results

## What's wrong

The scoped root cause is `Utils/AssetUtils.cpp:582-615`:
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

## Workaround

Immediately `asset.save` the returned PhysicsAsset and, when assigned, the SkeletalMesh.

## History
- `#1-pattern-scan` `OPEN` reporter — Root cause is in the assigned Utils area; outside-area callers were inspected only to confirm reachability. No editor, build, test, or RPC run.
