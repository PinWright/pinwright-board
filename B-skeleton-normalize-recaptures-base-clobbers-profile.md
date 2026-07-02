---
id: B-skeleton-normalize-recaptures-base-clobbers-profile
title: "skeleton.normalize_weights (and prune_weights) re-capture LOD base skinning into the target profile — silently clobbering set_vertex_weights authored edits, while reporting verticesNormalized:0"
status: IN-REVIEW
severity: High
category: bug
tags: [skeleton, skin-weights, skeletal-mesh, normalize, prune, profile-edit-recaptures-base-skinning, silent-data-loss]
encounters: 1
lastSeen: 2026-07-02T13:11:14.0145809+03:00
---

# normalize_weights re-captures base skinning into the profile, destroying authored set_vertex_weights edits

`skeleton.set_vertex_weights` and `skeleton.normalize_weights` both take a `profileName`
and both write into that named skin-weight profile, so the natural authoring workflow —
"author corrected influences into profile P with `set_vertex_weights`, then renormalize
profile P with `normalize_weights` so every edited vertex sums to 1.0" — targets one
profile with both calls. But `normalize_weights` does **not** renormalize the profile's
own authored influences in place. It unconditionally **re-captures the LOD's base
section skinning** and writes *that* (normalized) into the target profile, overwriting
every authored vertex the earlier `set_vertex_weights` wrote. And because base skinning
already sums to ~1.0, `NormalizeSkinWeights` changes nothing, so the handler reports
`verticesNormalized:0` — a "nothing happened" success — while it has in fact replaced the
entire authored profile with a fresh copy of base skinning.

The two mutators do not compose: authoring into a profile and then normalizing the same
profile are mutually exclusive. The authored edits are silently lost and the caller is
told zero vertices were touched.

## What's wrong

`normalize_weights` (and `prune_weights`, same defect) route through the shared
`ApplyWeightEditToProfile` scaffold, which always seeds its working array from the LOD's
base section vertices via `GetVertices()` + `CaptureBaseSkinWeights` — it never reads the
target profile's existing `FImportedSkinWeightProfileData.SkinWeights`. So when the target
profile already holds authored weights, the edit runs against base skinning instead and
`WriteSkinWeightProfile` overwrites the authored data.

Guilty source — `Plugins/PinWright/Source/PinWright/Private/Handlers/Animation/SkeletalMeshHandler.cpp`,
`ApplyWeightEditToProfile` (lines 186-195):

```cpp
// Capture the LOD's current base skinning (which lives in the section soft-vertices,
// not a named profile), apply the per-op edit, and persist through a named profile ...
TArray<FSoftSkinVertex> Vertices;
LODModel.GetVertices(Vertices);

TArray<FRawSkinWeight> SkinWeights;
SkinWeightTransferUtils::CaptureBaseSkinWeights(Vertices, SkinWeights);   // always base, never the profile

OutResult = EditFn(SkinWeights);
OutVertexCount = SkinWeights.Num();

SkinWeightTransferUtils::WriteSkinWeightProfile(Mesh, LODModel, FName(*ProfileName), SkinWeights);  // clobbers authored profile
```

The edit source is *always* `CaptureBaseSkinWeights(Vertices, ...)`; the profile's own
`LODModel.SkinWeightProfiles[ProfileName].SkinWeights` is never consulted. When the profile
is pre-populated by `set_vertex_weights`, this discards it.

## What it should do

When the target profile already exists and holds SkinWeights, `normalize_weights` /
`prune_weights` should operate on the **profile's existing per-vertex influences** in place
(renormalize/prune what is actually authored there), not re-seed from base skinning.
Re-seeding from base is only correct when the target profile is empty/absent. At minimum,
if a mutator is going to overwrite a non-empty target profile with base skinning, it must
not report `verticesNormalized:0` — that count actively misleads (it says "nothing changed"
while every authored vertex was replaced) — and it should signal the overwrite (or refuse).

## Verbatim repro (live, HEAD)

Mesh `/Game/ExampleContent/IKRig/Mesh/DinoDragon/SK_DinoDragon` (85540 verts, 1 LOD).
Bone indices: hip_L=2, knee_L=3, ankle_L=4.

1. `skeleton.set_vertex_weights` profileName=CustomWeights, lod0:
   v100 {2:0.5, 3:0.5}, v101 {2:0.3, 3:0.7}, v103 {3:0.75, 4:0.25}
   -> `{verticesModified:3}`
2. `skeleton.describe_skin_weights` profileName=CustomWeights sampleCount=104 -> authored
   weights exact: v100 sum=1.0 [{2:0.5},{3:0.5}], v101 [{2:0.3},{3:0.7}],
   v103 [{3:0.75},{4:0.25}]. (profile: norm=3, zero=10387, degen=75150.)
3. `skeleton.normalize_weights` profileName=CustomWeights lodIndex=0
   -> `{vertexCount:85540, verticesNormalized:0}`   <-- claims nothing changed
