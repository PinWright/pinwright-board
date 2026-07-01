---
id: F-staticmesh-texture-dsl-sidecars
title: "Add StaticMesh and Texture DSL sidecars"
status: DONE
severity: Low
category: feature
tags: [static-mesh, texture, dump-format, dsl, size-reduction]
---

# Add StaticMesh and Texture DSL sidecars

Two of the highest-file-count structured sidecars use pretty JSON for shallow, schema-stable data:

- `static_mesh.json` — **4.1 MB across 6,348 files** (avg 650 bytes)
- `texture.json` — **4.8 MB across 12,392 files** (avg 400 bytes)

Cluster total: **8.9 MB across 18,740 files.** Per-file payload is tiny but the per-file JSON syntactic overhead (`{`, `}`, `"key":`, indentation) is substantial relative to actual data. Additive text sidecars provide a compact grep-friendly companion while preserving the JSON schema for compatibility and live describe RPCs.

Less impactful than F-deprecate-niagara-graphs-model-for-nir or F-scs-dsl-sidecar but easy follow-up once the IR-pattern emitter infrastructure is in place for SCS. Both schemas are VERY stable (texture format especially).

## Proposed formats

### `static_mesh.txt`

```
# ==== Static Mesh ====
# /Game/Environment/Props/SM_Crate.SM_Crate

mesh {
    bounds: (Min=(-50, -50, 0), Max=(50, 50, 100))
    lods: 3
    sections: 2
    vertices: 1024
    triangles: 512
}

lod(0) {
    materials: [
        /Game/Materials/M_Wood,
        /Game/Materials/M_Metal,
    ]
    sections: 2
}

lod(1) {
    materials: [/Game/Materials/M_Wood]
    sections: 1
}
```

### `texture.txt`

```
# ==== Texture ====
# /Game/UI/Icons/T_Health.T_Health
# class: Texture2D

texture {
    dimensions: (256, 256)
    format: PF_DXT5
    compression: TC_Default
    srgb: true
    mip_count: 9
    streaming: true
}

import {
    source_path: "Textures/Icons/health.png"
    source_size: (1024, 1024)
}
```

Conventions match existing IRs:
- Section banner, `key: value` blocks, indent nesting
- UE enums emitted as-is (`PF_DXT5`, `TC_Default`) — already grep-friendly
- Arrays inline when short; multi-line with trailing commas for readability

## Implementation outline

1. New files: `Source/EditorAutomationRpcGateway/Private/Handlers/Asset/StaticMeshTextEmitter.{h,cpp}` and `TextureTextEmitter.{h,cpp}`.
2. Wire `static_mesh.txt` and `texture.txt` into `DumpFileNames`, explicit `AddStringFile` emission, and cache aspect versions.
3. Tests under `Source/EditorAutomationRpcGateway/Private/Tests/`.
4. Keep `static_mesh.json` and `texture.json` emitted as the structured dump format and the live `static_mesh.describe` / `texture.describe` surface.

Effort: 2 days each (4 days total) once SCS DSL infrastructure (F-scs-dsl-sidecar) is in place. Sequencing matters — SCS first to validate the pattern, these as follow-ons.

**Workaround:** None needed — current JSON is small enough to not block work. The implemented text sidecars are additive, so existing JSON consumers continue to work.

## History
- `#2-additive-sidecars` `IN-REVIEW` codex — implemented additive `static_mesh.txt` and `texture.txt` dump emission with new text emitters, cache aspect versioning, regression coverage, and docs. JSON files remain emitted for compatibility and live describe RPCs.
- `#1-initial-proposal` `OPEN` reporter — small per-file but high count (18,740 files combined). Stable schemas, perfect fit for the IR DSL pattern. Should follow F-scs-dsl-sidecar to reuse infrastructure.
- `#3-verify-fix` `DONE` tester — Verified: fresh `asset.dump` on `/Game/Textures/Asset` and `/App/App/Mesh/SM_Cube` emitted `texture.txt` and `static_mesh.txt` alongside the JSON sidecars (writtenPaths included all four files per asset). Sidecars are well-formed DSL with real data: texture.txt has kind/textureClass/size/pixelFormat/compressionSettings/lodGroup/srgb/source block; static_mesh.txt has bounds/materials/lods/trianglesByLod/verticesByLod/collision. JSON files still emitted, so additive contract holds.
