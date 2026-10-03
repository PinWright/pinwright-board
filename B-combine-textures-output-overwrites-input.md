---
id: B-combine-textures-output-overwrites-input
title: "texture.combine_textures (and every CreateEmptyTexture caller) re-creates an existing asset at the output name in place, so an output naming an input wipes that input before it is read"
status: OPEN
severity: Critical
category: bug
tags: [texture, combine_textures, data-loss, overwrite, CreateEmptyTexture, TextureHandler]
encounters: 1
lastSeen: 2026-10-02T18:34:30Z
---

# An output name that resolves to an existing asset is silently re-created in place

Found by code review of the `texture.combine_textures` docs change
(E-combine-textures-docs-omit-placement-size-alpha-limits); not reproduced live.

`combine_textures` (`Handlers/Material/TextureHandler.cpp`, SubAction `combine_textures`) calls
`CreateEmptyTexture(Path, Name, ...)` **before** it locks either input, and so before the
`Bgra8Only` format checks run. `CreateEmptyTexture` (same file, top) composes `<path>/<name>`,
calls `CreatePackage` and `NewObject<UTexture2D>(Package, ..., FName(*TextureName), ...)` with no
existing-asset guard. `NewObject` on the name of a live object of the same class re-allocates it in
place, and `Source.Init` zero-fills it. The handler's own comment already says so ("NewObject
re-allocates in place when the output name resolves to an existing asset - possibly one of the two
inputs") but nothing refuses it.

Consequences:

1. **Output names an input** (`name`/`path` resolving to `baseTexture` or `overlayTexture`): the
   input is re-allocated in place (same address) and zero-filled before it is read. The call then
   returns `TEXTURE_ERROR` from the output write lock: on UE 5.8 `FTextureSource::LockMipInternal`
   (`Texture.cpp:2817-2833`) logs "LockMip cannot lock for write when previously locked for read"
   and refuses ReadWrite under the input's ReadOnly lock. That error comes **after** the wipe: a
   dirty, zero-filled texture is left in memory under the input's name, and a later Save All (or
   any save of that package) makes the loss permanent.
2. **Output names any other existing texture**: it is replaced in place and, with `save: true`
   (the default), saved over on disk, reported as success. An asset that exists only **on disk**
   (not loaded) at the output path is not found by `NewObject` either: `CreatePackage` makes a
   fresh in-memory package and `save: true` writes over the file, whatever class it held - no
   Fatal, a silent overwrite.
3. **Refused calls leave an empty output asset**: a format refusal of either input happens after
   the output was created.
4. **Editor crash:** a loaded asset of a **different class** at the output name (a material, a
   `UTextureRenderTarget2D`, a `UTextureCube`, ...) kills the editor. `StaticAllocateObject`
   (`UObjectGlobals.cpp:3517-3534`, UE 5.8) logs at **Fatal** "Cannot replace existing object of a
   different class" when the existing object's class is not a child of `UTexture2D`.

The same unguarded helper serves `create_noise_texture`, `create_gradient_texture`,
`create_pattern_texture`, `create_normal_from_height`, `resize_texture`, `channel_pack`, and the
not-in-place paths of `invert`, `desaturate` and `adjust_curves`, so cases 2 and 4 apply to all of
them (10 call sites, normal paths, not a rare edge); case 1 applies wherever an output can name an input (`combine_textures`, `resize_texture`,
`channel_pack`, the copy paths). Existing tickets B-combine-textures-leaks-bulkdata-lock-then-crashes
and B-createpackage-unvalidated-paths-plugin-wide treat this only as a lock/crash concern.

**Fix:** refuse an existing asset at the output path with `ASSET_ALREADY_EXISTS` before creating
anything (as `texture.create_from_pixels` already does), at least in `CreateEmptyTexture`'s callers;
an explicit `overwrite` opt-in may follow, but must still refuse an output equal to an input. Do the
output-existence and input-format checks before any asset is created. Failure-direction test: an
output naming the base must be refused and leave the base's pixels unchanged.

**Workaround:** always pass a fresh, unused `name`/`path` for the output (documented on the
`texture.combine_textures` wiki page).

## History
- `#1-filed-from-review` `OPEN` reporter — Filed from the G35b review finding (review ad8f3e5b0f2a13074) on the combine_textures docs change. Source-read only, not reproduced live. The docs now warn that the output must not name an existing asset; the code fix is this ticket.
- `#2-review-critical` `OPEN` reporter — Re-review of G35b (review-G35b.md) verified case 4 against `UObjectGlobals.cpp:3517-3534` (Fatal on a different-class replace) and case 1 against `Texture.cpp:2817-2833` (TEXTURE_ERROR after the wipe, dirty zero-filled texture left in memory). Severity raised High -> Critical: editor crash plus asset data loss on 10 normal-path call sites (README Severity Levels). Added the unloaded-on-disk silent-overwrite case.
