---
id: E-texture-no-srgb-setter
title: "texture.* has no sRGB setter — 'turn off sRGB' forces fallback to property.set on the raw SRGB UPROPERTY"
status: OPEN
severity: Low
category: ergonomic
tags: [texture, srgb, setter-parity, property-set, fallback, docs]
encounters: 2
lastSeen: 2026-07-04T16:51:49.1377515+03:00
---

# `texture.*` exposes no sRGB setter, despite `describe` reporting `srgb` and the linear/mask workflow needing it

The `texture.*` namespace ships a coherent family of single-property setters for
the "configure a utility/mask texture" workflow:
`set_compression_settings`, `set_texture_group`, `set_lod_bias`,
`set_texture_filter`, `set_texture_wrap`, `set_streaming_priority`,
`configure_virtual_texture` (full registration block:
`TextureHandler.cpp:2747-2985`). **Conspicuously missing is a setter for the
sRGB flag** — even though:

- `texture.describe` *reports* `srgb` as a first-class field (see the canonical
  read schema in `E-texture-describe-omits-lodbias-wrap` /
  `B-asset-dump-texture2d-duplicate-sidecars`), so the surface treats sRGB as a
  primary texture property on the read side but offers no symmetric write, and
- toggling sRGB off is the **single most common step** when authoring non-color
  / linear utility textures (masks, noise, packed data) — exactly the task here:
  "these are masks/utility textures, not color maps ... turn off sRGB where the
  helper exposes it."

Because no helper exposes it, an agent doing the obvious thing is pushed off the
typed `texture.*` surface and onto a raw reflection write: `property.set` with
the literal engine UPROPERTY name `SRGB` (the attempt confirmed the spelling
against engine `Texture.h`). That works, but it (a) requires the caller to know
the exact native field name, (b) bypasses the `texture.*` setters'
load-as-`UTexture` + `UpdateResource()` + `MarkPackageDirty()` + save
convention, and (c) means the linear-texture setup is split across two different
RPC families (`set_compression_settings` / `set_texture_group` on `texture.*`,
but `SRGB` via `property.set`) for what is conceptually one coherent intent.

This is a **setter-parity ergonomic gap**, distinct from
`E-texture-describe-omits-lodbias-wrap` (which is about the *read* side dropping
`lodBias`/wrap): here a property the read side fully reports has *no write
helper at all*.

## What it should do

Add `texture.set_srgb` (param `srgb: boolean`, plus the standard
`assetPath` / `save`) that loads the asset as `UTexture`, sets `SRGB`, calls
`UpdateResource()` + `MarkPackageDirty()`, and saves — matching the convention of
its sibling setters. This completes the non-color/linear authoring workflow on a
single typed surface and removes the need to know the raw `SRGB` UPROPERTY name.
Docs follow-up (downstream wiki process): `docs/wiki-src/texture.md` currently
documents *only* the generic read (`describe` / `get_texture_info`) and lists
**none** of the setters — it should enumerate the setter family and, until/unless
a typed sRGB setter exists, explicitly note the `property.set SRGB` workaround so
the fallback is at least discoverable rather than found by reading engine headers.

**Workaround:** `property.set` with property name `SRGB` (boolean) on the texture
asset path.

## Evidence (from the task friction note)

> "no `texture.*` helper exposes the sRGB flag despite describe reporting it, so
> I had to fall back to `property.set` on the raw `SRGB` UPROPERTY (verified the
> name in engine `Texture.h`)."

Call-log corroboration: the run used `property.set SRGB=false` twice
(`T_Noise_Detail`, `T_Pattern_Grid`) immediately after the `texture.*` setters
for compression/group, i.e. the sRGB step is the one that fell off the typed
surface. Source confirmation: no `set_srgb` / `"srgb"` setter token exists in
`TextureHandler.cpp`; the only sRGB write path is the generic reflection setter.

## Additional angle — `set_compression_settings` to a mask/normal type leaves `srgb:true` (internal inconsistency with the plugin's own create paths)

