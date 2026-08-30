---
id: B-skeleton-copy-weights-noop-zero-fill
title: "skeleton.copy_weights transfers nothing — it zero-fills the target profile and reports success"
status: IN-REVIEW
severity: High
category: bug
tags: [skeleton, skin-weights, silent-noop, skeletal-mesh]
---

# skeleton.copy_weights transfers nothing — it zero-fills the target profile and reports success

`skeleton.copy_weights` is named, documented, and advertised as: *"Transfer skin weights
from a source SkeletalMesh to a target SkeletalMesh, matching by closest vertex … Useful
for porting weights to a remeshed asset."* In reality the handler **never reads a single
weight from the source mesh**. It creates the named skin-weight profile on the target, then
overwrites every target vertex with an **all-zero** `FRawSkinWeight` and returns success.
The caller is told the port succeeded while the profile it just "created" contains no
influences at all (a destructive zero-fill, not a copy).

In `SkeletalMeshHandler.cpp` (handler `skeleton.copy_weights`, around lines 418-446):

```cpp
FSkeletalMeshLODModel& SourceLOD = SourceModel->LODModels[LODIndex];
FSkeletalMeshLODModel& TargetLOD = TargetModel->LODModels[LODIndex];

FSkinWeightProfileInfo NewProfile;
NewProfile.Name = FName(*ProfileName);
TargetMesh->AddSkinWeightProfile(NewProfile);

FImportedSkinWeightProfileData& ProfileData = TargetLOD.SkinWeightProfiles.FindOrAdd(FName(*ProfileName));

uint32 VertsToCopy = FMath::Min(SourceLOD.NumVertices, TargetLOD.NumVertices);  // computed, never used
ProfileData.SkinWeights.SetNum(TargetLOD.NumVertices);

for (uint32 i = 0; i < TargetLOD.NumVertices; ++i)
{
    FMemory::Memzero(&ProfileData.SkinWeights[i], sizeof(FRawSkinWeight));  // zeroes — no copy from SourceLOD
}

TargetMesh->Build();
McpSafeAssetSave(TargetMesh);
// ... Result contains paths + profileName + lodIndex + a "note"; SourceLOD weights are never touched.
```

`SourceLOD` is dereferenced once for `VertsToCopy` and otherwise ignored; the closest-vertex
matching the docs promise does not exist. The result even ships a `note` that quietly admits
the real work was skipped: *"Skin weight profile created. Use
FSkinWeightProfileHelpers::ImportSkinWeightProfile for precise transfer."* — but that note is
not an error and a caller has no machine-readable signal (no `verticesCopied`, no failure
code) that the transfer was a no-op. Compare the sibling `skeleton.overwrite_weights`, which
returns a real `verticesModified` count.

**Impact:** a "port weights onto a remeshed body" task (the exact use case the tool
advertises) silently produces a SkeletalMesh carrying an empty, all-zero weight profile,
saved to disk, while the agent receives `ok:true` and reasonably believes the weights were
ported. There is no readback RPC for skin-weight-profile contents, so the emptiness is
undetectable through the API — the bug is fully masked.

**Workaround:** none through this RPC; real weight transfer requires
`FSkinWeightProfileHelpers::ImportSkinWeightProfile` (Python/C++), which the tool does not call.

**Fix:** actually copy from the source — for each of `VertsToCopy` target vertices, look up
the nearest source vertex and copy its `FRawSkinWeight` (the closest-vertex match the docs
describe), instead of memzeroing; and/or return a `verticesCopied` count so a no-op is
observable. If true closest-vertex transfer is out of scope, the handler should fail with a
clear error rather than report success on a zeroed profile.

## History
- `#1-initial-repro` `OPEN` reporter — Replayed `skeleton.copy_weights(sourceMeshPath=/Game/ExampleContent/AnimationRetargeting/Meshes/SK_M_MALE_Base, targetMeshPath=<duplicate of SK_S_MALE_Base>, profileName="PortedWeights", lodIndex=0)`. Returned `ok:true` with `{sourceMeshPath, targetMeshPath, profileName:"PortedWeights", lodIndex:0, note:"Skin weight profile created. Use FSkinWeightProfileHelpers::ImportSkinWeightProfile for precise transfer."}`. Source confirms (`SkeletalMeshHandler.cpp`, `skeleton.copy_weights`): the handler `FMemory::Memzero`s every target vertex's `FRawSkinWeight` and never reads `SourceLOD`'s weights — `VertsToCopy` is computed and discarded. The only existing unit test (`FSkelMeshCopyWeightsMissingParamsTest`) checks missing-param handling, not that any weights transfer. Silent success-with-no-effect: documented closest-vertex transfer never happens, and the saved profile is all-zero.
- `#2-fix` `IN-REVIEW` developer — Implemented the advertised closest-vertex transfer in `skeleton.copy_weights`. Replaced the `FMemory::Memzero` zero-fill loop: now gathers both LODs' `FSoftSkinVertex` arrays via `FSkeletalMeshLODModel::GetVertices`, and for each target vertex copies the nearest source vertex's `InfluenceBones`/`InfluenceWeights` into the target profile's `FRawSkinWeight` (extracted into `SkinWeightTransferUtils::CopyClosestVertexWeights`). Also rebuilds `FImportedSkinWeightProfileData::SourceModelInfluences` from the copied weights so the profile survives the `Build()` re-chunk, replaced the misleading `note` with a real `verticesCopied` count (mirroring sibling `set_vertex_weights`'s `verticesModified`), and returns `SendError(NO_SOURCE_WEIGHTS)` instead of fake-success when the source LOD has no vertices. Files: `Source/EditorAutomationRpcGateway/Private/Handlers/Animation/SkeletalMeshHandler.cpp`, new `Source/EditorAutomationRpcGateway/Private/Handlers/Animation/SkinWeightTransferUtils.h`. Regression test `FSkelMeshCopyWeightsClosestVertexTransferTest` (`EditorAutomationRpcGateway.skeleton.copy_weights.ClosestVertexTransfer` in `Tests/Gameplay/TestAnimationHandlers.cpp`) drives the production `CopyClosestVertexWeights` with two source vertices carrying distinct non-zero influences and asserts each target vertex inherits its nearest source's bones/weights — every assertion fails under the reverted zero-fill. Not compiled/tested here (later phase). Note: sibling `skeleton.mirror_weights` has the same zero-fill anti-pattern but is out of scope for this ticket.
- `#3-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 2 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
