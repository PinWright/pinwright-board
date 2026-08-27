---
id: E-texture-no-srgb-setter
title: "texture.* has no sRGB setter — 'turn off sRGB' forces fallback to property.set on the raw SRGB UPROPERTY"
status: OPEN
severity: Low
category: ergonomic
tags: [texture, srgb, setter-parity, property-set, fallback, docs, set_compression_settings, material-compile-failure, delayed-failure]
encounters: 3
lastSeen: 2026-08-27T19:35:00+05:00
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

## Encounter 2026-08-27 — the gap is not just ergonomic: it produces a shader that does not compile, and the error lands two verbs away

Found building the Atlantis level on UE 5.8 (PinWright at the EAContentExamples58
checkout's HEAD). The symptom and the workaround are exactly as recorded above
and in `#2` — this section does not restate them. Three things are new, and the
first is the reason this ticket is probably mis-rated.

**1. The downstream consequence, which this ticket does not have at all.** Its
worst stated outcome is friction ("forces fallback to `property.set`"). What
actually happens next is a **hard failure**: a material sampling the texture with
`SamplerType: "SAMPLERTYPE_Masks"` refuses to compile.

```
compileSucceeded: false
compileErrors: ["(Node TextureSample) To use 'Masks' as sampler type, SRGB must be disabled for
                 /Game/Atlantis/Textures/T_Caustic_Cells.T_Caustic_Cells"]
```

So the mask/data-texture workflow is not merely inelegant through the typed
surface — it is **broken** through it. Nothing in this ticket mentions
`SAMPLERTYPE_Masks`, a compile error, or that the defect is reachable from the
`material.*` namespace at all.

**2. The failure is delayed and displaced.** The error surfaces two verbs later,
in a different namespace, against a different asset, a long way from the
`set_compression_settings` call that caused it. This ticket frames the gap as a
same-breath ergonomic detour the author notices immediately; in practice an
author who does not already know the rule spends the detour debugging a material.

**3. Reach, and a create-path proposal this ticket does not make.** Reproduced on
all four textures authored in this session. The argument generalises: **the entire
output of this namespace's `create_*` family is data, never colour** — noise,
gradients, patterns — so every one of them needs sRGB off and none of them can get
it. Beyond the two fixes already proposed here (add `texture.set_srgb`; auto-clear
for known-linear compression types), `texture.create_noise_texture` should
**default to `srgb: false`**, because a noise texture is never colour. This ticket
proposes no create-path default change.

**Source enumeration confirming "no verb can clear it on an existing texture."**
Verified at HEAD in this tree during filing. `set_compression_settings`' entire
mutation is `TextureHandler.cpp:937-941` (`CompressionSettings`, `UpdateResource`,
`MarkPackageDirty`, save) — no `SRGB`. Every `SRGB` **write** in the whole texture
namespace targets a texture the same call just created: `TextureHandler.cpp:138`
(`CreateEmptyTexture`, behind every `texture.create_*`), `:750`
(`create_normal_from_height`), `:1883` (`channel_pack`), `:2315`
(`channel_extract`). The only other occurrence is a **read**, `:1201`
(`get_texture_info`). There is no `texture.set_srgb` and no verb accepts an `srgb`
parameter, so this ticket's central claim is now proven by enumeration rather than
by absence of a search hit.

**Re-rating recommendation (not applied — severity left to its author).** This is
filed `category: ergonomic`, `severity: Low`, on the premise that the write is
merely inelegant. On the board's own impact-x-reach rubric, a gap that yields a
non-compiling shader on a normal path, with a misdirecting error two verbs
downstream, across every procedural texture the namespace can create, does not
read as Low and arguably not as `ergonomic`.

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
- `#3-additional-masks-sampler-compile-failure` `OPEN` reporter — Additional evidence (Atlantis level build, UE 5.8, EAContentExamples58 checkout HEAD). Symptom and workaround identical to `#1`/`#2` and not restated; see the dated encounter section above. Three additions, and the first argues this ticket is mis-rated. (a) **The downstream consequence this ticket lacks entirely**: a material sampling the texture as `SamplerType:"SAMPLERTYPE_Masks"` returns `compileSucceeded: false` with `compileErrors: ["(Node TextureSample) To use 'Masks' as sampler type, SRGB must be disabled for /Game/Atlantis/Textures/T_Caustic_Cells.T_Caustic_Cells"]` — the mask/data-texture workflow is broken through the typed surface, not merely inelegant. This ticket has no compile error, no `SAMPLERTYPE_Masks`, and no indication the defect is reachable from `material.*`. (b) **Delayed and displaced failure**: the error surfaces two verbs later, in another namespace, against another asset, far from the `set_compression_settings` call that caused it — this ticket frames it as a detour the author notices at once. (c) **Reach plus a create-path proposal not made here**: reproduced on all four textures authored in the session, and the entire output of the namespace's `create_*` family is data and never colour, so `texture.create_noise_texture` should default to `srgb:false` in addition to the two fixes already proposed. **Source enumeration verified at HEAD**, which upgrades this ticket's central claim from "no search hit" to proven: `set_compression_settings`' whole mutation is `TextureHandler.cpp:937-941` with no `SRGB`; every `SRGB` write in the namespace targets a just-created texture (`:138` `CreateEmptyTexture` behind all `create_*`, `:750` `create_normal_from_height`, `:1883` `channel_pack`, `:2315` `channel_extract`); the only other occurrence is the read at `:1201` (`get_texture_info`); no `texture.set_srgb` exists and no verb accepts an `srgb` parameter. Recommend re-rating off `Low`/`ergonomic` — a non-compiling shader on a normal path with a misdirecting error two verbs downstream, across every procedural texture the namespace creates, is not pure friction. Severity deliberately NOT edited here; that is its author's call. No new bug; widens confirmed impact from ergonomic detour to material-compile failure. encounters→3.
