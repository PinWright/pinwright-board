---
id: F-static-mesh-section-material-map
title: "No published verb reports a static mesh's sections or the section→material-slot mapping — static_mesh.describe returns a slot list that reads clean while 98% of a mesh's triangles render on the wrong slot"
status: IN-REVIEW
severity: High
category: feature
tags: [static-mesh, static-mesh-describe, asset-dump, sections, material-slots, mesh-review, silent-clean-readback, read-only, weapons]
encounters: 1
lastSeen: 2026-09-03T00:00:00Z
---

# A mesh can have every slot bound correctly and still be almost entirely mis-assigned, and the readback says it is fine

`static_mesh.describe` answers "which materials are on this mesh?" It cannot answer "which
**triangles** are on which material", and those are different questions. The first one is a
four-line list that a caller reads as an assignment verdict; the second is the assignment.

## What was called

```
static_mesh.describe {assetPath: "/Game/FPS/Weapons/Meshes/SM_WPN_AR"}
```

## What came back

A `materials[]` array of `{slot, path}` pairs and nothing else about assignment — the same shape
`asset.dump`'s `static_mesh.json` sidecar carries (`materials`, `trianglesByLod`, `verticesByLod`,
`lods`, `bounds`, `collision`, `collisionTraceFlag`, `lightmapResolution`). There is no
`sections[]`, no per-section triangle count, no section→slot index, at any LOD.

**All four slots were bound to the right material instance.** The response is clean, and it is
clean for a mesh that is almost entirely mis-assigned.

## What the response could not see — measured

Section triangle counts on `SM_WPN_AR` LOD0, recovered the only way available (below):
**18,559 / 460 / 2,017 / 384**, total 21,420. The consequences that follow:

- The **Polymer** slot carries **460 of 21,420 triangles — 2.1%** of the mesh.
- The AR's entire **barrel + flash hider (1,342 tri)** renders on the *anodised-aluminium* slot.
- **85% of the stock**, **57% of the grip**, and **90% of the magazine floor** render as metal.
- On the pistol, the **entire polymer grip including all 110 stipple studs (672 tri)** renders as
  *phosphated steel*.

None of that is derivable from `materials[]`. The slot list is identical whether the polymer
material is on 2% of the mesh or 40% of it.

## The failure this actually caused

The builder called `static_mesh.describe`, saw four correctly-bound slots, and concluded the
material assignment was right. **The verb cannot see the defect it was being used to rule out** —
and, unlike a verb that errors or omits a field, it returns a positive-looking result, so nothing
prompts a second check. That is the shape the severity rubric calls silent wrong data: the caller
trusts a result that is a lie and builds on it.

## The only route that answers it today

`python.execute` plus the engine's procedural-mesh bridge:

```python
unreal.ProceduralMeshLibrary.get_section_from_static_mesh(mesh, lod, section)
```

which returns the section's vertices/triangles and must be called once per section, with the
section count discovered by walking indices until it throws. It is a raw-engine escape hatch on a
class that has nothing to do with static-mesh review, it is not on the published surface, and it is
not reachable from any of the mesh-review verbs a caller is steered to.

## What is asked for

Per-LOD `sections[]` on the **shared static-mesh builder**, so `static_mesh.describe` and the
`static_mesh.json` sidecar gain it together (the dump/RPC parity rule `E-dump-rpc-parity` records):

- **Minimum:** `{index, materialIndex, materialSlotName, triangleCount}` per section per LOD. That
  alone turns every measurement above into one call, and makes "is the polymer material actually on
  the polymer?" a comparison rather than an inspection.
- **Useful next:** `firstIndex` / `minVertexIndex` / `maxVertexIndex` (so a caller can locate the
  section in the buffer), and the per-section `bCastShadow` / `bEnableCollision` flags.
- **Derived, cheap, and the thing most reviews actually want:** each slot's **share of the mesh's
  triangles**. A slot at 2.1% next to a slot at 86.6% is the whole finding above, visible without
  the caller doing arithmetic.
- **Bulk reach:** available across a folder the way `geometry.audit_static_meshes` runs its checks —
  slot-balance is a per-kit question, not a per-asset one.

## Root cause — guess, no source read taken

The builder almost certainly emits `materials[]` from the asset's `StaticMaterials` array and never
walks `RenderData->LODResources[i].Sections`, which is where `MaterialIndex` and `NumTriangles`
live. **This is an inference from the response shape, not a source read** — no plugin source was
opened for this ticket, and no `file:line` is claimed. The data is asset-side and already in hand:
`trianglesByLod` is the sum of exactly the per-section counts that are being discarded.

## Severity

