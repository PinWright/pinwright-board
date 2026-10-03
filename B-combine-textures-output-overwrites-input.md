---
id: B-combine-textures-output-overwrites-input
title: "texture.combine_textures (and every CreateEmptyTexture caller) re-creates an existing asset at the output name in place, so an output naming an input wipes that input before it is read"
status: IN-REVIEW
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
- `#3-guard-in-create-empty-texture` `IN-REVIEW` developer — Premise confirmed at 7230b41d (no exists-guard in `CreateEmptyTexture`; combine created its output before the input locks). `PinWrightTextureAssets::CreateEmptyTexture` (`Handlers/Material/TextureHandler.cpp`, declared in `TextureAssetHelpers.h`) now refuses an output whose package exists on disk or whose object path resolves to any loaded object (`FPackageName::DoesPackageExist || StaticFindObject(UObject, ...)`, the `TextureAuthorHandler.cpp` `ValidateAuthoredDestination` guard) with `ASSET_ALREADY_EXISTS`, before `CreatePackage`/`NewObject`. It gained `OutErrorCode`/`OutError` out params; all 10 `TextureHandler.cpp` call sites put them on the wire (a refused path now quotes the engine's reason as `TEXTURE_ERROR` instead of a bare "Failed to create texture"), and `TextureAuthorHandler.cpp` appends the reason to its `CREATE_FAILED`. `combine_textures` now locks (format-checks) both inputs before creating the output, so a refused input leaves no empty output (case 3). Cases 1, 2, 4 and the on-disk overwrite are all refused at the shared helper. Docs: `docs/wiki-src/texture.md` (new `## An existing asset at the output is refused` section; combine H3 and `name` param describe the refusal instead of the old warning). CHANGELOG entry (behaviour change: re-running with the same output name, including defaulted names, is now refused). Tests: `PinWright.texture.combine_textures.OutputNamingAnInputIsRefused` (base pixels unchanged + `ASSET_ALREADY_EXISTS`), `PinWright.texture.create_noise_texture.ExistingOutputIsRefused`, `PinWright.texture.combine_textures.FailedLockReleasesSourceLock` (now also asserts no output after a refused overlay), `PinWright.infra.wiki_handler.MethodPage.TextureCombineTexturesLimits` (updated). Remaining, not done: `resize_texture` / `channel_pack` / the copy paths still create their output before format-checking their input, so a refused input there leaves an empty output asset (no data loss); an explicit `overwrite` opt-in was not added.
- `#4-review-fixes` `IN-REVIEW` developer — Review round 1 (SHOULD-FIX: a refused input on a create-first path left an unsaved output that blocked the retry with ASSET_ALREADY_EXISTS). Now every CreateEmptyTexture caller checks its inputs before it creates the output. `resize_texture` and `create_normal_from_height` take their source lock before creating. `channel_pack` reads every channel before creating. The `inPlace:false` copy paths of `invert`, `desaturate` and `adjust_curves` format-check the input first through `TextureHandlerProbeSourceLock`. The guard also refuses an in-memory package that holds a live asset under another name (`FindPackage` + `FindAssetInPackage`). A deleted-but-uncollected package does not block. The `name` param descriptions of the other 8 helper verbs now mention the refusal, and texture.md says a refused input leaves nothing to block a retry. Tests added: `PinWright.texture.resize_texture.RefusedInputLeavesNoOutput` and `PinWright.texture.invert.RefusedCopyInputLeavesNoOutput`. `OutputNamingAnInputIsRefused` now runs an in-place invert after the refusal to prove the read locks were released. This supersedes the "Remaining" note in #3. Still not done: no test for the on-disk-only (`DoesPackageExist`) half, and no `overwrite` opt-in.
- `#5-review-r2-tests` `IN-REVIEW` developer — Review round 2: added tests that fail on revert of the remaining reorders and of the in-memory-package clause, all in `TestCombineTexturesSourceLock.cpp` with G8 inputs: `PinWright.texture.desaturate.RefusedCopyInputLeavesNoOutput`, `PinWright.texture.adjust_curves.RefusedCopyInputLeavesNoOutput`, `PinWright.texture.channel_pack.RefusedInputLeavesNoOutput`, and `PinWright.texture.create_noise_texture.PackageHoldingOtherAssetIsRefused` (refused while the package holds a live other-named asset; accepted once that asset is deleted (garbage, not yet collected)). The `create_normal_from_height` reorder stays untested, because its format checks already ran first. Comment and whitespace NITs fixed; CHANGELOG entry re-wrapped.
- `#6-linux-verification` `IN-REVIEW` tester — Linux, UE 5.8 Vulkan, PinWright ae877ccc on origin/master (commit 59e8b29b). Tests: `PinWright.texture.combine_textures.OutputNamingAnInputIsRefused`, `.FailedLockReleasesSourceLock`, `PinWright.texture.create_noise_texture.ExistingOutputIsRefused`, `.PackageHoldingOtherAssetIsRefused`, `PinWright.texture.resize_texture.RefusedInputLeavesNoOutput`, `PinWright.texture.channel_pack.RefusedInputLeavesNoOutput`, `PinWright.texture.{invert,desaturate,adjust_curves}.RefusedCopyInputLeavesNoOutput` and `PinWright.infra.wiki_handler.MethodPage.TextureCombineTexturesLimits`. run3/full (offscreen full suite, 5827/5827 ok, 0 fail) passed them non-skipped: none is in skipids.txt and none carries a PINWRIGHT_ASSERTIONS_SKIPPED marker. What they show: an output naming the base is refused `ASSET_ALREADY_EXISTS` with the base's pixels unchanged (the ticket's failure-direction test, case 1). A loaded same-class texture at the output is refused (case 2, loaded half). So is a package that holds a live other-named asset. Every input check runs before the output is created, so a refused input leaves no output to block a retry (case 3, on all CreateEmptyTexture callers except `create_normal_from_height`, whose reorder #5 explains is not observable). The optional `overwrite` opt-in was not added; the Fix says it "may follow". What remains: no passing test covers the on-disk-only refusal, an unloaded package saved at the output path (the `FPackageName::DoesPackageExist` clause, case 2's silent on-disk overwrite). No test covers case 4 either, a loaded asset of another class at the output name (the editor-crash case). The case-4 refusal uses the same class-agnostic `StaticFindObject(UObject)` branch the same-class test exercises, but no test pins it. Next step: a developer adds two in-suite tests: save a texture, unload it and call `create_noise_texture` at that path; then call it at the path of a loaded `UMaterial`. Both should expect `ASSET_ALREADY_EXISTS` with the asset intact. After that the ticket can close.
