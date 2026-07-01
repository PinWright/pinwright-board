---
id: B-texture-describe-size-resident-mip
title: "texture.describe / texture.json top-level `size` (and `pixelFormat`) report the resident streaming mip, not the texture — a streamed 2048x2048 DXT1 texture reads back as 32x32 PF_B8G8R8A8 on the canonical read"
status: WONTFIX
severity: High
category: bug
tags: [texture, describe, dump-parity, size, pixelFormat, streaming, resident-mip, GetSurfaceWidth, get_texture_info, BuildTextureJson, silent-wrong-data]
encounters: 1
lastSeen: 2026-06-25T07:14:35Z
---

# `texture.describe` reports the resident-mip size/format as the texture's `size` and `pixelFormat`

`texture.describe` is documented (in `texture.md`) as **the canonical dump-parity
live read** for texture assets — "the result matches the `asset.dump`
`texture.json` sidecar shape, including `kind`, `textureClass`, `size`, ...
`pixelFormat`...". Its top-level `size{x,y,z}` is the most natural field a caller
reads to answer "what are this texture's dimensions", and `pixelFormat` to answer
"what compression is this stored as".

Both are **silently wrong on a streamed texture**: they report the *currently
resident streaming mip*, not the texture. On a normal 2048x2048 DXT1 streamed
texture the editor typically only has the smallest mip resident, so
`texture.describe` returns `size:{x:32,y:32}` and `pixelFormat:"PF_B8G8R8A8"`
(the uncompressed staging format of the resident mip), while the texture is
actually 2048x2048 DXT1. The value is **streaming-state-dependent** — it changes
with whatever mips happen to be resident at read time — so it's not even a stable
lie. The same builder backs the `texture.json` `asset.dump` sidecar, so dumps are
wrong too.

The legacy `texture.get_texture_info` — the very method `texture.md` tells callers
to stop using — reports both fields **correctly** (2048x2048, DXT1), and the
`describe` response's own nested `source{x,y}` block is also correct (2048x2048).
So within a single `describe` response the top-level `size` (32x32) contradicts
`source` (2048x2048), and the canonical read disagrees with the legacy read it
supersedes. A caller doing the natural "read back dimensions to confirm the round
trip preserved resolution" on `describe.size` is told the 2048x2048 texture is
32x32.

## Root cause

`TextureDumpBuilder::BuildTextureJson` (`Source/PinWright/Private/Handlers/Asset/TextureDumpBuilder.cpp:189-196`)
builds `size` from the base-`UTexture` **resident-resource** accessors, and
`pixelFormat` (via `TryAddConcretePixelFormat`, cpp:89) from the **resident mip 0**:

```cpp
TSharedPtr<FJsonObject> Size = MakeShared<FJsonObject>();
Size->SetNumberField(TEXT("x"), Texture->GetSurfaceWidth());   // resident mip width
Size->SetNumberField(TEXT("y"), Texture->GetSurfaceHeight());  // resident mip height
...
AddPixelFormatField(Root, Texture2D->GetPixelFormat(0));        // resident mip 0 platform format
```

`UTexture::GetSurfaceWidth()/GetSurfaceHeight()` return the resident platform-data
size (the smallest resident mip for a streamed texture, here 32x32), and
`UTexture2D::GetPixelFormat(0)` returns the resident mip's platform format (the
uncompressed `PF_B8G8R8A8` staging format when the compressed high mips aren't
streamed in), not the asset's intended DXT1.

By contrast `texture.get_texture_info` (`Handlers/Material/TextureHandler.cpp:1291-1293`)
uses the **imported-size / asset-format** accessors and is correct:

```cpp
TextureInfo->SetNumberField(TEXT("width"),  Texture->GetSizeX());                    // 2048
TextureInfo->SetNumberField(TEXT("height"), Texture->GetSizeY());                    // 2048
TextureInfo->SetStringField(TEXT("format"), GPixelFormats[Texture->GetPixelFormat()].Name); // DXT1
```

(`BuildTextureJson` already reads the correct imported size for its `source` block
via `Texture->Source.GetSizeX()/GetSizeY()` at cpp:207-208 — only the top-level
`size`/`pixelFormat` use the resident accessors.)

## What it should do

Top-level `size` should report the texture's actual dimensions, and `pixelFormat`
its actual stored format — independent of streaming residency — so that `describe`
agrees with `get_texture_info` and with its own `source` block. For Texture2D,
use `GetSizeX()/GetSizeY()` (imported size, same as `get_texture_info`) for `size`
and `GetPixelFormat()` (no mip index) for `pixelFormat`. The generic-UTexture path
can use `Source.GetSizeX()/GetSizeY()` (already correct for `source`) when source
is valid. Render targets (whose `GetSurfaceWidth/Height` *is* the real surface
size) are unaffected by the size change if the fix is gated to source-backed
textures.

**Workaround:** read dimensions/format from the legacy `texture.get_texture_info`
(Texture2D only), or from the `describe` response's nested `source{x,y}` block, not
the top-level `size`/`pixelFormat`.

## Repro (verbatim, live)

Two different 2048x2048 DXT1 streamed textures in the Content Examples project:

1. `texture.describe {assetPath:"/Game/Global/Textures/T_Leather.T_Leather"}` →
   `{"kind":"Texture2D","textureClass":"TwoD","size":{"x":32,"y":32,"z":0},"arraySize":0,"pixelFormat":"PF_B8G8R8A8","compressionSettings":"TC_Default","lodGroup":"TEXTUREGROUP_World","srgb":true,"mipGenSettings":"TMGS_FromTextureGroup","neverStream":false,"source":{"x":2048,"y":2048,"slices":1,"format":"TSF_BGRA8"}}`
   — top-level `size` is **32x32** and `pixelFormat` **PF_B8G8R8A8**, but the same
   response's `source` is **2048x2048**.
2. `texture.get_texture_info {assetPath:"/Game/Global/Textures/T_Leather.T_Leather"}` →
   `{"message":"Texture info retrieved","textureInfo":{"width":2048,"height":2048,"format":"DXT1","mipCount":12,"sRGB":true,...}}`
   — legacy read is correct: **2048x2048 DXT1**.
3. `texture.describe {assetPath:"/Game/Global/Textures/T_FloorMarble_D.T_FloorMarble_D"}` →
   `...,"size":{"x":32,"y":32,"z":0},...,"pixelFormat":"PF_B8G8R8A8",...,"source":{"x":2048,"y":2048,...}}` (same 32x32 / PF_B8G8R8A8 mismatch).
4. `texture.get_texture_info {.../T_FloorMarble_D}` → `width:2048,height:2048,format:"DXT1"` (correct).

## History
- `#1-initial-repro` `OPEN` reporter — Building leather-mask texture variants, the
  attempt described `T_Leather` to confirm dimensions and noticed `texture.describe`
  reported `size` 32x32 (resident mip) while `texture.get_texture_info` and the
  describe response's own `source` both reported 2048x2048. Replay-confirmed live on
  two 2048x2048 DXT1 textures (`/Game/Global/Textures/T_Leather`,
  `/Game/Global/Textures/T_FloorMarble_D`): the canonical `texture.describe`
  top-level `size` is `{x:32,y:32}` and `pixelFormat` `PF_B8G8R8A8`, both wrong and
  streaming-state-dependent, contradicting both the legacy `get_texture_info`
  (correct 2048x2048 / DXT1) and the same response's `source` block (2048x2048).
  Source: `TextureDumpBuilder.cpp:190-191` builds `size` from
  `GetSurfaceWidth()/GetSurfaceHeight()` (resident platform-mip size) and cpp:89
  builds `pixelFormat` from `GetPixelFormat(0)` (resident mip 0 staging format),
  whereas `get_texture_info` (`TextureHandler.cpp:1291-1293`) uses
  `GetSizeX()/GetSizeY()` + `GetPixelFormat()` and is correct. The canonical
  dump-parity read (and the `texture.json` sidecar it mirrors) silently lies about
  texture resolution and stored format on every streamed texture. Fix: build
  top-level `size`/`pixelFormat` from the imported-size / asset-format accessors
  (or the already-correct `Source.GetSizeX/Y()`), not the resident-resource ones.
- `#2-wontfix` `WONTFIX` developer — Root cause is factually wrong and the proposed
  fix is a literal no-op; the two "disagreeing" accessors are the SAME engine call.
  Verified against UE 5.7 engine source: `UTexture2D::GetSurfaceWidth()` is defined
  as `return static_cast<float>(GetSizeX());` and `GetSurfaceHeight()` as
  `return static_cast<float>(GetSizeY());` (Texture2D.h:131-132), and
  `GetPixelFormat(uint32 LayerIndex = 0u)` (Texture2D.h:158) makes `GetPixelFormat(0)`
  identical to `GetPixelFormat()`. So `describe`'s `GetSurfaceWidth()/GetPixelFormat(0)`
  (TextureDumpBuilder.cpp:190-191,89) and `get_texture_info`'s `GetSizeX()/GetPixelFormat()`
  (TextureHandler.cpp:1291-1293) are the identical functions — they cannot return 32
  vs 2048 for the same texture, and the swap proposed in the Fix would not change a
  single output byte. `GetSizeX()` returns `PrivatePlatformData->SizeX`
  (Texture2D.cpp:349-363) — the full top-mip platform width (2048), NOT a
  resident-streaming-mip value; streaming out mips does not change it, so the
  "resident mip" premise is false. The genuinely observed 32x32 / PF_B8G8R8A8 (if the
  repro was real) is the `#if WITH_EDITOR` `IsDefaultTexture()` branch inside those
  same accessors returning the async-compile placeholder (Texture2D.cpp:354-357,
  370-373, 405-407) during the cold-load LoadObject — a transient timing artifact that
  hits `get_texture_info` identically (same GetSizeX) and that this fix does not touch;
  the only accessor immune to it is the editor `Texture->Source.*` read the `source`
  block already uses. No code change can be derived from this ticket as written; closing
  WONTFIX. (If a stable, non-transient describe-vs-get_texture_info size discrepancy is
  ever re-reproduced after `FTextureCompilingManager::Get().FinishCompilation`, file a
  fresh ticket targeting an async-compile/FinishCompilation wait in the builder, not the
  accessor swap.)
