---
id: B-boolean-fillholes-caps-skinned-mesh-with-garbage-weights
title: "The skeletal boolean round trip CORRUPTS skin weights near the cut: vertices at the feet come out 93% weighted to `head` 162 cm away with all 160 other bones closer, from a source mesh that audits perfectly clean — and `fillHoles: false` does not avoid it"
status: IN-REVIEW
severity: High
category: bug
tags: [geometry, boolean, boolean-subtract, convert-to-skeletal-mesh, skeletal-mesh, skin-weights, skinning, viewmodel, audit-skin-weights, fill-holes]
---

> **The id is a misnomer, kept only so existing references resolve.** It was filed against
> `fillHoles`; the controlled re-run below shows `fillHoles` is not the cause.

# The skeletal boolean round trip corrupts skin weights near the cut

## The source is clean — this is the control

`skeleton.audit_skin_weights {skeletalMeshPath: /Game/Characters/Mannequins/Meshes/SKM_Manny}`:

```
pass: true, failedChecks: 0, vertexCount: 48779
zeroInfluence pass  weightSum pass  influenceCount pass
coincidentSplit  PASS   coincidentVertices 5288, splittingGroups 0, maxSplitCoefficient 0
influenceReach   p999 5.66, max 7.63   (worst offender: pelvis, 44.9 cm, weight 0.01)
```

Stock `SKM_Manny` has **zero** splitting groups and a maximum influence reach of 7.63. Whatever
comes out of the round trip below is made by the round trip.

## Repro

UE 5.8, EAContentExamples58, PinWright on 27145.

1. `geometry.create_from_skeletal_mesh {assetPath: SKM_Manny}` -> 46098 v / 92178 t,
   `fullyWeighted: true`, 161 bones.
