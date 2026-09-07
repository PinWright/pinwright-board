---
id: B-material-verbs-report-green-while-mesh-renders-default-material
title: "Four material verbs report a material healthy — rendersDefaultMaterial:false included — while that exact material renders as the engine Default Material on the skeletal mesh it is assigned to, because none of them compiles the mesh's permutation"
status: OPEN
severity: High
category: bug
tags: [material, material-authoring, get_material_info, compile_material, generate_thumbnail, rendersDefaultMaterial, usage-flags, bUsedWithSkeletalMesh, skeletal-mesh, false-green, silent-substitution, shader-permutation]
encounters: 1
costly: 1
lastSeen: 2026-09-07T07:06:30Z
---

# `rendersDefaultMaterial:false` is emitted at the moment the asset is rendering the Default Material

A `UMaterial` drawn by a `USkeletalMeshComponent` needs `bUsedWithSkeletalMesh`. Without it the
renderer substitutes the engine Default Material on that mesh. **PinWright has four verbs that a
caller would reasonably use to answer "is this material healthy and will it render?", and on a
material in exactly this state all four answer yes** — one of them in a field whose name is a direct
denial of the failure that is happening.

This is not the same defect as `E-material-usage-flags-unreadable` (which asks for a read/write of
the flags, and which I appended to as `#3` with the same evidence). That one is an ergonomic gap:
the value is not exposed. **This one is a correctness bug in what the verbs assert.** A caller who
never asks about usage flags is still told, positively and in four independent ways, that the
material renders — and it does not. Filing separately so the reporting claim can be fixed even if
the flag accessor is not.

## What I called, and what each said

Target: `/Game/FPS/Player/M_FPSArms`, assigned to `/Game/FPS/Player/SKM_FPSArms` (a skeletal mesh,
2 sections, both slots on `MI_FPSArms`), carrying `bUsedWithSkeletalMesh = false`.

| call | response | why it cannot see the failure |
|---|---|---|
| `asset.generate_thumbnail {assetPath:"…/MI_FPSArms", primitive:"sphere"}` | `usingDefaultMaterial:false`, `fallbackOccurred:false`, `meanLuminance:0.132`, and a correct dark render | renders a **static** primitive, which needs no skeletal permutation |
| `material.authoring.compile_material` | `compileSucceeded:true`, `shaderCompile.status:"completed"`, **`rendersDefaultMaterial:false`** | compiles the **default** permutation, not the one the component requires |
| `material.authoring.get_material_info` | domain, blend mode, shading model, node count, parameters, wired `mainInputs` — all correct | reports no usage flag at all |
| `material.decompile_mgir` / `get_material_node_details` | graph fully wired, every input bound | the graph *is* fine; the material is simply not permitted on that mesh |

Meanwhile the actual frames (PIE, pinned `ev100 0`) show the arms as flat pale grey with dense fine
speckle — the Default Material — against an authored near-black sleeve (`SleeveTint` 0.035/0.038/
0.045). Evidence frames: `Docs/fps/evidence/player/b11-01-idle-arms-default-material.png` and
`b11-02-ads-reddot-arms-default-material.png` in the host project.

`property.get {objectPath:"…M_FPSArms.M_FPSArms", propertyName:"bUsedWithSkeletalMesh"}` returned
`false`, and `property.set` of the same name returned `applied:true, markedDirty:true` and saved
(19706 B, mtime `2026-09-07T07:06:30Z`). So the truth was reachable the whole time — through the
reflection namespace, which is not where anyone doing material work is looking.

## What it cost

Five consecutive builds of one stream. The pale arms were attributed in turn to albedo, texture
tiling, exposure, material slot assignment, an uncompiled shader permutation, component material
overrides, and World Position Offset. **Each of those was measured and each measured correct**,
because each was a property of a material that was never being executed. Two of those builds ended
with a written claim that the arms were fixed, on the strength of `compileSucceeded:true` and a
correct thumbnail.

## What I expected

That a verb reporting `rendersDefaultMaterial:false` means the material will not render as the
Default Material, or says which permutation it measured.

## Asks, in priority order

1. **`rendersDefaultMaterial` must not read as a general claim when it measures one permutation.**
   Either account for usage flags against the material's actual consumers, or rename/document it in
   the response itself (the `shaderCompile.hint` string is the natural place — it already warns that
   a graph write is not a shader compile, and this is the same class of trap one level deeper).
2. **`get_material_info` should report the `bUsedWith*` set**, per `E-material-usage-flags-unreadable`.
3. **`asset.generate_thumbnail` should name the primitive class it rendered** next to
   `usingDefaultMaterial`, so `false` from a sphere is not read as a verdict about a skinned mesh.
4. A checkable condition worth a verb: given a material and a component/mesh, does the material
   declare the usage that mesh requires? That is the question every one of these four calls was
   standing in for.

## Related

- `E-material-usage-flags-unreadable` — the flags are not exposed in the material namespace
  (appended `#3` with this evidence; also corrects its "no read and no write" claim, since the
  reflection verbs do reach them).
- `material.compile_mgir` does not carry `bUsedWith*` flags, so a skeletal-mesh material authored
  into a fresh path via MGIR is silently broken by construction and MGIR has no syntax to set them.

## Workaround

Read and write the flag through `property.get` / `property.set` on the `UMaterial`, then
`compile_material` and save. Never treat a sphere thumbnail or `rendersDefaultMaterial:false` as
evidence that a material renders on a skinned mesh; the only proof is a frame of the actual
component.

## History

- `#1-filed` `OPEN` PLAYER — Found while root-causing five builds of pale first-person arms in the
  host project. `bUsedWithSkeletalMesh` was `false` on the arms master material, so the skinned mesh
  drew the engine Default Material in the editor and in PIE, in every frame, while
  `asset.generate_thumbnail`, `material.authoring.compile_material`,
  `material.authoring.get_material_info` and `material.decompile_mgir` all reported the material
  healthy — `compile_material` explicitly so, via `rendersDefaultMaterial:false`. Fixed in the host
  project by setting the flag through `property.set`; filed here because the reporting is the defect,
  not the project's asset. The four-verb table above is the whole finding.
