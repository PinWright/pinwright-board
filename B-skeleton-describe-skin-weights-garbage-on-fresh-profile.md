---
id: B-skeleton-describe-skin-weights-garbage-on-fresh-profile
title: "skeleton.describe_skin_weights reports a fresh set_vertex_weights profile as ~95% degenerate with out-of-range garbage bone indices (33537 on a 61-bone skeleton) until a normalize pass rewrites it"
status: IN-REVIEW
severity: Medium
category: bug
tags: [skeleton, skin-weights, skeletal-mesh, describe-skin-weights, silent-wrong-data, uninitialized-buffer]
encounters: 1
lastSeen: 2026-07-02T13:14:11.6193939+03:00
---

# describe_skin_weights surfaces uninitialized/garbage influences for a just-authored profile

Immediately after a `skeleton.set_vertex_weights` that authored **only 4 vertices** into a
freshly-created `CustomWeights` profile (the profile had `profileCount:0` before the call),
`skeleton.describe_skin_weights` (`sampleCount=5`) reports the *whole* 85540-vertex profile
as overwhelmingly invalid — and the sample rows carry bone indices that **cannot exist** on
this mesh's skeleton:

- `vertexCount:85540`, `normalizedVertexCount:2`, `zeroWeightVertexCount:4069`,
  `degenerateVertexCount:81469` (i.e. ~95% of the profile classified degenerate)
- sampled influences show `boneIndex` values like **33537, 456, 113, 115, 117** with
  near-zero weights, even though `skeleton.list_bones` reports the skeleton has only
  **61 bones** (max valid index 60). The agent's own narration flagged these as
  "garbage: boneIndex values like 33537 and 456 that don't exist."

After a `normalize_weights` pass the *same* profile reads back fully valid
(`normalizedVertexCount:85540`, `degenerate:0`, `zero:0`) — because normalize re-captures
the full-mesh base skinning into the profile (see
`B-skeleton-normalize-recaptures-base-clobbers-profile`). So the degenerate/garbage state is
a **transient dirty read of a partially-authored profile**: `set_vertex_weights` writes 4
vertices and leaves the other 85536 entries in whatever state the underlying
`FImportedSkinWeightProfileData.SkinWeights` buffer happens to hold (uninitialized /
`SetNum`-without-init / stale), and `describe_skin_weights` faithfully serializes that
buffer — including out-of-range `boneIndex` values that are clearly uninitialized memory,
not real influences.

## What's wrong

Either (a) `set_vertex_weights`, when it creates/extends a profile, leaves the vertices it
did not author in an uninitialized state (raw `FRawSkinWeight` bytes never zeroed or seeded
from base skinning), so the profile buffer genuinely holds garbage until a later normalize
rewrites the whole thing; or (b) `describe_skin_weights` reads and reports those
un-authored / uninitialized entries verbatim, surfacing `boneIndex` values outside the
skeleton's bone count as if they were real influences.

Either way the readback **actively contradicts the edit** (only 4 vertices were authored,
cleanly) and forces investigative work: here the agent re-authored and re-read twice to
convince itself the anomaly was a dirty buffer and not a `set_vertex_weights` fault. A
readback that reports 81469 "degenerate" vertices and bone index 33537 on a mesh where only
4 vertices were touched is a wrong-data readback: a caller checking "did my authored edits
land and are they valid?" is told the profile is 95% broken when in fact its 4 authored
vertices are exact.

## What it should do

- `set_vertex_weights` should initialize the vertices it does not author to a valid state
  (zero-influence / all-zero `FRawSkinWeight`, or seeded from base skinning) so the profile
  buffer is never left holding uninitialized bytes; **and/or**
- `describe_skin_weights` should never emit `boneIndex` values outside the mesh's bone
  count — an out-of-range index is a diagnostic that the buffer is uninitialized, not a real
  influence, and should be reported as such (or those vertices excluded from the validity
  buckets) rather than counted as "degenerate."

At minimum the validity summary should distinguish "authored + valid" vertices from
"never-authored / uninitialized" ones, so a 4-vertex edit does not read as a 95%-degenerate
profile.

## Verbatim evidence (live, HEAD)

Mesh `/Game/ExampleContent/IKRig/Mesh/DinoDragon/SK_DinoDragon` (85540 verts, 1 LOD, 61 bones).

1. `describe_skin_weights` pre-edit -> `profileCount:0` (no CustomWeights profile yet).
2. `set_vertex_weights` profileName=CustomWeights lod0: authored 4 vertices (v100-v103).
3. `describe_skin_weights` profileName=CustomWeights sampleCount=5 (post-set, pre-normalize)
   -> `vertexCount:85540, normalizedVertexCount:2, zeroWeightVertexCount:4069,
   degenerateVertexCount:81469`; sampled `boneIndex` 33537 / 456 / 113 / 115 / 117 (all >
   the 61-bone max). Confirmed in the transcript (`degenerateVertexCount:81469`,
   `zeroWeightVertexCount:4069`, `boneIndex 33537`).