2. `geometry.create_box {width: 37, height: 220, depth: 420}` at `(0, 6.85, 90)` — a central column.
   (`width/height/depth` map to **X/Y/Z**, per the verb's own `sizeX/sizeY/sizeZ` echo.)
3. `geometry.boolean_subtract {keepTool: false, fillHoles: true}` -> 41034 t, `changed: true`.
4. `geometry.convert_to_skeletal_mesh {assetPath: /Game/FPS/Player/SKM_FPSArms, skeletonPath:
   SK_Mannequin}` -> `verticesUnweighted: 0`, `fullyWeighted: true`. **No error anywhere.**

`skeleton.audit_skin_weights` on the result:

- `coincidentSplit` **fail**: 4506 coincident verts, 2161 groups, **7 splitting**,
  `maxSplitCoefficient` **2.99** (source: 0 splitting, coefficient 0).
- `influenceReach`: verts 51 and 382 are **0.934 weighted to `head` at 162.6 cm**, `closerBones:
  160` — every other bone in the skeleton is nearer. `normalizedDistance.max` **27.65**
  (source: 7.63).
- `boneCoverage` on a mesh that should be two arms: `foot_r` 594, `foot_l` 596, `spine_04` 2208,
  plus `calf_*`, `thigh_*`, `ankle_*`, `head` 12, `neck_01/02`, `pelvis`.

## `fillHoles: false` is NOT the fix — the controlled re-run

Re-ran the identical chain with `fillHoles: false`:

- Triangle count 41034 -> **41000**. The cap was **34 triangles**, not the wall the first version of
  this ticket assumed.
- The audit was **unchanged where it matters**: `coincidentSplit` still fail with
  **`maxSplitCoefficient` 2.9859159227893493 — identical to 15 decimal places** — and the `head`
  offenders still at 162.6 cm with `closerBones: 160`. `normalizedDistance.max` still 27.65.

So the corrupt influences live on **original mesh vertices near the cut**, not on the cap. The cut
plane at `|x| = 18.5` runs through the outer edge of the feet and hips (they are wider than the
box), and it is that surviving foot/hip geometry that carries `head` at 162 cm.

What actually cleaned it was **deleting the geometry**: a second `boolean_subtract` with a box
spanning `z < 60` removed feet, ankles, calves and knees, after which

```
coincidentSplit  fail  splittingGroups 7 -> 2,  maxSplitCoefficient 2.986 -> 0.00177
influenceReach   max 27.65 -> 7.63   (equal to the untouched source mesh)
boneCoverage     foot_*, ball_*, ankle_*, calf_twist_*, head, neck_* all GONE
```

`maxSplitCoefficient` 0.00177 against a source of 0 is the honest residue of one boolean.

## Why it matters

Nothing in the chain reports a problem: the boolean says `changed: true`,
`convert_to_skeletal_mesh` says `fullyWeighted: true, verticesUnweighted: 0`. The asset renders as
a viewmodel with a **full-height black sliver** across the frame and a large flat pale mass — a
vertex bound to a bone 162 cm away is a triangle stretched 162 cm the moment the mesh is posed.
Before / after: `Docs/fps/evidence/player/08-build03-fillholes-cap-disc.png` and
`10-build03-ads-sight-and-cap-sliver.png` (broken) vs `11-build04-hipfire-clean-mesh.png` and
`12-build04-ads-sight-centred.png` (clean).

Only `skeleton.audit_skin_weights` catches it, and only if you read `influenceReach` — which is
`status: reported`, no verdict — so a caller gating on `pass` sees one `coincidentSplit` failure and
can reasonably read it as an expected seam artefact. I did exactly that and shipped it.

## Asked for

1. **Find the weight corruption.** A source mesh with `splittingGroups: 0` and reach 7.63 must not
   become `splittingGroups: 7` and reach 27.65 through a cut that does not touch those vertices.
   Suspect the weight carry-through in `boolean_subtract` / `convert_to_skeletal_mesh` around
   coincident and newly-split vertices.
2. **Do not report `fullyWeighted: true` for this.** A vertex weighted to a bone with 160 bones
   closer is unweighted for every practical purpose. Gate the write, or report the reach outlier.
3. **Give `influenceReach` a verdict for the unambiguous case.** `closerBones` equal to the entire
   skeleton is not a threshold judgement call.
4. `geometry.select_by_bone_influence` (see `F-isolate-skeletal-mesh-region-arms-only-viewmodel`)
   would sidestep the whole thing: delete by bone, and no cut-seam weights are created at all.

## Workaround

Cut away every region you do not want **as geometry** — one box per region — and re-audit until
`influenceReach.max` matches the source mesh. Do not trust `fullyWeighted`. Do not bother with
`fillHoles: false` for this; it changes 34 triangles and no weights.

severity rationale: impact=ships a visibly broken asset while every verb in the chain reports
success x reach=any skinned boolean, which is the only region-selection tool the skeletal geometry
path has -> High

## Fix

**Verified TRUE against source, and the mechanism is neither the hole filler nor the cut
interpolation.** `FMeshBoolean` deletes the triangles it discards with
`bRemoveIsolatedVertices=true` (`MeshBoolean.cpp:532`), which frees those vertex IDs; it then
appends the other operand's surviving geometry into the SAME mesh (`:733`), where
`FDynamicMesh3::AppendVertex` takes IDs from `VertexRefCounts.Allocate()`
(`DynamicMesh3_Edits.cpp:11`) and RE-USES the freed ones. The skin-weight layer's hook is
`TDynamicVertexSkinWeightsAttribute::OnNewVertex` -> `ResizeAttribStoreIfNeeded`
(`DynamicVertexSkinWeightsAttribute.h:447-464`): it grows the store when the ID is past the end
and does **nothing at all** for a re-used slot. `FDynamicMeshEditor::AppendMesh` then writes skin
weights only for profiles the APPENDED mesh carries (`DynamicMeshEditor.cpp:2073-2106`), and a
cutting box carries none. So every vertex of the tool wall silently inherits the bone weights of
whichever deleted vertex last occupied its slot, and vertices whose IDs land past the end of the
store get empty weights.

That is the whole ticket: a full-height column box deletes head/neck/spine/pelvis vertices and
hands their weights to wall vertices at the feet. It also explains `fullyWeighted: true` (stale
weights are non-empty), and why `fillHoles` changed nothing - the wall is the tool's own geometry,
not the filler's. The interpolating hooks (`OnSplitEdge` lerps, `OnPokeTriangle` blends
barycentrically, `:430` and `:467`) were always correct and are left in charge.

**Fix shape.** Snapshot the target mesh before the engine call; afterwards classify every result
vertex and re-derive only the ones whose weights cannot be accounted for:

- *carried through* - a snapshot vertex at the same position (tolerance `bounds.MaxDim() * 1e-6`)
  carrying exactly these bone weights. Left byte-identical, so an untouched region cannot be
  degraded by the repair.
- *transferred* - everything else, re-derived with the engine's `FTransferBoneWeights` in
  `ClosestPointOnSurface` mode against the snapshot, which barycentrically blends the containing
  triangle's corner weights and renormalizes.

Provenance is decided from position + value rather than from a marker attribute **because** of the
defect: any per-vertex marker rides the same `OnNewVertex` hook and inherits the same stale slot,
so a marker cannot distinguish a re-used slot from a real one.
`FTransferBoneWeights::TransferWeightsToMesh` is deliberately NOT used even though it takes a
`TargetVerticesSubset`: on 5.8 that path sizes `MatchedVertices` by the subset length and then
indexes it by mesh vertex ID (`TransferBoneWeights.cpp:257`, `:285`), which is an out-of-bounds
write for any subset that is not a dense prefix. The per-point entry point the same header
documents as the alternative is used instead.