**High** by the rubric's silent-wrong-data band, held there after the reach adjustment. The verb
does not merely omit a field — it returns a clean-looking assignment verdict for a mesh whose
assignment is wrong, on the normal path, with no warning and nothing in the response to prompt a
second look. Reach is not every-session (mesh material review is not a per-session task), which
argues for a step down, but the verb is *the* primary static-mesh read and the failure is
unfalsifiable from its own output, which holds it at High.

## Related

- `F-static-mesh-uv-channel-readout` (OPEN, Medium) — **same builder, same round of review, adjacent
  missing field.** That ticket asks for per-LOD `uvChannels` + `lightMapCoordinateIndex`; this one
  asks for per-LOD `sections[]`. Filed separately because the evidence and the consequence differ
  (there: texel density is unverifiable; here: a wrong assignment reads as right), but both land in
  the same place in the shared static-mesh builder, next to the `verticesByLod` loop, and a fixer
  opening that code should do both at once.
- `E-static-mesh-describe-doc-promises-nanite` (OPEN) — a third field on the same verb, where the doc
  over-promises rather than staying silent. Same builder again.
- `B-asset-dump-doc-omits-static-mesh-sidecar` (OPEN, Low) — `asset.dump.md` does not list the
  static-mesh sidecar at all, so a reviewer looking for a per-mesh baseline to diff sections against
  does not learn one exists.
- `F-geometry-mesh-material-assign` (IN-REVIEW) — the **write** side: assigning a material to a
  static-mesh slot. It changes which material a slot points at; it does not change which triangles
  are on the slot, and neither verb lets you check.
- `B-revolve-polygon-drops-material-id`, `B-pwmodel-modifier-output-takes-slot-zero`,
  `B-pwmodel-untagged-generator-inserts-default-slot` — three authoring-side defects that all
  produce exactly this state (geometry on the wrong section). Each of them would be caught at
  review time by this readout instead of by rendering the asset and looking at it.

## Fix

**Verdict: TRUE.** Source inspection confirmed `StaticMeshDumpBuilder.cpp` emitted the
`StaticMaterials` slot list and aggregate triangle/vertex counts, but never walked
`UStaticMesh::GetRenderData()->LODResources[*].Sections`. Because both
`static_mesh.describe` and the `static_mesh.json` registry entry call this builder, both published
surfaces had the same omission.

The shared builder now emits a flat `sections[]` row for every LOD/section with `lodIndex`,
`index`, `materialIndex`, `materialSlotName`, `firstIndex`, `numTriangles`, `minVertexIndex`,
`maxVertexIndex`, `bEnableCollision`, and `bCastShadow`. It also emits `slotUsage[]`, one row per
material slot, with aggregated `lod0TriangleCount` and `lod0TriangleFraction`. The text sidecar
emits the same new facts, the RPC description and `Docs/wiki-src/static_mesh.md` document them,
and the `static_mesh.json` / `static_mesh.txt` aspect versions are now 2.

Files changed: `StaticMeshDumpBuilder.cpp`, `StaticMeshTextEmitter.cpp`, `AssetDumpCache.cpp`,
`StaticMeshDescribeHandler.cpp`, `TestStaticMeshDumpSections.cpp`,
`TestStaticMeshDescribeHandler.cpp`, and `Docs/wiki-src/static_mesh.md`.

Automation coverage added: `PinWright.AssetDump.StaticMeshSections.Cube` compares the engine
cube's single LOD0 section against raw render data and checks text parity;
`PinWright.AssetDump.StaticMeshSections.MultiSection` builds a two-slot, three-triangle fixture
with complete normal/tangent/UV attributes and checks section mapping, 1/3 and 2/3 slot shares,
and the text emitter's material names/counts/fractions and UV/light-map values. The fixture uses a
unique package path, is rooted for the test, and is unrooted and cleaned on every exit. Existing
`PinWright.static_mesh.describe.ReturnsDumpShape` now checks that the RPC carries the new arrays.
Tests were not run in this implementation pass by instruction.

Deliberately unchanged: no separate `geometry.audit_static_meshes` result field was added because
folder dumps already use this shared sidecar; the adjacent UV ticket covers only its requested
minimum; the section count field is `numTriangles` as required by the implementation brief even
though the original ticket prose called it `triangleCount`; no duplicate alias was added. The
separate Nanite documentation mismatch remains owned by `E-static-mesh-describe-doc-promises-nanite`.

