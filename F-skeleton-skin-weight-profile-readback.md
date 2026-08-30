---
id: F-skeleton-skin-weight-profile-readback
title: "No RPC reads skin-weight-profile contents — a copy/normalize/set-weights result is unverifiable through the API"
status: IN-REVIEW
severity: Medium
category: feature
tags: [skeleton, skin-weights, weight-profile, readback, docs]
---

# No RPC reads skin-weight-profile contents — skin-weight writes are unverifiable

The `skeleton.*` namespace has a family of skin-weight **mutators**
(`skeleton.copy_weights`, `skeleton.normalize_weights`, `skeleton.set_vertex_weights`,
`skeleton.prune_weights`, `skeleton.auto_skin_weights`, `skeleton.mirror_weights`) but **no
corresponding read** for skin-weight-profile data. After a write a caller cannot ask:
*which skin-weight profiles exist on this mesh, how many vertices/influences does a
named profile hold, do the per-vertex influences sum to 1.0?* The only read that touches
a `USkeletalMesh` is `skeleton.describe_mesh`, whose dump-parity shape is geometry/
materials/LODs only — it has **no `skinWeightProfiles` field** (confirmed in
`docs/wiki-src/skeleton.md`: returns `bounds`/`materials`/`lods`/`*ByLod`/`physicsAsset`/
`skeleton`, nothing about weight profiles). `skeleton.get_info` only summarises the bound
`USkeleton` (bone/socket/virtual-bone counts).

The consequence is a **verification dead end**: any skin-weight authoring task whose
acceptance criterion is "confirm the profile was written / weights renormalized" cannot
be closed through the API. The caller must trust each mutator's own echoed success result
— which is exactly how the masking in `B-skeleton-copy-weights-noop-zero-fill` (a
zero-fill that reports `ok:true`) stays undetectable.

## What it should do

Add a read-only RPC — recommended `skeleton.describe_skin_weights` (or
`skeleton.list_skin_weight_profiles`) — that, given `skeletalMeshPath` (and optional
`profileName`, `lodIndex`), reports the skin-weight-profile state of the mesh:

- the profile names present (any `FSkinWeightProfileInfo` entries from
  `USkeletalMesh::GetSkinWeightProfiles()`, each cross-referenced against the editor-only
  `LODModel.SkinWeightProfiles` map keyed by the same `FName`);
- per profile/LOD: vertex count, max influences per vertex, and a cheap validity summary
  (e.g. count of vertices whose influence weights sum to ~1.0 vs. zero/degenerate) so a
  zero-filled or un-normalized profile is observable;
- ideally a small per-vertex influence sample (first N vertices) for spot-checking,
  consistent with the inline-budget/`limit` conventions used by the other verbose
  `skeleton.*` readers.

Implementation note: profile *names* are also reachable today through the generic
`property.get` RPC (the `SkinWeightProfiles` array is a reflected `UPROPERTY`), but the
actual per-vertex influence data the validity summary needs lives in the non-reflected,
editor-only `FImportedSkinWeightProfileData.SkinWeights` and is **not** reachable that way —
so a dedicated reader is still required. The new method's docs should mention the
`property.get` name-only shortcut.

This restores the asset-write → asset-read verification round-trip that the other
namespaces already provide via their readbacks (cf. `F-sequencer-track-state-readback`,
`E-niagara-modify-parameter-no-override-readback`, and the many other "no readback"
tickets), and makes the skin-weight mutators independently testable.

**Docs (`docs/wiki-src/skeleton.md`):** document the new method, and in the
`### skeleton.describe_mesh` section note explicitly that it does **not** report
skin-weight profiles and point at the new readback for that.

## Distinct from

- `B-skeleton-copy-weights-noop-zero-fill` (IN-REVIEW, bug) — that ticket is the *defect*
  (copy_weights zero-fills and reports success) and adds a `verticesCopied` count on
  the **mutator**. This ticket is the missing **reader**: even a correctly-functioning
  copy/normalize/set leaves no way to read the resulting profile back. The two are
  complementary — fixing the mutator's echo only signals the immediate caller and requires
  trusting the mutator's self-report; it gives no way to inspect an already-saved mesh's
  weight profiles independently. Only a readback RPC does that.