Applied to `boolean_union` / `boolean_subtract` / `boolean_intersection` / `boolean_trim` /
`self_union`, not only to subtract: all five reach the same delete-then-append engine path, and
fixing only the verb the defect was reported through would leave four verbs corrupting skinning
silently.

**Honest reporting (asks 2 and 3, within the geometry module):**

- the boolean responses gain `skinWeights: {verticesTransferred, verticesUnresolved}`, emitted only
  when the target carried skin weights, so no existing dispatcher response changes;
- `FSkinWeightCoverage` gains a dominant-bone reach check. The rule is **ordinal, not a distance
  threshold**: a vertex is flagged when its dominant influence names the bone FARTHEST from it of
  every bone the mesh carries - which is the ticket's own "closerBones equal to the entire
  skeleton is not a threshold judgement call". `bind_skin_weights` and `convert_to_skeletal_mesh`
  now publish `verticesFarBone` / `farBoneWorst` and answer `fullyWeighted: false` for such a mesh
  (`IsWeightingTrustworthy()`). `IsFullyWeighted()` keeps its old meaning, because the `.pwmodel`
  compiler gates on it with a message about missing influences;
- it reports and never blocks: the reach check measures the mesh against the reference pose its own
  bone attributes carry, so a mesh deliberately moved away from its skeleton would false-positive.

**Not done here, deliberately.** Ask 3's verdict on the `influenceReach` block of
`skeleton.audit_skin_weights` lives in the MAIN PinWright module
(`Handlers/Animation/SkinAuditHandler.cpp` + `SkinAuditAnalysis.h`), not in `PinWrightGeometry`;
it is a reporting change on a different handler and wants its own ticket now that the source of
the corruption is closed. Ask 4 (`geometry.select_by_bone_influence`) remains
`F-isolate-skeletal-mesh-region-arms-only-viewmodel`.

### Files changed

- `Plugins/PinWright/Source/PinWrightGeometry/Private/Handlers/Geometry/GeometrySkinWeightRepair.h`
  (new) - the defect, the engine source that produces it, and the snapshot/repair contract.
- `Plugins/PinWright/Source/PinWrightGeometry/Private/Handlers/Geometry/GeometrySkinWeightRepair.cpp`
  (new) - `FPreEditSnapshot` and `Repair`.
- `Plugins/PinWright/Source/PinWrightGeometry/Private/Handlers/Geometry/GeometryOps_Boolean.cpp`
  - snapshot before, repair after, in `Boolean` / `Trim` / `SelfUnion`.
- `Plugins/PinWright/Source/PinWrightGeometry/Private/Handlers/Geometry/GeometryOps.h`
  - `bSkinned` / `SkinWeightsTransferred` / `SkinWeightsUnresolved` on `FOpResult`.
- `Plugins/PinWright/Source/PinWrightGeometry/Private/Handlers/Geometry/GeometryOpWarnings.h`
  - `AddOpSkinWeights`.
- `Plugins/PinWright/Source/PinWrightGeometry/Private/Handlers/Geometry/BooleanHandler.cpp`
  - three `AddOpSkinWeights` call sites.
- `Plugins/PinWright/Source/PinWrightGeometry/Private/Handlers/Geometry/GeometryUtils.h` and
  `GeometryUtils.cpp` - `ScanDominantBoneReach`, the new `FSkinWeightCoverage` fields,
  `IsWeightingTrustworthy`.
- `Plugins/PinWright/Source/PinWrightGeometry/Private/Handlers/Geometry/SkeletalMeshAssetIOHandler.cpp`
  - `fullyWeighted` from `IsWeightingTrustworthy`, plus `verticesFarBone` / `farBoneWorst` and a
  far-bone warning on `bind_skin_weights`.
- `Plugins/PinWright/Source/PinWrightGeometry/Private/Tests/Geometry/TestGeometryBooleanSkinWeights.cpp`
  (new) - four tests.
- `Plugins/PinWright/Docs/wiki-src/geometry.md` - two new paragraphs under "Skinned characters",
  the `convert_to_skeletal_mesh` field list, and one `boolean_subtract` bullet.

### Reviewer verification

NOT COMPILED AND NOT RUN - a separate compile pass follows this change.

1. These four must be green:
   `PinWright.Geometry.Ops.Boolean.SubtractKeepsEveryVertexOnItsNearerBone`,
   `PinWright.Geometry.Ops.Boolean.SubtractLeavesUntouchedVerticesExactlyAsTheyWere`,
   `PinWright.Geometry.SkinWeights.FarBoneReachPassesACorrectlyWeightedMesh`,
   `PinWright.Geometry.SkinWeights.FarBoneReachFlagsAVertexBoundToTheFarthestBone`.
   The fixture is a two-bone bar cut by a tunnel that runs its whole length, which is this
   ticket's failure at 1/500th the size: it deletes vertices at BOTH ends and creates wall
   vertices along the entire bar.