4. `normalize_weights` on the same profile -> readback flips to `normalizedVertexCount:85540,
   degenerate:0, zero:0` (because normalize re-captures full-mesh base skinning — a
   separate defect).

severity rationale: impact=silent wrong data on a normal readback path (a 4-vertex clean edit reads back as a 95%-degenerate profile with impossible bone indices, misleading the caller and costing extra isolation calls) × reach=rare (skin-weight profile authoring) -> Medium

## Relationship to other skin-weight tickets

- `B-skeleton-normalize-recaptures-base-clobbers-profile` (OPEN) — the *other* half of this
  task's failure: normalize clobbering the authored profile with base skinning. That is why
  the profile "heals" to fully-valid after normalize (it is no longer the authored partial
  profile — it is a fresh full-mesh base-skinning capture). This ticket is specifically the
  **pre-normalize dirty/garbage readback** of the partially-authored profile, a distinct
  method (`describe_skin_weights` / `set_vertex_weights` buffer init) and symptom.
- `F-skeleton-skin-weight-profile-readback` (IN-REVIEW) — added `describe_skin_weights`. This
  ticket is a defect in that new reader's (or `set_vertex_weights`') handling of an
  un-fully-initialized profile buffer.

## History
- `#1-initial-audit` `OPEN` reporter — PROCESS/struggle audit of the `skeleton.set_vertex_weights` focus task (SK_DinoDragon; the judge filed the normalize clobber as `B-skeleton-normalize-recaptures-base-clobbers-profile`). Distinct method + symptom: right after a 4-vertex `set_vertex_weights` into a fresh `CustomWeights` profile (profileCount:0 before), `describe_skin_weights` sampleCount=5 reported the full 85540-vertex profile as `degenerateVertexCount:81469 / zeroWeightVertexCount:4069 / normalizedVertexCount:2` with sampled `boneIndex` 33537/456/113/115/117 — impossible on a 61-bone skeleton (confirmed in the Attempt transcript: `degenerateVertexCount:81469`, `zeroWeightVertexCount:4069`, `boneIndex 33537`; `list_bones`=61 bones). The agent (SAY) flagged the indices as garbage and spent extra re-author/re-read calls isolating that `set_vertex_weights` itself is faithful. Root: either `set_vertex_weights` leaves un-authored profile vertices uninitialized, or `describe_skin_weights` serializes those uninitialized `FRawSkinWeight` entries (out-of-range boneIndex) verbatim into the validity buckets. Suspect is `describe_skin_weights` / `set_vertex_weights` buffer initialization — Audit to decide which. All calls returned ok:true; the anomaly is a wrong-data readback, not an execution error.
- `#2-fix` `IN-REVIEW` developer — Root cause confirmed writer-side: `skeleton.set_vertex_weights` grew a fresh profile's DENSE per-vertex buffer with a plain `ProfileData.SkinWeights.SetNum(LODModel.NumVertices)` (`Handlers/Animation/SkeletalMeshHandler.cpp:597`), which default-constructs the trivial POD `FRawSkinWeight` (engine `SetNum` -> `DefaultConstructItems`, no zeroing for a non-`TIsZeroConstructType` type; `FRawSkinWeight` = bare `InfluenceBones[]`/`InfluenceWeights[]` C-arrays with no ctor). It then zeroed+authored ONLY the named vertices, leaving every other entry as uninitialized garbage that `describe_skin_weights` -> `SummarizeSkinWeights` faithfully serialized (out-of-range boneIndex like 33537, ~95% classified degenerate). It also never rebuilt `SourceModelInfluences`, so even the authored vertices risked not surviving `Build()`'s re-chunk. Fix (fix (a), the root cause): replaced the hand-rolled `FindOrAdd`+`SetNum`+author-only body with the established seed->author->persist pattern the sibling handlers use — seed the whole buffer via `SkinWeightTransferUtils::SeedWeightEditSource` (captures the LOD's base section skinning for a fresh profile so un-authored vertices read back as their real normalized base influences; preserves prior authored influences on a repeat edit), apply the authored overrides on top, then persist via `WriteSkinWeightProfile` (which rebuilds `SourceModelInfluences`). Also tightened the LOD guard to reject negative indices. Files: `Plugins/PinWright/Source/PinWright/Private/Handlers/Animation/SkeletalMeshHandler.cpp` (set_vertex_weights body). Test: added `PinWright.skeleton.set_vertex_weights.SeedsUnauthoredFromBase` (`Tests/Gameplay/TestAnimationHandlers.cpp`) — builds an in-code 4-vertex LOD with distinct base skinning, runs the fixed seed+author+`SummarizeSkinWeights` sequence, and asserts every un-authored vertex retains its valid in-range base bone with ZERO degenerate/zero-weight vertices and no out-of-range sampled bone; reverting to the `SetNum`-only writer leaves those vertices as garbage and the assertions fail. Compiles clean.
