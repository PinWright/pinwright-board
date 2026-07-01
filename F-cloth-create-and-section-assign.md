---
id: F-cloth-create-and-section-assign
title: "No verb creates a UClothingAsset or attaches one to a section; cloth authoring is impossible end-to-end"
status: IN-REVIEW
severity: Medium
category: feature
tags: [skeleton, cloth, chaos-cloth, skeletal-mesh, rpc-coverage]
---

# No verb creates a UClothingAsset or attaches one to a section; cloth authoring is impossible end-to-end

Setting up Chaos Cloth on a SkeletalMesh through the MCP is impossible
end-to-end. The two cloth verbs in `SkeletalMeshHandler.cpp` can only bind/list
an **already-existing** `UClothingAsset` — nothing in the namespace creates one,
and nothing attaches one to a chosen section. A mesh that ships with zero
clothing assets (the common case, e.g. `SK_DinoDragon`) therefore can never be
given cloth simulation via the API.

This ticket is the **C++ capability** work (a create-cloth verb + real
section-assign params). Its docs-process sibling
`E-cloth-verbs-wiki-overpromise-create-section` (category: ergonomic) owns
correcting the misleading verb summaries / adding `skeleton.md` wiki overlays and
stands on its own regardless of whether this feature lands — the same accepted
split as `F-ik-rig-retargeter-family-not-compiled` (capability) +
`E-ik-rig-family-wiki-advertises-compiled-out-workflow` (docs).

## The capability gap (verified against source)

- **No create path.** `skeleton.bind_cloth_to_skeletal_mesh`
  (`SkeletalMeshHandler.cpp` ~L848-868) never creates: with `clothAssetName` set
  it only looks up an existing asset by name and returns `CLOTH_NOT_FOUND` when
  absent; with the name omitted it just lists. There is no creation code path
  anywhere in the namespace. (`ClothingAssetFactory.h` is `#include`d under a
  `__has_include` guard but never used.)
- **No section-assign.** `skeleton.assign_cloth_asset_to_mesh` (~L920-960)
  declares **only** `skeletalMeshPath` and merely lists clothing assets. It
  accepts no `clothAssetName` and no `sectionIndex`, so it cannot attach anything
  to any section; passing those params is rejected `UNKNOWN_PARAMS`.

Net effect: the "create cloth → bind → share across sections" pipeline cannot be
performed; a mesh with 0 cloth assets stays at 0 both before and after.

## Fix

