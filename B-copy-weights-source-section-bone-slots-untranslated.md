---
id: B-copy-weights-source-section-bone-slots-untranslated
title: "skeleton.copy_weights stores the SOURCE mesh's section-local bone slots on the target — every influence lands on a different bone"
status: IN-REVIEW
severity: High
category: bug
tags: [skeleton, skin-weights, copy-weights, multi-section, index-space, silent-corruption]
---

# copy_weights copies source-section bone slots onto a target with different sections

`SkinWeightTransferUtils::CopyClosestVertexWeights` matches each target vertex to its nearest
source vertex and then memcpys that source vertex's influence arrays verbatim:

```cpp
    FMemory::Memcpy(TargetWeight.InfluenceBones, MatchedSource.InfluenceBones, sizeof(...));
    FMemory::Memcpy(TargetWeight.InfluenceWeights, MatchedSource.InfluenceWeights, sizeof(...));
```

`FSoftSkinVertex::InfluenceBones` holds **section-local slots** — indices into
`FSkelMeshSection::BoneMap` — and those slots belong to the **SOURCE** mesh's sections. The
target's sections have their own, independently chunked bone maps. Storing the source slot
unchanged means slot *N* of some source section is read as slot *N* of whatever target section
owns the target vertex, which is a different real bone whenever the two bone maps differ.

This is a distinct defect from `B-skin-weight-transfer-writes-section-local-bone-indices`. That
ticket is about the section-local → reference-skeleton crossing on the way OUT to
`SourceModelInfluences`. This one is about a source-space value never being brought into target
space at all: fixing the outbound crossing alone would faithfully resolve the *wrong* slot
through the *target's* bone map and produce a confidently wrong reference-skeleton index.

Preconditions, both common:

- the two meshes' bone maps order or subset the shared skeleton's bones differently — routine
  whenever the meshes were chunked separately, which is every case this verb exists for
  ("porting weights to a remeshed asset");
- more than one section on either mesh.

Symptoms: no error, no warning, a plausible `verticesCopied` count, and a profile whose
influences name the wrong bones. Invisible on a single-section-to-single-section transfer where
both bone maps happen to be the identity over the range in use.

Acceptance: every copied influence is translated source-local → reference-skeleton →
target-local before it is stored, and an influence naming a bone the target section's bone map
does not carry is dropped and **counted** rather than substituted (a section's bone map is fixed
until the next re-chunk, so there is no honest slot to write, and defaulting to 0 weights the
vertex to the root). The count must reach the caller. Regression test: source and target listing
the same two bones in **opposite order**, so a verbatim copy stores the wrong one.

## History
- `#1-found-while-fixing-index-space` `OPEN` reporter — Found while implementing the fix for `B-skin-weight-transfer-writes-section-local-bone-indices` at plugin HEAD `f0ab5a74`. Not previously on the board and not named by the mesh-authoring triage, which treated `copy_weights` only as a second call site of the outbound crossing. The two defects compose badly: repairing only the outbound crossing resolves the SOURCE's slot through the TARGET's bone map, turning a wrong slot into a wrong reference-skeleton bone with more confidence than before.
- `#2-translate-through-refskeleton` `IN-REVIEW` developer — Fixed. (a) `CopyClosestVertexWeights` gained an optional `TArray<int32>* OutMatchedSourceVertex` out-param (default `nullptr`, so the existing helper-level test call site still compiles) recording which source vertex each target vertex inherited from — without it the source section is unknowable after the copy. (b) New `SkinWeightTransferUtils::TranslateCopiedWeightsBetweenLODs(SourceLOD, TargetLOD, MatchedSourceVertex, InOutSkinWeights)` maps each influence source-local → reference-skeleton (`ResolveSectionLocalBone` on the SOURCE LOD) → target-local (`ResolveBoneToSectionLocalSlot` on the TARGET LOD), repacks the survivors largest-first so the packed zero-terminated layout holds, renormalizes any vertex that lost an influence so it does not read as merely degenerate, and returns the drop count. (c) `skeleton.copy_weights` calls it between the copy and the persist, and reports `influencesDropped` plus a `transferWarning` naming the likely cause (source and target not sharing a reference skeleton). The verb summary now states the shared-skeleton precondition and the drop behaviour. (d) Test `PinWright.skeleton.copy_weights.TranslatesBetweenSectionBoneMaps` in `Source/PinWright/Private/Tests/Gameplay/TestSkinWeightBaseReadback.cpp`: source and target list `hand_l`/`foot_l` in opposite order, so a verbatim copy stores slot 1 where slot 0 is correct; plus a narrow-target case asserting the untranslatable influence is dropped, counted, and not substituted with the root. **NOT COMPILED, NOT RUN** — the editor holds the DLL; an integration pass must build and run the suite.
