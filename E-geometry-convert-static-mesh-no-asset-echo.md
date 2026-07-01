---
id: E-geometry-convert-static-mesh-no-asset-echo
title: "geometry.convert_to_static_mesh returns only {actorName, assetPath} and echoes nothing about the asset it just created — forces a separate asset.exists + asset.get_metadata readback to confirm the bake"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [geometry, convert_to_static_mesh, static-mesh, asset, readback, round-trip, response-shape, confirmation, consistency]
---

# `geometry.convert_to_static_mesh` confirms success in prose but carries no machine-readable proof the asset materialized, so "confirm the StaticMesh now exists" always costs two follow-up reads

`geometry.convert_to_static_mesh` (MeshOpsHandler.cpp:364-417) is the terminal
"bake the DynamicMesh into a real `/Game/...` StaticMesh asset" verb at the end
of every procedural-prop authoring chain. On success it returns only:

```
{ "actorName": "<name>", "assetPath": "/Game/GeneratedMeshes/<name>" }
  message: "StaticMesh created from DynamicMesh"
```

The `assetPath` it echoes is just the input path (or the
`/Game/GeneratedMeshes/<actorName>` default it derived) — **not** a confirmation
that anything exists at that path. The handler has *already proven* the asset was
created: it branches on `EGeometryScriptOutcomePins::Success` from
`CreateNewStaticMeshAssetFromMesh(...)` (MeshOpsHandler.cpp:406-410) and only
reaches the success response when the bake succeeded. But it surfaces none of the
new asset's identity to the caller — no `exists`/`created` flag, no `class`
(`StaticMesh`), no triangle count, no LOD count, none of the
Nanite/normals/tangents flags it itself just configured in `CreateOptions`
(MeshOpsHandler.cpp:396-399). So an agent whose intent is "bake it AND confirm
the asset is really there" gets a prose "created" string it cannot
programmatically trust, and must re-derive the same fact with extra round-trips.

## Why this is friction (not just normal granularity)

This is the same response-shape gap as `E-geometry-deformer-echo-mesh-counts`
(mutators omit the post-op count the caller needs, forcing a `get_mesh_info`
readback) — but on a **different verb and a different surface**: that ticket is
explicitly scoped to the MeshOps **mutator family** (twist/taper/bevel/array…)
that all hold `Target.Mesh` and should echo `vertexCount`/`triangleCount`.
`convert_to_static_mesh` is a **create-asset** verb whose natural confirmation is
the *new asset's* existence/metadata, not a mesh count on the source dynamic
mesh — a distinct slot the deformer ticket does not (and should not) own.

It is also distinct from `E-asset-path-vs-assetpath-list-drift`, which uses the
very same convert→`asset.exists` chain as evidence but for a *param-naming* drift
(`path` vs `assetPath`), not for the missing confirmation echo. No board ticket
covers the convert verb's empty success payload.

The cost is paid at the close of every "fabricate a prop and bake it" workflow:
the convert response is unverifiable, so the canonical "confirm it exists" step
becomes two more RPCs (`asset.exists` then `asset.get_metadata`) that re-read
exactly what the convert handler already knew at `Outcome == Success`.

## What it should do

Have the `convert_to_static_mesh` success response echo a minimal confirmation
block so the bake is verifiable in one call. The data is in hand at the success
branch:
- `exists: true` (or `created: true`) — the success branch *is* the proof; the
  handler reached it only because the bake returned `Success`.
- `class: "StaticMesh"` (the asset class it just created).
- `triangleCount` (and `vertexCount`) — readable off the same `Target.Mesh` the
  handler still holds (`GetTriangleCount`/`GetVertexCount`), the exact figures
  `asset.get_metadata` reports back for a baked mesh.
- Optionally the flags it configured in `CreateOptions`
  (`nanite:false`, `recomputeNormals:true`, `recomputeTangents:true`), so the
  caller can confirm the bake settings without a metadata re-read.

This collapses the canonical "bake + confirm" intent from 3 RPCs
(`convert_to_static_mesh` → `asset.exists` → `asset.get_metadata`) to 1, mirroring
how `subdivide`/`simplify_mesh` already carry their post-op topology inline. A
docs-only floor (note on `docs/wiki-src/geometry.md` that the convert response is
fire-and-forget and that confirmation requires `asset.exists`/`asset.get_metadata`)
would at least make the round-trip expected rather than discovered, but the
behavior fix is preferred because the confirming data is already in the handler.

**Workaround:** after `convert_to_static_mesh`, call `asset.exists {assetPath}`
and/or `asset.get_metadata {assetPath}` to confirm the bake landed and read its
class/tri-count/LODs — exactly the two extra calls this task made.

## Friction evidence (this task — geometry.create_pipe "IndustrialPipe_01" build, 10 RPCs, outcome clean, friction:"none")

Struggle audit of the procedural-pipe-prop build (10 RPCs, all first-try clean,
no retries/errors/`python.execute`, judge filed `E-geometry-deformer-echo-mesh-counts #7`
for the bevel-count half). Story steps 8-9 chained the bake and its confirmation:

- `geometry.convert_to_static_mesh {IndustrialPipe_01 → /Game/GeneratedMeshes/IndustrialPipe_01}` → ok, returned only `{actorName, assetPath}` + "StaticMesh created from DynamicMesh".
- `asset.exists {/Game/GeneratedMeshes/IndustrialPipe_01}` → ok (exists=true) — forced solely to confirm the bake the convert response asserted but did not prove.
- `asset.get_metadata {/Game/GeneratedMeshes/IndustrialPipe_01}` → ok (StaticMesh class, LODs, NaniteEnabled, StaticMaterials, 572 triangles, 1 UV channel, collision present) — re-reading the class/tri-count/LOD/Nanite facts the convert handler already held at `Outcome == Success`.

The self-report's closing confirmation ("confirmed it exists with asset.exists=true
plus asset.get_metadata returning StaticMesh-class tags … 572 triangles") is
built entirely from those two follow-up reads. Nothing errored — this is pure
PROCESS overhead: the convert verb's success payload could have carried the
`exists`/`class`/`triangleCount` confirmation inline, folding two readbacks into
the bake call. The pattern recurs at the tail of every geometry→StaticMesh
authoring chain (the same convert→exists/dump readback appears verbatim in the
`E-asset-path-vs-assetpath-list-drift` repro and the `E-geometry-create-name-vs-actorname`
build chain), so the cost scales once per baked prop.

## Docs page to improve (if taken docs-only)

`docs/wiki-src/geometry.md` — note that `convert_to_static_mesh` returns only
`{actorName, assetPath}` and that confirming the bake requires a follow-up
`asset.exists` / `asset.get_metadata`, with reciprocal `## See also` links to
those verbs, so the round-trip is expected at the point the convert verb is
discovered.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the `geometry.create_pipe` "IndustrialPipe_01" procedural-pipe-prop build (10 RPCs, outcome clean, friction:"none", all first-try clean, judge filed `E-geometry-deformer-echo-mesh-counts #7` for the bevel count-echo half). PROCESS finding distinct from that filing: `geometry.convert_to_static_mesh` (MeshOpsHandler.cpp:364-417) returns only `{actorName, assetPath}` + "StaticMesh created from DynamicMesh" — no `exists`/`class`/`triangleCount`/LOD/Nanite confirmation — even though it reaches the success branch only because `CreateNewStaticMeshAssetFromMesh` returned `EGeometryScriptOutcomePins::Success` (MeshOpsHandler.cpp:406-410) and still holds `Target.Mesh`. So story steps 8-9 ("confirm the StaticMesh now exists … existence/metadata readback") forced TWO follow-up reads — `asset.exists` (exists=true) then `asset.get_metadata` (StaticMesh class, LODs, 572 tris, collision) — to verify and read what the convert handler already knew. Fix: echo a minimal confirmation block (`exists:true`, `class:"StaticMesh"`, `triangleCount`/`vertexCount`, optional Nanite/normals flags) on the convert success response, collapsing bake+confirm from 3 RPCs to 1; docs-only floor names `docs/wiki-src/geometry.md`. Dedup: distinct from `E-geometry-deformer-echo-mesh-counts` (scoped to MeshOps *mutators* echoing source-mesh counts, not a create-asset verb's asset-existence echo) and from `E-asset-path-vs-assetpath-list-drift` (same convert→exists chain as evidence but for `path`-vs-`assetPath` param naming, not the empty success payload). Workaround: call `asset.exists`/`asset.get_metadata {assetPath}` after the convert.
- `#2-reword-and-fix` `IN-REVIEW` developer — Reworded: refreshed the stale `MeshOpsHandler.cpp` line citations (288-340 → 364-417, 327-334 → 406-410, 320-323 → 396-399) to match current source; every factual claim already held. Implemented the behavior fix: the `geometry.convert_to_static_mesh` success branch now echoes an inline confirmation block alongside `{actorName, assetPath}` — `exists:true`, `created:true`, `class:"StaticMesh"`, `triangleCount`/`vertexCount` (read off the held `Target.Mesh` via the existing `SnapshotMeshCounts` helper), and the `CreateOptions` flags `nanite:false`/`recomputeNormals:true`/`recomputeTangents:true` — collapsing bake+confirm from 3 RPCs to 1, mirroring `simplify_mesh`/`subdivide`. Files: `Source/PinWright/Private/Handlers/Geometry/MeshOpsHandler.cpp` (success branch). Regression test: `Source/PinWright/Private/Tests/Geometry/TestGeometryConvertStaticMeshEcho.cpp` (`PinWright.geometry.convert_to_static_mesh.EchoesConfirmation`) spawns a real DynamicMeshActor via `geometry.create_box`, routes `geometry.convert_to_static_mesh` through the real dispatcher, and asserts every confirmation field is present with the expected values (non-zero tri/vert counts, Nanite off, normals/tangents on), then deletes the baked asset + probe actor; reverting the echo to `{actorName, assetPath}` only fails it.