Add the capability (this ticket's scope; the summary/wiki wording correction is
the E-ticket's):

1. **Add `skeleton.create_cloth_from_section`** — author a NEW `UClothingAsset`
   from a mesh section via the editor's "Create Clothing Data from Section" path
   (`FClothingSystemEditorInterfaceModule::GetClothingAssetFactory()` →
   `UClothingAssetFactoryBase::CreateFromSkeletalMesh`, then
   `USkeletalMesh::AddClothingAsset`), and optionally bind it back to the source
   section so it simulates immediately. The clothing-editor modules are editor-only
   engine modules, so add them conditionally (like the IK Rig family) and report a
   clean error when they are unavailable rather than failing to compile.
2. **Give `skeleton.assign_cloth_asset_to_mesh` real
   `clothAssetName` + `sectionIndex` (+ `meshLodIndex`/`assetLodIndex`) params** so
   an existing named asset can be attached to / shared across specific sections,
   keeping the no-name path as the list behavior.

(Out of scope here: the `bind_cloth_to_skeletal_mesh` "Create and bind" summary
overpromise correction and wiki overlays — `E-cloth-verbs-wiki-overpromise-create-section`.
The earlier mention of a `skeleton.describe_mesh` "no cloth field" is dropped: no
such verb exists today; adding one is tracked by `F-rpc-mesh-describe-skeletal`.)

## Verbatim repro

Mesh: `/Game/ExampleContent/IKRig/Mesh/DinoDragon/SK_DinoDragon` (2 sections, 0
clothing assets).

1. `skeleton.bind_cloth_to_skeletal_mesh`
   args `{"skeletalMeshPath":"/Game/ExampleContent/IKRig/Mesh/DinoDragon/SK_DinoDragon","clothAssetName":"DinoDragon_BodyCloth","meshLodIndex":0,"sectionIndex":0,"assetLodIndex":0}`
   → `[CLOTH_NOT_FOUND] Cloth asset 'DinoDragon_BodyCloth' not found on mesh`
   (no verb creates the asset).

2. `skeleton.assign_cloth_asset_to_mesh`
   args `{"skeletalMeshPath":".../SK_DinoDragon","clothAssetName":"DinoDragon_BodyCloth","sectionIndex":1}`
   → `[UNKNOWN_PARAMS] Unknown parameter(s) for 'skeleton.assign_cloth_asset_to_mesh': [clothAssetName, sectionIndex]. Valid parameters: [skeletalMeshPath].`
   (the verb accepts neither an asset name nor a section index — it only lists).

## History
- `#1-initial-repro` `OPEN` reporter — On `SK_DinoDragon` (2 sections, 0 cloth assets): `bind_cloth_to_skeletal_mesh` with `clothAssetName` returns `CLOTH_NOT_FOUND` (never creates an asset despite "Create and bind" doc); `assign_cloth_asset_to_mesh` rejects `clothAssetName`/`sectionIndex` with `UNKNOWN_PARAMS` and only lists (despite "Attach to a specific section" doc). No verb creates a UClothingAsset, so cloth setup is impossible end-to-end and the two summaries overpromise. Confirmed against handler source (`SkeletalMeshHandler.cpp` ~L848-868 bind, ~L920-960 assign).
- `#2-reword-scope-to-capability` `OPEN` developer — Re-scoped to the pure C++ capability per the validity review: this F-ticket owns (a) a new create-cloth verb and (b) real section-assign params; the misleading-summary/wiki-overlay correction is carved out to `E-cloth-verbs-wiki-overpromise-create-section`, and the stale "`skeleton.describe_mesh` exposes no cloth field" aside (no such verb exists; tracked by `F-rpc-mesh-describe-skeletal`) is dropped. Title/body/Fix updated accordingly. No behavior change in this entry.
- `#3-capability-implemented` `IN-REVIEW` developer — Added `skeleton.create_cloth_from_section` and gave `skeleton.assign_cloth_asset_to_mesh` real `clothAssetName`+`sectionIndex` (+`meshLodIndex`/`assetLodIndex`) params. The create verb authors a new `UClothingAsset` via `FClothingSystemEditorInterfaceModule::GetClothingAssetFactory()` → `UClothingAssetFactoryBase::CreateFromSkeletalMesh`, then `USkeletalMesh::AddClothingAsset`, and optionally binds it to the source section (`bindToSection`, default true); it rejects a duplicate name (`CLOTH_NAME_IN_USE`) and degrades to `CLOTH_CREATE_UNSUPPORTED` when the engine's clothing-editor modules are absent. `assign_cloth_asset_to_mesh` now binds the named existing asset to the requested section (no-name path still lists). Files: `Source/PinWright/Private/Handlers/Animation/SkeletalMeshHandler.cpp` (new verb + extended assign + shared `FindClothAssetByNameSM` helper + `ClothingSystemEditorInterface` include guard `PINWRIGHT_HAS_CLOTH_CREATE`), `Source/PinWright/PinWright.Build.cs` (conditionally link `ClothingSystemEditorInterface` + `ClothingSystemEditor`, mirroring the IK Rig family). The `bind_cloth_to_skeletal_mesh` "Create and bind" summary correction stays with the E-ticket. Regression test: `Source/PinWright/Private/Tests/Assets/TestClothCreateAndSectionAssign.cpp` — asserts the create verb is registered with `skeletalMeshPath`+`clothAssetName`, that `assign_cloth_asset_to_mesh` declares `clothAssetName`+`sectionIndex`, and routes `{clothAssetName, sectionIndex}` through the real dispatcher proving it is no longer `UNKNOWN_PARAMS` (reaches the body → `MESH_NOT_FOUND` on a bogus path). Reverting any part fails the test.