4. `skeleton.describe_skin_weights` profileName=CustomWeights sampleCount=104 ->
   authored weights GONE, replaced by base skinning:
   - v100 -> [{2:0.5057},{3:0.4943}]  (authored 0.5/0.5 replaced)
   - v101 -> [{2:0.886},{3:0.114}]    (authored 0.3/0.7 destroyed)
   - v103 -> [{2:1.0}]                 (authored knee_L/ankle_L GONE; now 100% hip_L)
   profile flipped to norm=85540, zero=0, degen=0 — i.e. the whole authored+partial
   profile was replaced by a fresh full-mesh base-skinning capture.

The `set_vertex_weights` seed itself is faithful (step 2 confirms exact persistence across
Build()); `normalize_weights` is the clobberer.

severity rationale: impact=silent false-success + silent loss of authored profile data on a normal path (verticesNormalized:0 lies while the authored profile is replaced by base skinning) × reach=rare (skin-weight authoring) but this IS the intended composition of these two very methods -> High

## Relationship to B-skeleton-auto-skin-weights-noop-rebuild (IN-REVIEW)

Distinct failure shape and a defect *introduced by* that ticket's not-yet-merged fix. That
ticket was "normalize/prune/auto do NO weight math — bare Build() + echo." Its fix added
`ApplyWeightEditToProfile` + `CaptureBaseSkinWeights` + `NormalizeSkinWeights` so the
methods now *do* math — but they do it against **base skinning**, so composing them with
`set_vertex_weights` on the same profile silently destroys the authored edits (a new
silent-data-loss shape, not "does nothing"). Filed separately so it is not lost when that
IN-REVIEW ticket is marked DONE; the reviewer of that ticket should fold this in.

## History
- `#2-seed-from-profile-fix` `IN-REVIEW` developer — Root-caused and fixed the clobber in `Plugins/PinWright/Source/PinWright/Private/Handlers/Animation/SkeletalMeshHandler.cpp`. `ApplyWeightEditToProfile` no longer always seeds from base skinning: the seed now routes through a new pure helper `SkinWeightTransferUtils::SeedWeightEditSource(LODModel, ProfileName, OutSkinWeights)` (added in `SkinWeightTransferUtils.h` next to `CaptureBaseSkinWeights`/`WriteSkinWeightProfile`), which reads the target profile's existing `SkinWeights` when that profile is populated on this LOD (so `normalize_weights`/`prune_weights` edit the authored influences IN PLACE) and only falls back to `CaptureBaseSkinWeights(GetVertices())` when the profile is absent/empty. This fixes both mutators (they share the scaffold) — composing `set_vertex_weights(P)` then `normalize_weights(P)` now preserves the authored edits instead of replacing P with a fresh base-skinning capture. Regression test `PinWright.skeleton.normalize_weights.SeedsFromAuthoredProfile` in `Source/PinWright/Private/Tests/Gameplay/TestAnimationHandlers.cpp` drives the production `SeedWeightEditSource` directly against an in-code 2-vertex `FSkeletalMeshLODModel` (distinct base section skinning + a populated named profile; no `Build()`/example content): asserts the seed comes from the authored profile (bones 7/9/3) not base (2/5) and that normalizing it keeps the authored bone assignments, plus that an absent profile still falls back to base. Every assertion fails against the pre-fix always-base seeding, guarding the revert. Chose GO over the two lens defer-on-parent votes: the parent `B-skeleton-auto-skin-weights-noop-rebuild`'s `#2-fix` (commit 52d7a52) is already committed as an ancestor of plugin HEAD, so its IN-REVIEW→DONE is a tester verifying "does real math," which will not re-touch this clobber — deferring `blockedBy` the parent would strand a live silent-data-loss bug behind a gate that never releases for it, so per the board README ("a defer that can name neither a real blocking ticket nor a justifiable date is not a defer") this is fixed directly at HEAD. Compiled clean (later phase runs the tests).
- `#1-initial-repro` `OPEN` reporter — Seed `skeleton.set_vertex_weights` (SEED mode). Task: hand-author corrected influences into a custom profile on SK_DinoDragon, then renormalize that profile. Source-confirmed and live-replay-confirmed: `set_vertex_weights` persists authored weights into CustomWeights exactly (v100 0.5/0.5, v101 0.3/0.7, v103 knee_L 0.75/ankle_L 0.25, all sum~1.0). `normalize_weights` on the SAME profile returns `verticesNormalized:0` yet the post-normalize readback shows all three authored vertices replaced by base skinning (v103 lost knee_L+ankle_L entirely -> 100% hip_L; profile norm-count jumped 3 -> 85540). Root cause: `ApplyWeightEditToProfile` (`SkeletalMeshHandler.cpp` lines 186-195) always seeds from `CaptureBaseSkinWeights(GetVertices())` and never reads the target profile's existing SkinWeights, so `WriteSkinWeightProfile` clobbers the authored profile with base skinning; `prune_weights` shares the scaffold and the defect. Culprit is `skeleton.normalize_weights`, not the seed. Distinct from B-skeleton-auto-skin-weights-noop-rebuild (that = no math at all; this = math against the wrong source, destroying authored edits) — this defect ships in that ticket's IN-REVIEW fix.