A second run (single procedural roughness mask: `create_noise_texture` →
`adjust_levels` → `set_compression_settings TC_Masks` → `set_texture_group`) hit
the same gap from a different direction. The agent expected
`set_compression_settings` to `TC_Masks` to also clear sRGB (mask/normal
compression is linear data), but the handler writes only `CompressionSettings`
and leaves `SRGB` untouched — so the caller still has to fall back to
`property.set SRGB=false` + `asset.save`.

This is not merely a missing setter; it is an **internal inconsistency**. The
plugin's own mask/normal *create* paths already pair `SRGB=false` with the
mask/normal compression type — `TextureHandler.cpp:948-949`
(`AOTexture->SRGB = false; AOTexture->CompressionSettings = TC_Masks;`) and
`TextureHandler.cpp:746-747`
(`NormalMap->SRGB = false; NormalMap->CompressionSettings = TC_Normalmap;`) —
yet the standalone `set_compression_settings` handler
(`TextureHandler.cpp:1068`, `Texture->CompressionSettings = NewSetting;`) never
applies that pairing. So the plugin "knows" mask/normal compression wants linear,
but only when it creates the texture, not when it re-compresses one.

**Alternative / complementary fix:** have `set_compression_settings` also set
`SRGB=false` when the target is a known-linear compression type
(`TC_Masks` / `TC_Normalmap` / `TC_Grayscale` / …) — matching both the plugin's
own create-path convention and the UE texture-editor UX — in addition to (or
instead of) the dedicated `texture.set_srgb` proposed above.

Verbatim replay (HEAD, this iteration):

- `texture.create_noise_texture {name:"T_ReplaySrgb", path:"/Game/Textures/ReplayTmp", noiseType:"Perlin", width:128, height:128, seamless:true}` → created; `texture.describe` → `"compressionSettings":"TC_Default", "srgb":true`.
- `texture.set_compression_settings {assetPath:".../T_ReplaySrgb", compressionSettings:"TC_Masks", save:true}` → `"Compression set to TC_Masks"`.
- `texture.describe {...}` → `"compressionSettings":"TC_Masks", ..., "srgb":true` — sRGB left unchanged.

## History
- `#1-initial-audit` `OPEN` reporter — Authoring three procedural utility
  textures (`/Game/Textures/Procedural`), the attempt configured compression,
  group, LOD bias, and wrap via typed `texture.*` setters but had to drop to
  `property.set SRGB=false` for sRGB because no `texture.*` setter exists for it
  — even though `texture.describe` reports `srgb` as a primary field. Source-
  confirmed: `TextureHandler.cpp:2747-2985` registers seven `set_*`/`configure_*`
  setters and no sRGB setter. PROCESS friction: one coherent "make it linear"
  intent splits across two RPC families and requires knowing the raw engine
  `SRGB` UPROPERTY name. Distinct from `E-texture-describe-omits-lodbias-wrap`
  (read-side omission). Fix: add `texture.set_srgb`; tag `docs` to enumerate the
  setter family + workaround on `docs/wiki-src/texture.md`.
- `#2-additional-set-compression-no-srgb-clear` `OPEN` reporter — Additional
  evidence / new angle: `set_compression_settings TC_Masks` leaves `srgb:true`
  (replay-confirmed at HEAD — `describe` after → `"compressionSettings":"TC_Masks","srgb":true`).
  Same sRGB-workflow gap, but from the setter side rather than the missing-setter
  side: the plugin's own create-mask / create-normal paths pair `SRGB=false` with
  `TC_Masks` / `TC_Normalmap` (`TextureHandler.cpp:948-949`, `746-747`), yet the
  standalone `set_compression_settings` (`TextureHandler.cpp:1068`) writes only
  `CompressionSettings`, so authoring a non-color mask still forces
  `property.set SRGB=false` + `asset.save`. Complementary fix: auto-clear `SRGB`
  for known-linear compression types in `set_compression_settings`, alongside the
  proposed `texture.set_srgb`. encounters→2.
