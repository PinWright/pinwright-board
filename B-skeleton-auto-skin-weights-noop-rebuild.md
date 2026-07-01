---
id: B-skeleton-auto-skin-weights-noop-rebuild
title: "skeleton.auto_skin_weights / prune_weights / normalize_weights do no weight math — each just calls Mesh->Build() and reports success"
status: IN-REVIEW
severity: High
category: bug
tags: [skeleton, skin-weights, silent-noop, skeletal-mesh, prune, normalize]
---

# skeleton.auto_skin_weights (and prune_weights / normalize_weights) are silent no-ops

`skeleton.auto_skin_weights` is documented as: *"Trigger a full mesh rebuild that
**recomputes skin weights from current bind-pose data**."* In reality the handler does
none of that — it loads the mesh, calls `Mesh->Build()`, saves, and returns
`{rebuilt:true}`. `USkeletalMesh::Build()` only re-derives render chunks from the mesh's
**existing imported weights**; it does not recompute/auto-skin anything. The "recompute
from bind pose" the name and docs promise never happens. The caller is told the weights
were rebuilt while the per-vertex influences are byte-for-byte unchanged.

Two sibling mutators in the same file share the identical defect:

- `skeleton.prune_weights` — documented *"Drop bone influences whose weight falls below the
  given threshold, then renormalize."* The handler reads `threshold`, **ignores it
  entirely**, calls `Mesh->Build()`, saves, and echoes `{skeletalMeshPath, threshold}`. No
  influence is ever dropped; `threshold` is decorative.
- `skeleton.normalize_weights` — documented *"Renormalize each vertex's skin weights so they
  sum to 1.0."* The handler calls `Mesh->Build()`, saves, and echoes `{skeletalMeshPath}`.
  No weight is ever touched, let alone renormalized.

All three handler bodies in `Source/EditorAutomationRpcGateway/Private/Handlers/Animation/SkeletalMeshHandler.cpp`
are the same load → `Mesh->Build()` → `McpSafeAssetSave(Mesh)` → echo-success skeleton, with
**zero** skin-weight code:

```cpp
// skeleton.normalize_weights (~lines 270-294) — no weight logic
Mesh->Build();
McpSafeAssetSave(Mesh);
Result->SetStringField(TEXT("skeletalMeshPath"), SkeletalMeshPath);  // that's it

// skeleton.prune_weights (~lines 308-334) — reads Threshold, never uses it
double Threshold = Ctx.GetNumber(TEXT("threshold"), 0.01);
Mesh->Build();
McpSafeAssetSave(Mesh);
Result->SetNumberField(TEXT("threshold"), Threshold);  // echoed, never applied

// skeleton.auto_skin_weights (~lines 489-513) — no recompute
Mesh->Build();
McpSafeAssetSave(Mesh);
Result->SetBoolField(TEXT("rebuilt"), true);
```

This is the same masking class as `B-skeleton-copy-weights-noop-zero-fill` (copy_weights
reports `ok:true` over a no-real-work body), but here the body performs no weight operation
at all rather than a destructive zero-fill. None of the three returns a machine-readable
signal of work done (no `influencesRemoved`, no `verticesNormalized`, no `verticesRebuilt`),
so a caller has no way to tell the documented operation was skipped — and because the
default vertex skinning is not a named skin-weight profile, even the new
`skeleton.describe_skin_weights` readback can't catch it (it reports `profileCount:0` on a
mesh with normal base skinning; see "Verification gap" below).

**Impact:** the exact advertised "tidy up skin weights before shipping" workflow
(prune low-impact influences → auto-skin a fresh weighting → renormalize) runs end-to-end,
every call returns success, and the mesh's skinning is **completely unchanged**. The agent
reports the cleanup succeeded; nothing was cleaned. Worse, `prune_weights` accepting and
silently discarding `threshold` invites data-loss assumptions in the other direction
(a caller who believes a high threshold pruned aggressively).

**Verification gap:** replayed `prune_weights(threshold=0.9)` — a threshold that on a
normalized mesh would strip nearly every single influence (catastrophic) — and it still
returned plain `{skeletalMeshPath, threshold:0.9}` success with the mesh fully intact
afterward (`describe_mesh` bone/LOD/tri/vert counts unchanged), which is only possible
because nothing was pruned.

