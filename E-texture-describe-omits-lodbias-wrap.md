---
id: E-texture-describe-omits-lodbias-wrap
title: "texture.describe / texture.json omits LODBias, VirtualTextureStreaming, wrap/addressing, and Filter — sibling set_lod_bias / configure_virtual_texture / set_texture_wrap / set_texture_filter writes have no canonical readback; forces fallback to legacy get_texture_info (Filter has none even there)"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [texture, describe, dump-parity, lod-bias, wrap, addressing, readback, get_texture_info, virtual-texture, configure_virtual_texture, virtualTextureStreaming, filter, set_texture_filter]
---

# `texture.describe` (and the `texture.json` dump sidecar it mirrors) omits LODBias and wrap/addressing

`texture.describe` is documented as the **canonical dump-parity live read** that
supersedes the legacy `texture.get_texture_info` (the `texture.md` namespace page
says describe covers "`kind`, `textureClass`, `size`, `arraySize`, `pixelFormat`,
compression, **LOD**, sRGB, mip generation, streaming, and editor source fields").
`asset-audit.md` and `asset.md` further promise that describe and the
`texture.json` sidecar "delegate to the same exported `*DumpBuilder::Build*Json`
function, so they cannot drift."

They don't drift — but the **shared schema itself omits two fields that sibling
write RPCs set**, so after writing them there is no way to read them back through
the canonical surface:

1. **`LODBias`** — `texture.set_lod_bias` writes it, but `texture.describe`
   reports `lodGroup` and `mipGenSettings` and **no `lodBias` field at all**.
2. **Wrap / addressing (`AddressX`/`AddressY`)** — `texture.set_texture_wrap`
   writes them, but `texture.describe` emits no wrap/address field.

The full `BuildTextureJson` key set is enumerated verbatim in
`B-asset-dump-texture2d-duplicate-sidecars` (DONE): `kind`, `textureClass`,
`size{x,y,z}`, `arraySize`, `pixelFormat`, `compressionSettings`, `lodGroup`,
`srgb`, `mipGenSettings`, `neverStream`, `source{x,y,slices,format}` — confirming
at the source level that neither `lodBias` nor any address/wrap key is present.

The value is real, reflected, and stored — only the canonical *read* drops it.
After `set_lod_bias lodBias=2`:
- `properties.json` shows `"LODBias": { "type": "int32", "value": 2, "is_overridden_locally": true, "flags": ["BlueprintReadWrite","Edit"] }`
- the **legacy** `texture.get_texture_info` returns `"lodBias": 2`
- but `texture.describe` returns an object with **no `lodBias` key**.

So an agent that does the natural write-then-confirm round-trip on the canonical
surface is forced back onto the very legacy method (`get_texture_info`) that
`texture.md` tells it to stop using ("legacy and intentionally remains
Texture2D-shaped... Keep using it only for older callers"). For a non-Texture2D
texture (cube/array/volume) `get_texture_info` errors entirely
(`TEXTURE_ERROR: Failed to load texture`, per `F-rpc-texture-describe-generic`),
so on those classes the LOD bias a caller just wrote is **unreadable through any
live RPC**.

## What it should do

Add `lodBias` (the texture's `LODBias` int32) and the wrap/addressing fields
(`addressX` / `addressY`, e.g. `TA_Wrap`/`TA_Clamp`/`TA_Mirror`) to
`TextureDumpBuilder::BuildTextureJson` so they appear in both `texture.json` and
`texture.describe`. These are exactly the fields the sibling setters
`set_lod_bias` and `set_texture_wrap` mutate, and `set_compression_settings` /
`set_texture_group` already round-trip cleanly via `compressionSettings` /
`lodGroup` — the LOD-bias and wrap setters are the two write RPCs with no
canonical readback. Adding them also makes the `texture.md` "covers ... LOD ..."
claim literally true (today it covers LOD *group* but not LOD *bias*).

**Workaround:** read back LODBias via the legacy `texture.get_texture_info`
(Texture2D only); read back wrap by trusting the `set_texture_wrap` success
message; or `asset.dump` + read `properties.json` (`LODBias` / `AddressX` /
`AddressY`).

## Repro (verbatim, live)

1. `texture.create_pattern_texture {name:"T_Replay_Grid", path:"/Game/Textures/Procedural", patternType:"Grid", width:256, height:256}` → created.
2. `texture.set_lod_bias {assetPath:".../T_Replay_Grid", lodBias:2}` → `"LOD bias set to 2"`.
3. `texture.set_texture_wrap {assetPath:".../T_Replay_Grid", wrapMode:"Wrap"}` → `"Wrap mode set to Wrap"`.
4. `texture.describe {assetPath:".../T_Replay_Grid"}` →
   `{"kind":"Texture2D","textureClass":"TwoD","size":{"x":256,"y":256,"z":0},"arraySize":0,"pixelFormat":"PF_DXT1","compressionSettings":"TC_Default","lodGroup":"TEXTUREGROUP_World","srgb":true,"mipGenSettings":"TMGS_FromTextureGroup","neverStream":false,"source":{"x":256,"y":256,"slices":1,"format":"TSF_BGRA8"}}`
   — no `lodBias`, no wrap/address field.
5. `texture.get_texture_info {assetPath:".../T_Replay_Grid"}` →
   `{"textureInfo":{"width":256,"height":256,"format":"DXT1","mipCount":9,"sRGB":true,"virtualTextureStreaming":false,"neverStream":false,"lodBias":2,"compression":"TC_Default"}}`
   — legacy method DOES report `"lodBias":2`.
6. `asset.dump` then `texture.json` → same 11-key shape as step 4 (no `lodBias`/wrap);
   `properties.json` → `"LODBias": { "value": 2, ... }`.

## History
- `#1-initial-repro` `OPEN` reporter — Authoring three procedural textures, the
  attempt set LOD bias and wrap via the `texture.set_lod_bias` / `set_texture_wrap`
  sibling writers, then could not confirm either through the canonical
  `texture.describe` and fell back to the legacy `texture.get_texture_info` for
  LOD bias. Replay-confirmed live: `set_lod_bias lodBias=2` persists
  (`get_texture_info` returns `lodBias:2`, `properties.json` shows
  `LODBias.value=2`, `is_overridden_locally:true`) but `texture.describe` (and the
  `texture.json` sidecar it mirrors) emit none of `lodBias` / `addressX` /
  `addressY`. Source confirmation: `BuildTextureJson`'s 11-key set is enumerated in
  `B-asset-dump-texture2d-duplicate-sidecars` and contains neither field.
  Contradicts the `texture.md` doc claim that describe covers "LOD". Not drift
  (describe == dump sidecar, both omit it) — a schema-completeness gap in the
  shared builder. Sibling setters with no canonical readback: `set_lod_bias`,
  `set_texture_wrap`. Fix: add `lodBias` + `addressX`/`addressY` to
  `TextureDumpBuilder::BuildTextureJson`.
- `#2-additional-virtualtexturestreaming` `OPEN` reporter — Same readback gap
  extends to a THIRD sibling write field: **`VirtualTextureStreaming`**, written
  by `texture.configure_virtual_texture`, is also absent from
  `BuildTextureJson` / `texture.describe`, yet the `texture.md` describe claim
  explicitly promises "**streaming**" fields. Replay-confirmed live on
  `/Game/Global/DemoRoom/Materials/Textures/T_GrungySurface_slnneipc_4K_MR`
  (4096x4096, PF_DXT1, TC_Masks, TEXTUREGROUP_World): baseline `texture.describe`
  omits both VT and lodBias. After `configure_virtual_texture
  {virtualTextureStreaming:true}` (→ `"Virtual texture streaming enabled"`) and
  `set_lod_bias {lodBias:1}` (→ `"LOD bias set to 1"`), `texture.describe` returns
  output **byte-identical to baseline** —
  `{"kind":"Texture2D","textureClass":"TwoD","size":{"x":4096,"y":4096,"z":0},"arraySize":0,"pixelFormat":"PF_DXT1","compressionSettings":"TC_Masks","lodGroup":"TEXTUREGROUP_World","srgb":false,"mipGenSettings":"TMGS_FromTextureGroup","neverStream":false,"source":{...}}`
  with NO `virtualTextureStreaming` and NO `lodBias` key — while the legacy
  `texture.get_texture_info` correctly reports
  `{"virtualTextureStreaming":true,"neverStream":false,"lodBias":1,...}`, proving
  the mutations stuck. Source: `TextureHandler.cpp:1189`
  (`Texture->VirtualTextureStreaming = bVirtualTextureStreaming`) writes it;
  `TextureDumpBuilder.cpp:178-219` `BuildTextureJson` never emits it (only
  `neverStream` at :202). Net: all three of the texture-tuning write fields
  (`VirtualTextureStreaming`, `LODBias`, address/wrap) that `configure_virtual_texture`
  / `set_lod_bias` / `set_texture_wrap` mutate have NO canonical describe readback,
  forcing callers onto the very legacy `get_texture_info` the docs tell them to
  stop using. Fix should also add `virtualTextureStreaming` to `BuildTextureJson`.
- `#3-additional-filter` `OPEN` reporter — A FOURTH sibling write field has no
  canonical (or even legacy) readback: **`Filter`**, written by
  `texture.set_texture_filter`, is absent from BOTH `texture.describe` AND
  `texture.get_texture_info`. So unlike `lodBias`/`virtualTextureStreaming` (which
  at least round-trip via legacy `get_texture_info`), filter is unverifiable
  through ANY texture-namespace read RPC — `asset.dump` → `properties.json` is the
  only readback. Replay-confirmed live on
  `/Game/Global/Textures/T_FloorMarble_D` (2048x2048, PF_DXT1, TC_Default,
  TEXTUREGROUP_World). Setters succeed with bare confirmation strings:
  `set_texture_wrap {wrapMode:"Clamp", save:false}` → `"Wrap mode set to Clamp"`,
  `set_texture_filter {filter:"Nearest", save:false}` → `"Filter set to Nearest"`.
  Both readbacks are byte-identical before and after: `texture.describe` →
  `{"kind":"Texture2D","textureClass":"TwoD","size":{"x":2048,"y":2048,"z":0},"arraySize":0,"pixelFormat":"PF_DXT1","compressionSettings":"TC_Default","lodGroup":"TEXTUREGROUP_World","srgb":true,"mipGenSettings":"TMGS_FromTextureGroup","neverStream":false,"source":{"x":2048,"y":2048,"slices":1,"format":"TSF_BGRA8"}}`
  (no `filter`, no `addressX`/`addressY`); `texture.get_texture_info` →
  `{"textureInfo":{"width":2048,"height":2048,"format":"DXT1","mipCount":12,"sRGB":true,"virtualTextureStreaming":false,"neverStream":false,"lodBias":-1,"compression":"TC_Default"}}`
  (no `filter`, no wrap). The values DID stick: `asset.dump` → `properties.json`
  shows `"AddressX":{"value":"TA_Clamp"...}`, `"AddressY":{"value":"TA_Clamp"...}`,
  `"Filter":{"value":"TF_Nearest","type":"TEnumAsByte<TextureFilter>"...}`,
  `"LODBias":{"value":-1...}`. Fix should add `filter` (the `Filter`
  `TF_*` enum) to `BuildTextureJson` alongside `lodBias` / `addressX` / `addressY` /
  `virtualTextureStreaming` so the set/verify round-trip on the canonical surface
  is complete for all of the texture-tuning setters.
- `#4-retriage` `OPEN` triage — Low→Medium: canonical readback omits 4 setter-written fields forcing legacy/dump fallback, Filter unreadable via any texture RPC.
- `#5-fix` `IN-REVIEW` developer — Added all four setter-written fields to the shared
  `TextureDumpBuilder::BuildTextureJson` (`Source/PinWright/Private/Handlers/Asset/TextureDumpBuilder.cpp`),
  so they now appear in both `texture.describe` and the `texture.json` sidecar:
  `lodBias` (`SetNumberField` from `Texture->LODBias`), `virtualTextureStreaming`
  (`SetBoolField`), `filter` (`AddEnumField` from `Texture->Filter.GetValue()` →
  `TF_*`), and `addressX`/`addressY` (`AddEnumField` from the base virtual
  `GetTextureAddressX()`/`GetTextureAddressY()` → `TA_*`, so the generic builder emits
  wrap for every texture class without a UTexture2D downcast). Bumped the `texture.json`
  aspect version 3→4 in `Source/PinWright/Private/Handlers/Asset/AssetDumpCache.cpp` per
  the cache-freshness rule. `texture.txt` left unchanged (the text emitter only serializes
  a fixed allowlist that excludes these fields, so its bytes don't move — no bump). Scope is
  the read/describe surface only; the existing sibling write RPCs are untouched. Regression
  test: `Source/PinWright/Private/Tests/Assets/TestTextureDescribeSamplingFields.cpp`
  (`PinWright.texture.describe.EmitsSamplingFields`) constructs a transient `UTexture2D`,
  sets LODBias=2 / AddressX=TA_Clamp / AddressY=TA_Mirror / VirtualTextureStreaming=true /
  Filter=TF_Nearest, calls the exported production `BuildTextureJson`, and asserts each of
  the five JSON fields is present with the matching value — it fails if the fix is reverted.
