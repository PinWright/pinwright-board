---
id: B-rendertarget-source-format-missing
title: "texture.json for RenderTarget types lacks source format / dimensions metadata"
status: DONE
severity: Low
category: bug
tags: [texture, render-target, sidecar]
---

# texture.json for RenderTarget types lacks source format metadata

For non-Texture2D types (TextureRenderTarget2D, TextureRenderTargetCube, etc.), `texture.json` emits `pixelFormat` or `format` but no `source` block, while Texture2D dumps include a full source block with import resolution/format.

`TextureDumpBuilder.cpp` guards `source` emission on `Texture->Source.IsValid()` — render targets generally fail this check.

## Sample

`App/App/Drone/DroneRenderTarget/texture.json` — has `pixelFormat`, no `source`.

## Fix sketch

Replace the `Source.IsValid()` guard with a broader check that also emits a `source`-shaped object for RenderTarget types, populated from `UTextureRenderTarget2D::RenderTargetFormat` (or equivalent for cube/array variants). Mark with `"sourceKind": "renderTarget"` so consumers can distinguish.

## History
- `#1-rt-source-gap` `OPEN` reporter — minor schema completeness issue.
- `#2-render-target-source-metadata` `IN-REVIEW` developer — Added render-target fallback source metadata in `TextureDumpBuilder.cpp`, using `UTextureRenderTarget::CanConvertToTexture` for `source.format` and pixel format while preserving imported texture source handling.
- `#3-verify-fix` `DONE` tester — Verified: `asset.dump` on `/App/App/Drone/DroneRenderTarget` produced `texture.json` with a `source` block containing `sourceKind: "renderTarget"`, `format: "TSF_BGRA8"`, `pixelFormat: "PF_B8G8R8A8"`, and dimensions `x: 640, y: 360`.
