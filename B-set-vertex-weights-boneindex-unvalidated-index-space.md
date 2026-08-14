---
id: B-set-vertex-weights-boneindex-unvalidated-index-space
title: "skeleton.set_vertex_weights takes an unvalidated boneIndex in an unspecified index space — no value the caller passes can be correct"
status: IN-REVIEW
severity: High
category: bug
tags: [skeleton, skin-weights, set-vertex-weights, index-space, validation, unusable-verb]
---

# set_vertex_weights: unvalidated boneIndex, and no correct value exists

`skeleton.set_vertex_weights` accepts a caller-supplied `boneIndex` per influence, writes it
straight into the weight buffer with no range check, and never states which index space it expects.

Line numbers as read at plugin HEAD `9da255f6d0ef5017cfddee03cd9458266c99d764`
(`Plugins/PinWright/`).

`Source/PinWright/Private/Handlers/Animation/SkeletalMeshHandler.cpp:612-615`:

```cpp
                                (*InfluenceObj)->TryGetNumberField(TEXT("boneIndex"), BoneIndex);
                                (*InfluenceObj)->TryGetNumberField(TEXT("weight"), Weight);

                                SkinWeight.InfluenceBones[InfluenceIndex] = static_cast<FBoneIndexType>(BoneIndex);
```

No validation against the reference skeleton's bone count, and none against the section's `BoneMap`
either. An out-of-range value is cast to `FBoneIndexType` and persisted; `Mesh.Build()` runs in the
same handler (`:196` via the shared scaffold), so it is baked into the re-chunked sections
immediately.

## The sharp point: the verb cannot be used correctly today

This is not "the caller must be careful". The buffer the handler writes into is **section-local**
— the scaffold seeds it via `SeedWeightEditSource` → `CaptureBaseSkinWeights`
(`SkinWeightTransferUtils.h:328-344`, `:297-313`), which memcpys `FSoftSkinVertex::InfluenceBones`
verbatim, and those slots index `FSkelMeshSection::BoneMap`. But the same buffer is then re-emitted
as **reference-skeleton** indices by `RebuildSourceModelInfluences`
(`SkinWeightTransferUtils.h:443-462`, `:459` assigns `VertInfluence.BoneIndex` from the raw slot),
which is what `Build()`'s re-chunking consumes.

So the caller's `boneIndex` has to be a section-local slot to be consistent with the surrounding
seeded influences it sits beside, and a reference-skeleton index to survive the rebuild.
**No single value is correct in both places.** On a single-section mesh the two spaces coincide and
the problem is invisible; on a multi-section mesh — which is what the section-local/RefSkeleton
mismatch ticket is about — every choice is wrong somewhere. The verb is unusable regardless of
caller diligence, and it fails silently in either direction.

The plugin's own newer audit code states the contract the mutator lacks —
`Source/PinWright/Private/Handlers/Animation/SkinAuditAnalysis.h:100-103`:

```cpp
    // One vertex's resolved skinning. Bone indices are REFERENCE-SKELETON indices, already
    // resolved through FSkelMeshSection::BoneMap by the caller — the raw InfluenceBones on a
    // soft vertex are SECTION-LOCAL and comparing those across sections silently compares two
    // different bones that happen to share a slot number.
```

## Fix

Two things, both required:

1. **Pin the index space in the contract.** Declare `boneIndex` as a reference-skeleton index (the
   space `skeleton.list_bones` returns and the space `SkinAuditAnalysis.h` uses), in the
   `RPC_PARAM` description, in the handler summary, and in the `docs/wiki-src/skeleton.md` overlay.