**Fix options:**
- Implement the documented math: `normalize_weights` should iterate each LOD's vertex
  influences and renormalize to sum 1.0; `prune_weights` should drop influences `< threshold`
  then renormalize; `auto_skin_weights` should actually recompute (or invoke an engine
  auto-weight path) rather than a bare `Build()`. Each should return a count
  (`verticesNormalized` / `influencesRemoved` / etc.) so the no-op is observable.
- If the real math is out of scope, at minimum stop advertising work that isn't done:
  return a clear unsupported/partial result (or fail) instead of documented success, and
  do not accept-then-discard `threshold`.

## History
- `#2-fix` `IN-REVIEW` developer — Replaced all three silent no-ops with real behavior in `Source/PinWright/Private/Handlers/Animation/SkeletalMeshHandler.cpp`, backed by new pure helpers in `Source/PinWright/Private/Handlers/Animation/SkinWeightTransferUtils.h`. **normalize_weights**: now captures the LOD's base skinning via `LODModel.GetVertices()` + `CaptureBaseSkinWeights`, renormalizes each vertex to sum 1.0 with `NormalizeSkinWeights`, writes the result into a named profile (default `NormalizedWeights`) with rebuilt `SourceModelInfluences` so it survives `Build()`, and returns `{verticesNormalized, vertexCount, profileName, lodIndex}`. **prune_weights**: now actually applies `threshold` (was accept-then-discarded) via `PruneSkinWeights` (drop influences `< threshold`, repack largest-first, renormalize survivors; a vertex whose influences would all be pruned is left intact — never zero-filled), rejects `threshold` outside `[0,1)` with `INVALID_THRESHOLD`, persists to a named profile (default `PrunedWeights`), and returns `{influencesRemoved, verticesAffected, threshold, vertexCount}`. **auto_skin_weights**: a from-bind-pose re-skin has no editor-scriptable path, so per the ticket's fallback + the CLAUDE.md "never fake-success" rule it now rejects with `UNSUPPORTED_OPERATION` (was `{rebuilt:true}`) pointing callers at the real mutators / re-import. Updated the wiki overlay (`docs/wiki-src/skeleton.md`) and all three handler docstrings to match. Regression test `PinWright.skeleton.normalize_prune_weights.RealMath` in `Source/PinWright/Private/Tests/Gameplay/TestAnimationHandlers.cpp` drives the production helpers `NormalizeSkinWeights` / `PruneSkinWeights` directly: asserts an un-normalized vertex (sum ~0.5) is rescaled to 1.0 while an already-normalized one is untouched, a sub-threshold influence is dropped and survivors renormalized, and a prune that would strip every influence is a no-op (removes 0, no zero-fill) — every assertion fails against the old bare-`Build()` bodies. Added `MissingRequiredParam` handler-found tests for `normalize_weights` and `prune_weights`. Did not compile/run (later phase).
- `#1-initial-repro` `OPEN` reporter — Seed `skeleton.auto_skin_weights`. Replayed on `/Game/ExampleContent/IKRig/Mesh/DinoDragon/SK_DinoDragon` (61 bones, 1 LOD, 164434 tris / 85540 verts). `skeleton.auto_skin_weights` → `{skeletalMeshPath, rebuilt:true}`; `skeleton.prune_weights(threshold=0.9)` → `{skeletalMeshPath, threshold:0.9}`; `skeleton.normalize_weights` → `{skeletalMeshPath}` — all `ok:true`, no crash/hang. Source-of-truth confirms (`SkeletalMeshHandler.cpp`): all three handler bodies are identical `Mesh->Build(); McpSafeAssetSave(Mesh);` + echo with no skin-weight code at all; `prune_weights` reads `Threshold` and never references it again. `Mesh->Build()` re-derives render data from existing imported weights — it does not recompute/prune/normalize anything. Silent success-with-no-effect across the whole advertised prune→auto-skin→normalize cleanup flow. Distinct from `B-skeleton-copy-weights-noop-zero-fill` (different method; that one zero-fills a profile, these do no weight op at all) and from `F-skeleton-skin-weight-profile-readback` (that adds a readback; it cannot catch this defect because the default skinning is not a named profile — `describe_skin_weights` returns `profileCount:0` here).
