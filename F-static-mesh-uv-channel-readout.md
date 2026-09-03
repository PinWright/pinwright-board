---
id: F-static-mesh-uv-channel-readout
title: "No verb reports a static mesh's UV channel count or UV layout — static_mesh.describe omits it, asset.dump's static_mesh.json omits it, and GeometryScript's query library is not exposed to the bundled interpreter, so texel density and UV overlap are unverifiable without spawning a DynamicMeshActor"
status: OPEN
severity: Medium
category: feature
tags: [static-mesh, static-mesh-describe, asset-dump, uv, texel-density, lightmap, geometry-script, mesh-review, read-only, world-lock]
encounters: 1
lastSeen: 2026-09-03T00:00:00Z
---

# A mesh review cannot answer "how many UV channels, and are they sane?"

UV channel count is a first-order acceptance question for any static mesh: whether a lightmap
channel exists at all, whether channel 0 is packed or overlapping, whether texel density is
consistent across a kit. Nothing on the published surface answers it, and the one engine route
that would is not reachable from where an asset-only agent runs.

## The three surfaces that should carry it, and do not

**1. `static_mesh.describe` — measured, no UV field.** The response for
`/Game/FPS/Weapons/Meshes/SM_WPN_AR` carried exactly: `bounds`, `materials`, `lods`,
`trianglesByLod`, `verticesByLod`, `lightmapResolution`, `collision`, `collisionTraceFlag`,
`rebuildRenderConsumers`. There is no `uvChannels`, no per-LOD UV count, nothing about layout.
Note the shape of the gap: the verb *does* report `lightmapResolution`, so it reports the
resolution of a lightmap whose channel it cannot confirm exists.

**2. `asset.dump`'s `static_mesh.json` — measured, no UV field.** Checked against an existing
dump on disk rather than by re-running the verb:
`X:\src\unreal\EAContentExamples58\asset-dumps\Game\Dota2\Creeps\Meshes\SM_Creep_D_Wheel\static_mesh.json`
carries **no UV, normal, tangent or hard-edge data** of any kind. So the sidecar is not the
fallback either.

**3. GeometryScript's query library — not exposed to the bundled interpreter.** In
`python.execute`, `unreal.GeometryScriptLibrary_MeshQueryFunctions` raises `AttributeError`, and
the discovery probe returns empty:

```python
[n for n in dir(unreal) if 'GeometryScript' in n and 'Library' in n]   # -> []
```

## Why the remaining route is not usable

The only route left is the dynamic-mesh path: spawn a `DynamicMeshActor`, copy the static mesh
into it, and query the mesh through `geometry.*`. That is a **world mutation**, and an asset-only
agent working under the world lock must not take it — the mesh review that needs the answer is
precisely the review that is not allowed to spawn an actor to get it. The `geometry.get_mesh_info`
`hasUVs` boolean lives behind that same wall (and is a boolean, not a count).

So the capability is not merely undiscoverable; for the caller who needs it, it does not exist.

## What is asked for

A read-only UV readout on the static-mesh **asset** surface, in whichever of the two shared
builders the parity rule mandates so both surfaces gain it at once:

- **Minimum:** per-LOD `uvChannels` count, plus the `lightMapCoordinateIndex` the asset actually
  uses. That alone answers "is there a lightmap channel, and is `lightmapResolution` describing a
  channel that exists?" — the question `static_mesh.describe` currently half-answers.
- **Useful next:** per-channel bounds/extent (does the channel live in 0..1?), an overlap or
  wrapped-island signal, and enough to derive texel density given a material's texture size.
- **Bulk reach:** whatever gains it should be reachable across a folder, the way
  `geometry.audit_static_meshes` runs its checks — a kit's UV consistency is a per-kit question,
  not a per-asset one.

The data is asset-side and cheap: it is on `UStaticMesh`'s render data, which the describe
handler and the dump builder both already hold when they emit `verticesByLod`.

## Severity

**Medium** by the rubric's hard-blocker band, adjusted down for reach. There is no workaround
available to the caller who needs it — not a documented one, not a many-extra-calls one — so a
reasonable and standard review task (verify texel density / UV overlap on a weapons kit) is
simply impossible through the published surface. Not High because mesh UV review is not an
every-session path.

## Related

- `E-static-mesh-describe-doc-promises-nanite` (OPEN) — the same verb missing a *different*
  advertised field. Different field, and there the doc over-promises; here nothing was promised.
  A fixer adding a typed `nanite` object to the shared static-mesh builder is touching exactly
  the place this feature belongs, so the two should land together.
- `B-static-mesh-subsystem-pie-sentinels` (OPEN, High) — the raw-engine workaround this feature
  would replace: `StaticMeshEditorSubsystem.get_num_uv_channels` via `python.execute`, which
  silently returns `0` for every mesh while another stream holds PIE. That the only available
  route also lies under load is part of the case for a first-class verb.
- `E-geometry-mesh-info-omits-bbox` (IN-REVIEW) — the `geometry.get_mesh_info` read verb, which
  does carry a `hasUVs` **boolean** but requires a dynamic mesh in the world, i.e. the route ruled
  out above. Its `hasUVs` is the shape to *not* copy: a boolean does not answer channel count.
- `B-geometry-uv-gen-silent-noop`, `B-pack-uv-islands-is-unwrap`, `F-geometry-uv-prep-pipeline-batch`
  — the UV **authoring** side. All of them mutate; none of them reads an existing asset's channels,
  and each of them would be easier to verify if this readout existed.
- `E-pwmodel-corpus-lightmap-gap-undeclared` (OPEN) — the lightmap-channel coverage question this
  readout would make checkable on a shipped asset rather than by reading `.pwmodel` source.

## History
- `#1-filed` `OPEN` WEAPONS-critic — Found during a WEAPONS critic review that needed texel density and UV overlap on the FPS weapons kit. Three surfaces checked, none carries UV data. `static_mesh.describe {assetPath:"/Game/FPS/Weapons/Meshes/SM_WPN_AR"}` returned exactly `bounds, materials, lods, trianglesByLod, verticesByLod, lightmapResolution, collision, collisionTraceFlag, rebuildRenderConsumers` — no UV channel count, no layout, though it does report `lightmapResolution` for a lightmap channel it cannot confirm exists. `asset.dump`'s `static_mesh.json` checked against an existing on-disk dump, `X:\src\unreal\EAContentExamples58\asset-dumps\Game\Dota2\Creeps\Meshes\SM_Creep_D_Wheel\static_mesh.json`, which carries no UV, normal, tangent or hard-edge data at all. And `unreal.GeometryScriptLibrary_MeshQueryFunctions` is not exposed to the bundled interpreter — `AttributeError`, with `[n for n in dir(unreal) if 'GeometryScript' in n and 'Library' in n]` returning `[]` — so there is no read-only engine route either. The only remaining path spawns a `DynamicMeshActor` into the world, which an asset-only agent under the world lock must not do, so for the caller who needs the answer the capability does not exist rather than merely being awkward. Ask: per-LOD `uvChannels` plus `lightMapCoordinateIndex` on the shared static-mesh builder (so `static_mesh.describe` and the `static_mesh.json` sidecar gain it together), then per-channel bounds / overlap signal, and bulk reach across a folder the way `geometry.audit_static_meshes` runs. Severity Medium: hard blocker with no workaround, adjusted down for reach since mesh UV review is not an every-session path. Should land with `E-static-mesh-describe-doc-promises-nanite`, which asks for a typed field on the same builder.