## History
- `#1-filed` `OPEN` WEAPONS-critic — Measured during a WEAPONS critic review round 2. `static_mesh.describe {assetPath:"/Game/FPS/Weapons/Meshes/SM_WPN_AR"}` returns a `materials[]` slot list and no section data at all — no `sections[]`, no per-section triangle counts, no section→slot index, at any LOD; `asset.dump`'s `static_mesh.json` sidecar carries the same shape. All four slots on the AR were bound to the right material instance, so the response read clean, and the builder concluded on that basis that the assignment was right. It was not: measured LOD0 section triangle counts are 18,559 / 460 / 2,017 / 384 (total 21,420), so the **Polymer** slot carries 460 triangles — 2.1% of the mesh — while the entire barrel + flash hider (1,342 tri) renders on the anodised-aluminium slot, 85% of the stock / 57% of the grip / 90% of the magazine floor render as metal, and on the pistol the entire polymer grip including all 110 stipple studs (672 tri) renders as phosphated steel. The verb cannot see the defect it was used to rule out, and because it returns a positive-looking result rather than an error or an omission, nothing prompts a second check. The only route that answers the question is `python.execute` + `unreal.ProceduralMeshLibrary.get_section_from_static_mesh(mesh, lod, section)` — a raw-engine escape hatch on an unrelated class, off the published surface, called once per section with the section count discovered by walking until it throws. Ask: per-LOD `sections[]` on the shared static-mesh builder so `static_mesh.describe` and `static_mesh.json` gain it together (per `E-dump-rpc-parity`) — minimum `{index, materialIndex, materialSlotName, triangleCount}`, then buffer offsets and per-section flags, plus each slot's share of the mesh's triangles (the 2.1%-vs-86.6% contrast is the finding, and the caller should not have to compute it), reachable across a folder the way `geometry.audit_static_meshes` runs. Root cause is a **guess**: the builder likely emits `materials[]` from `StaticMaterials` and never walks `RenderData->LODResources[i].Sections` — inferred from the response shape, no plugin source was opened for this ticket and no `file:line` is claimed; note that `trianglesByLod` is already the sum of the per-section counts being discarded. Severity High on the silent-wrong-data band (clean verdict over a wrong assignment, normal path, unfalsifiable from the verb's own output), held there despite mesh review not being an every-session path because this is the primary static-mesh read. Should land with `F-static-mesh-uv-channel-readout`, which asks for a different missing field in the same builder.
- `#2-published-section-map` `IN-REVIEW` developer — Confirmed the builder omission and added all-LOD section-to-slot rows, LOD0 per-slot triangle counts/fractions, matching text output, cache invalidation, documentation, and cube/multi-section automation coverage. Tests not run by instruction.
- `#3-hardened-section-tests` `IN-REVIEW` developer — Hardened the generated fixture with unique package lifetime cleanup and complete tangent basis, guarded material-slot indexing, and asserted exact text-sidecar slot names/counts/fractions plus UV/light-map values. Tests not run by instruction.
- `#4-verified-sections-in-use` `IN-REVIEW` tester — Verified live in a WEAPONS critic review round 3, on the assets the ticket was filed from. `static_mesh.describe` now returns `sections[]` with `lodIndex`, `index`, `materialIndex`, `materialSlotName`, `firstIndex`, `numTriangles`, `minVertexIndex`/`maxVertexIndex` and the per-section flags, plus `slotUsage[]` carrying `lod0TriangleCount` and `lod0TriangleFraction` — exactly the minimum-plus-derived ask in the body. Re-measured four compiled weapon assets in **four calls, no spawn, no world lock** (`#1` needed `python.execute` + `ProceduralMeshLibrary.get_section_from_static_mesh` once per section, with the section count discovered by walking until it throws). `SM_WPN_AR` LOD0 now reads directly off the response: slot0 Receiver/MetalAnodised **16,783 tri (74.09%)**, slot1 Polymer **666 (2.94%)**, slot2 Barrel/MetalPhosphate **4,819 (21.27%)**, slot3 Optic **384 (1.70%)**. The 2.94% polymer share is the finding `#1` said the slot list could not express, and it is now one field rather than an arithmetic step — the "is the polymer material actually on the polymer?" question is a comparison, as asked. Note the AR's section counts differ from `#1`'s (16,783/666/4,819/384 vs 18,559/460/2,017/384): the mesh was rebuilt between the two rounds, so this is not a discrepancy in the readout — the totals are self-consistent (21,420 → 22,652) and the per-section numbers agree with the slot fractions. The follow-on ask this round did **not** find covered — per-triangle material id, per-slot spatial extent (a bounding box on each `slotUsage[]` row), and per-part triangle counts — is filed separately as `F-static-mesh-slot-spatial-extent`, which cross-references this ticket; it is a new capability, not a defect in what shipped here.
