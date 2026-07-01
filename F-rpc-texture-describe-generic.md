---
id: F-rpc-texture-describe-generic
title: "Add live RPC `texture.describe` covering all UTexture subclasses (not just Texture2D)"
status: DONE
severity: Medium
category: feature
tags: [texture, live-rpc, dump-parity, render-target, cube, volume]
---

# Add live RPC `texture.describe` covering all UTexture subclasses (not just Texture2D)

`asset.dump` now writes a generic `texture.json` sidecar for any `UTexture` via `TextureDumpBuilder::BuildTextureJson` (`Source/EditorAutomationRpcGateway/Private/Handlers/Asset/TextureDumpBuilder.cpp`). The sidecar covers `kind`, `textureClass`, `size.{x,y,z}`, `arraySize`, `pixelFormat`, `compressionSettings`, `lodGroup`, `srgb`, `mipGenSettings`, `neverStream`, and `WITH_EDITORONLY_DATA` `source.{x,y,slices,format}` for `UTexture2D`, `UTextureCube`, `UTexture2DArray`, `UTextureCubeArray`, `UVolumeTexture`, and `UTextureRenderTarget` / `UTextureRenderTarget2D`.

The only existing live read is `texture.get_texture_info` (`TextureHandler.cpp:1273`), which hard-casts to `UTexture2D` and rejects every other subclass with `"Failed to load texture"`. Cube / array / volume / render-target assets therefore have **no live read path** — callers must rely on the dump cache (which may be stale) or fall back to `asset.dump` for a single asset, which is heavier than a focused describe call.

This violates the board policy that asset dumps must not have exclusive functionality: any field a dump emits must also be reachable via a live RPC.

**Repro:**
1. `texture.get_texture_info assetPath=/Engine/EngineResources/DefaultTextureCube.DefaultTextureCube`
   → `Failed to load texture: ...` (silent class-cast rejection).
2. `asset.dump assetPath=/Engine/EngineResources/DefaultTextureCube.DefaultTextureCube` writes `texture.json` with full `kind="TextureCube"`, `textureClass`, size, pixelFormat, etc.

**Fix (proposed):** Add `texture.describe assetPath=...` that loads as `UTexture` (not `UTexture2D`), reuses `TextureDumpBuilder::BuildTextureJson` (already generic and `EDITORAUTOMATIONRPCGATEWAY_API`-exported), and returns the same JSON shape as the `texture.json` sidecar. Place the handler alongside the other `texture.*` registrations in `TextureHandler.cpp`. Leave `texture.get_texture_info` untouched for backward compat (its `width`/`height`/`mipCount` shape differs and tools may depend on it); document `texture.describe` as the canonical generic read in `Docs/wiki/texture.md`.

Widening `get_texture_info` instead is rejected: its response shape is Texture2D-specific (`width`, `height`, `mipCount`, `lodBias`, `compression` as a hand-written switch) and changing it would break existing callers. A new method with the dump-parity shape is cleaner.

## History
- `#1-initial-repro` `OPEN` reporter — `TextureDumpBuilder` emits a generic `texture.json` for every `UTexture` subclass, but the only live read (`texture.get_texture_info`) is gated on `Cast<UTexture2D>` at `TextureHandler.cpp:1273`. Cube / array / volume / render-target classes are unreachable except via `asset.dump`. Violates the dump-parity policy. Proposed fix: add `texture.describe` that loads `UTexture` and reuses `TextureDumpBuilder::BuildTextureJson` for identical JSON shape.
- `#2-texture-describe-rpc` `IN-REVIEW` developer — Added direct `texture.describe` live RPC in `TextureHandler.cpp` that loads `assetPath` as `UTexture` and returns `TextureDumpBuilder::BuildTextureJson`; documented it as the canonical generic texture read while keeping `texture.get_texture_info` legacy Texture2D-shaped; added `FTextureDescribeGenericTextureCoversNonTexture2DTest` to cover a package-backed `UTextureRenderTargetCube` so the old Texture2D load path fails the success/result-field assertions.
- `#3-verify-fix` `DONE` tester — Verified: `texture.describe assetPath=/Engine/EngineResources/DefaultTextureCube.DefaultTextureCube` returns full dump-parity JSON (`kind=TextureCube`, `textureClass=Cube`, `size=128x128`, `arraySize=6`, `pixelFormat=PF_DXT1`, `source.format=TSF_BGRA8`, etc.), while the legacy `texture.get_texture_info` on the same asset still errors with `TEXTURE_ERROR: Failed to load texture` — confirms the new live read path covers non-Texture2D subclasses.