2. Re-run the ticket's own chain on `SKM_Manny` (create_from_skeletal_mesh -> create_box column ->
   boolean_subtract -> convert_to_skeletal_mesh). Expected: the subtract response carries
   `skinWeights.verticesTransferred > 0` and `verticesUnresolved: 0`, and
   `skeleton.audit_skin_weights` on the written asset reports `influenceReach.max` at or near the
   source mesh's **7.63** rather than 27.65, with `head` gone from `boneCoverage`.
   `maxSplitCoefficient` should collapse from 2.986 toward the ~0.00177 this ticket already
   recorded as the honest residue of one boolean. Those numbers are the acceptance criterion.
3. Confirm no unskinned response moved: an ordinary `geometry.boolean_subtract` on two boxes must
   emit no `skinWeights` key at all, and `verticesFarBone` must be absent from every response for
   a mesh that carries no bone attributes.

## History
- `#1-filed` `OPEN` reporter — Found building the FPS first-person arms viewmodel. Cut `SKM_Manny` down to arms with one `boolean_subtract` (see `F-isolate-skeletal-mesh-region-arms-only-viewmodel` #3, which recorded the round trip as WORKING on the strength of `fullyWeighted: true` — that verdict was wrong, and this ticket is the correction). The cut itself is fine; `fillHoles: true` is what poisons it. Numbers above are from `skeleton.audit_skin_weights` on the shipped `/Game/FPS/Player/SKM_FPSArms`, and the visible failure is in two PIE captures. Not yet re-run with `fillHoles: false` — that needs a world-lock slot, since the round trip mutates the active level.
- `#2-fillholes-was-the-wrong-cause` `OPEN` reporter — Retracting `#1`'s diagnosis and rewriting the body around a controlled re-run. Ran the identical chain with `fillHoles: false`: the cap turned out to be **34 triangles** (41034 -> 41000), and the audit came back with `maxSplitCoefficient` **2.9859159227893493 — identical to 15 decimal places** to the `fillHoles: true` run, with the `head`-at-162.6 cm offenders and `closerBones: 160` untouched. So the corruption is on original mesh vertices near the cut, not on generated cap geometry, and `fillHoles` is a red herring. Added the control that turns this from a suspicion into a defect: `skeleton.audit_skin_weights` on stock `SKM_Manny` returns **`pass: true`, `splittingGroups: 0`, `maxSplitCoefficient: 0`, reach max 7.63** — the source is clean, so the round trip made all of it. What actually fixed the asset was deleting the offending geometry: a second `boolean_subtract` with a `z < 60` box removed feet/ankles/calves/knees and took `splittingGroups` 7 -> 2, `maxSplitCoefficient` 2.986 -> 0.00177, reach max 27.65 -> 7.63 (equal to source), with every `foot_*`/`ball_*`/`ankle_*`/`head`/`neck_*` influence gone from `boneCoverage`. Verified visually in PIE afterwards: the full-height sliver and the pale mass are gone (`Docs/fps/evidence/player/11-build04-hipfire-clean-mesh.png`, `12-build04-ads-sight-centred.png`). Caveat on that last point, stated because it is a real gap in my evidence: I changed the cap and the leg geometry before capturing, so the captures alone do not separate the two — the audit numbers do, and they say the cap changed nothing. The ticket id still says `fillholes` only so existing references resolve.
- `#3-stale-vertex-slots-not-fillholes` `IN-REVIEW` developer - Confirmed TRUE by reading engine source only. Root cause is FMeshBoolean deleting triangles with bRemoveIsolatedVertices and then RE-USING those freed vertex IDs for the appended tool geometry, while the skin-weight layer's OnNewVertex only ever grows its store and never clears a re-used slot; AppendMesh writes weights only for profiles the appended mesh carries, and a cutting box carries none. So the tool wall inherits the deleted vertices' bones (head/neck/spine, from the column the cut removed) and vertices past the end of the store get none - which is exactly why fillHoles was a red herring and why fullyWeighted came back true. Fixed by snapshotting the mesh before the engine call and re-deriving, via FTransferBoneWeights ClosestPointOnSurface against that snapshot, every result vertex whose weights cannot be traced to a pre-op vertex at the same position; untouched vertices are left byte-identical, which is its own test. Applied to all four boolean_* verbs plus self_union, since all five reach the same engine path. Reporting: the boolean responses carry skinWeights {verticesTransferred, verticesUnresolved}, and an ORDINAL dominant-bone reach check (dominant bone is the farthest bone in the skeleton - no threshold to tune) now makes fullyWeighted answer false for this mesh and publishes verticesFarBone / farBoneWorst. Ask 3's influenceReach verdict is NOT done here: it lives in the main PinWright module and wants its own ticket. NOT compiled and NOT run - a separate compile pass follows. See the ## Fix section for files and verification.