- `F-rpc-mesh-describe-skeletal` (DONE) — added `skeleton.describe_mesh` but deliberately
  scoped to the geometry/material/LOD dump shape; skin-weight profiles were out of scope.

## History
- `#2-reword-and-implement` `IN-REVIEW` developer — Reworded: corrected the two non-existent mutator names in the body (`skeleton.overwrite_weights` → `skeleton.set_vertex_weights`, `skeleton.auto_weights` → `skeleton.auto_skin_weights`), updated the `B-skeleton-copy-weights-noop-zero-fill` cross-reference from OPEN to IN-REVIEW (it uses `verticesCopied`, not a proposal), and added the `property.get`-for-names workaround note (names are reflection-reachable; the per-vertex `FImportedSkinWeightProfileData.SkinWeights` validity data is not). Implemented the readback: new RPC `skeleton.describe_skin_weights` enumerates `USkeletalMesh::GetSkinWeightProfiles()` and, per LOD, cross-references the editor-only `FSkeletalMeshLODModel::SkinWeightProfiles` map and summarizes each profile's `SkinWeights` into `vertexCount` / `maxInfluencesPerVertex` and a sum-to-1.0 validity breakdown (`normalizedVertexCount` / `zeroWeightVertexCount` / `degenerateVertexCount`), plus an optional first-N `sample` (`profileName`/`lodIndex`/`sampleCount` filters; errors `MESH_NOT_FOUND`/`INVALID_LOD`/`PROFILE_NOT_FOUND`). The validity/sample math is a pure, unit-testable function `SkinWeightTransferUtils::SummarizeSkinWeights` (+ `FSkinWeightProfileSummary`/`FSkinWeightVertexSample`) sharing the existing `RawWeightToFloat` fixed-point conversion, so a zero-filled profile (the masked B- defect) is observable. Files: `Source/EditorAutomationRpcGateway/Private/Handlers/Animation/SkeletalMeshHandler.cpp`, `Source/EditorAutomationRpcGateway/Private/Handlers/Animation/SkinWeightTransferUtils.h`, `docs/wiki-src/skeleton.md` (new method section + a `describe_mesh` cross-note that it omits skin weights). Regression test `FSkelMeshDescribeSkinWeightsSummaryClassifiesValidityTest` (`EditorAutomationRpcGateway.skeleton.describe_skin_weights.SummaryClassifiesValidity` in `Tests/Gameplay/TestAnimationHandlers.cpp`) drives the production `SummarizeSkinWeights` over a normalized / zero-filled / single-full / degenerate vertex set and asserts the three validity buckets stay distinct (a zero-fill is not miscounted as normalized) plus the sample honors `sampleCount`; a missing-param test covers handler registration. Not compiled/tested here (later phase). Aspect-version bump: not needed — the new method is a live read, and `BuildSkeletalMeshJson` (the `skeletal_mesh` dump sidecar) is unchanged.
- `#1-initial-audit` `OPEN` reporter — PROCESS friction from the `skeleton.copy_weights` weight-port task (focus `skeleton.copy_weights`; the judge filed the zero-fill *bug* as `B-skeleton-copy-weights-noop-zero-fill`). Distinct angle: the task's step-7 acceptance ("confirm copy_weights actually wrote the PortedWeights profile and the mesh is still valid") was **unsatisfiable through the API**. Friction note verbatim: "the success criterion expected a skin-weight-profile readback to surface 'PortedWeights', but describe_mesh's dump shape has no skin-weight-profiles field and the skeleton namespace has no dedicated profile/skin-weight readback method, so profile creation is confirmed only by copy_weights' own success result; copy_weights also returns no count of vertices/influences transferred." Verified against source-of-truth: `docs/wiki-src/skeleton.md` shows `skeleton.describe_mesh` returns bounds/materials/lods/*ByLod/physicsAsset/skeleton only (no `skinWeightProfiles`), and a board grep finds no skin-weight-profile readback ticket. Proposes a read-only `skeleton.describe_skin_weights` reporting profile names + per-profile vertex/influence counts + a sum-to-1.0 validity summary, plus a wiki cross-note on `describe_mesh`. Call counts: 19 calls, all `ok:true`, no retries / no python fallback — friction was a verification dead end, not an execution failure.
- `#3-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 2 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