2. **Validate it and map it.** Reject out-of-range indices with a diagnostic error naming the bone
   count and the offending value, then map the accepted RefSkeleton index down to the section-local
   slot before writing `SkinWeight.InfluenceBones[]`, via
   `FSkeletalMeshLODModel::GetSectionFromVertexIndex` + a reverse lookup in `Section.BoneMap`
   (adding the bone to the section's `BoneMap` if absent). Without step 2 the declared contract is
   still violated by the buffer it writes into, so the two halves cannot be split.

Step 2 shares its machinery with `B-skin-weight-transfer-writes-section-local-bone-indices`; fixing
that ticket first makes this one a validation + doc change on top of a settled index space.

severity rationale: impact=silent wrong data on a normal path plus a hard blocker with no
workaround — the verb has no correct usage on a multi-section mesh and no validation to say so ×
reach=rare (skin-weight authoring) but it is the namespace's only hand-authoring entry point -> High

## Distinct from

- `E-set-vertex-weights-docs-bonename-vs-boneindex` (OPEN, Low) — covers the *wording* drift only:
  the summary says `(boneName, weight)` (`SkeletalMeshHandler.cpp:525`) while the param says
  `{boneIndex, weight}` (`:529`). Resolving that contradiction does not state which index space
  `boneIndex` lives in, and does not add validation. This ticket is the missing contract and the
  missing range check; that one is the typo. Fix them together if convenient.
- `B-skin-weight-transfer-writes-section-local-bone-indices` (OPEN, High) — the plugin-side index
  mistranslation on all four mutators. That is about indices the plugin itself produces; this is
  about indices the *caller* supplies, which the plugin never checks and never maps.

## History
- `#1-triage-unvalidated-unspecified-boneindex` `OPEN` reporter — Found during a mesh/skeletal authoring triage at plugin HEAD `9da255f6d0ef5017cfddee03cd9458266c99d764`. `skeleton.set_vertex_weights` reads `boneIndex` and assigns it to `SkinWeight.InfluenceBones[InfluenceIndex]` with a bare `static_cast<FBoneIndexType>` and no range check (`SkeletalMeshHandler.cpp:612-615`), and no doc anywhere states the expected index space. Source-confirmed the deeper problem: the destination buffer is seeded section-local (`SeedWeightEditSource` → `CaptureBaseSkinWeights`, `SkinWeightTransferUtils.h:328-344`, `:297-313`, memcpying `FSoftSkinVertex::InfluenceBones`, which index `FSkelMeshSection::BoneMap`) but re-emitted as reference-skeleton by `RebuildSourceModelInfluences` (`:443-462`, `:459`) for `Build()`'s re-chunking — so no caller-supplied value is correct in both spaces and the verb has no correct usage on a multi-section mesh. Invisible on single-section meshes where the two spaces coincide. The plugin's own audit code documents the contract the mutator violates (`SkinAuditAnalysis.h:100-103`). Fix = declare `boneIndex` as reference-skeleton, validate the range with a diagnostic error, and map to the section-local slot before writing; shares machinery with `B-skin-weight-transfer-writes-section-local-bone-indices`. Distinct from `E-set-vertex-weights-docs-bonename-vs-boneindex`, which is the summary-vs-param wording only. Not previously on the board.
- `#2-refskeleton-space-with-preflight` `IN-REVIEW` developer — Fixed by specifying the space and enforcing it. `skeleton.set_vertex_weights` now accepts `boneName` (preferred, wins when both are given) or `boneIndex` as a REFERENCE-SKELETON index — the same index `skeleton.list_bones` reports — and translates it into the section-local slot the profile storage requires via the new `SkinWeightTransferUtils::ResolveBoneToSectionLocalSlot`. That removes the ticket's core finding that no caller value could be correct in both places: the caller now names a bone in one stated space and the handler owns the crossing. Validation is an all-or-nothing PRE-FLIGHT that runs before the profile write and before `Build()`, so a rejected call cannot leave a half-authored profile: `BONE_NOT_FOUND` (unknown name, or `boneIndex` outside the reference skeleton), the new `BONE_NOT_IN_SECTION` (the bone exists but the section owning that vertex does not list it in its `BoneMap`, so no slot exists and one cannot be added without a re-chunk — registered in `Handlers/ErrorCodes.h` with that rationale), `INDEX_OUT_OF_RANGE` (`vertexIndex` past the LOD's section coverage — previously skipped in silence while reporting success), and `INVALID_ARGUMENT` for an empty influence list, more than `MAX_TOTAL_INFLUENCES` influences, a repeated bone, or a weight outside (0,1]. A zero weight is now rejected specifically because the influence arrays are packed largest-first and zero-TERMINATED, so a zero in the middle hid every influence after it. Influences are also repacked largest-first, which the packed layout requires and the old caller-order write did not guarantee. The response adds `boneIndexSpace: referenceSkeleton`, `verticesRequested` and `vertexCount`. Test `PinWright.skeleton.set_vertex_weights.RejectsUnusableBoneAndVertexIndices` drives all six rejections through the production handler on a two-section fixture with disjoint bone maps, and asserts none of them registered a profile or wrote LOD profile data. **NOT COMPILED, NOT RUN.**
