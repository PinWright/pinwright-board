---
id: B-combine-textures-leaks-bulkdata-lock-then-crashes
title: "texture.combine_textures returns a soft TEXTURE_ERROR 'Failed to lock texture data' and LEAKS the bulk-data lock — the next identical call kills the editor on Assertion failed: IsUnlocked() (BulkData.cpp:965) from TextureHandler.cpp:2189"
status: IN-REVIEW
severity: Critical
category: bug
tags: [texture, combine_textures, editor-crash, assertion, bulkdata, lock-leak, retry-kills, TextureHandler, procedural-texture, shared-editor]
encounters: 1
lastSeen: 2026-09-02T22:32:45+00:00
---

# The soft failure arms the crash: a failed lock is never released, so the retry asserts

## Symptom, in the order it happens

Two calls, byte-identical arguments, minutes apart.

**Call 1 — soft error, editor survives:**

```jsonc
call("texture.combine_textures", {
  baseTexture:    "/Game/FPS/Env/Textures/T_ENV_Asphalt_Agg",
  overlayTexture: "/Game/FPS/Env/Textures/T_ENV_Asphalt_Macro",
  blendMode: "Overlay", opacity: 0.6,
  name: "T_ENV_Asphalt_H", path: "/Game/FPS/Env/Textures", save: true })
-> [TEXTURE_ERROR] Failed to lock texture data
```

Nothing in that response suggests the process is now in a bad state. It reads as an ordinary
retryable failure — and "retry it" is exactly what the wording invites.

**Call 2 — identical arguments — the editor dies:**

```
Assertion failed: IsUnlocked()
  [File: .../Runtime/CoreUObject/Private/Serialization/BulkData.cpp] [Line: 965]

FDebug::CheckVerifyFailedImpl2()                 AssertionMacros.cpp:797
FBulkData::LockReadOnly()                        BulkData.cpp:965
UnrealEditor-PinWright.dll!ExecuteTextureAction()  TextureHandler.cpp:2189
UnrealEditor-PinWright.dll!RunTextureAction()      TextureHandler.cpp:2703
UnrealEditor-PinWright.dll!AutoHandler_354_()      TextureHandler.cpp:2929
FRpcDispatcher::DrainAutoRegistrations lambda      RpcDispatcher.cpp:436
FRpcDispatcher::ProcessRequest()                   RpcDispatcher.cpp:813
```

Evidence: `X:/src/unreal/EAContentExamples58/Saved/Logs/EAContentExamples58.log`, `appError` at
`2026.09.02-22.32.45:280`, Critical-error block at `22.32.57:416`.

## The diagnosis the two calls together give you

`IsUnlocked()` fires because the bulk data is **already locked when the second call tries to lock
it**. Nothing else in the session touched those textures. So call 1 did not merely fail to acquire
a lock — it acquired or half-acquired one, hit its error path, and returned without unlocking.
`ExecuteTextureAction` (`TextureHandler.cpp:2189`) needs an unlock on **every** exit path, including
the one that produces `TEXTURE_ERROR`, and today it plainly has one that does not.

That makes the soft error the *cause* of the crash rather than an independent event, which is why
this is Critical and not Medium: the failure mode is **"the safe-looking error you are meant to
retry is the thing that arms the kill"**, and a caller has no way to know.

## Why call 1 failed at all (probably a second, smaller defect)

Both source textures had been created seconds earlier by `texture.create_noise_texture`, which
reported `saved:true, existsOnDisk:true`, and both `.uasset` files were on disk at the moment of
the call (3.4 MB and 1.2 MB, verified by `ls`). A freshly created 2048×2048 texture is still being
built asynchronously at that point, so the likely first fault is that `combine_textures` reads
platform/bulk data without waiting for the texture's build to finish — the same class of thing
`asset.generate_thumbnail` handles with an `assetCompilationWaited` step. If so the verb needs a
`FTextureCompilingManager::FinishCompilation` (or equivalent) wait before it locks, in addition to
the unlock fix.

## What should happen

1. **Release the lock on every exit path** in `ExecuteTextureAction`, ideally by replacing the raw
   `LockReadOnly()`/unlock pair with a scope guard so an early return cannot skip it. This alone
   downgrades the incident from "editor dies" to "one call fails".
2. **Wait for the texture build before locking**, so the common case stops failing at all.
3. **Say which texture could not be locked and why** — the message names neither the asset nor the
   side (base vs overlay), so a caller cannot even narrow it down.
4. If a lock genuinely cannot be taken, the response should say the operation is **not** retryable,
   because as shipped the retry is fatal.

## Reach

`texture.combine_textures` is the composition step in the procedural-texture chain the wiki
advertises (`create_noise_texture` → `combine_textures` → `create_normal_from_height`). Anyone
generating source textures in-editor rather than importing them hits this verb, and hits it right
after creating the inputs — the exact timing that makes call 1 fail.

Workaround in force for my stream: **do not call `texture.combine_textures` at all.** Generate the
noise layers separately and compose them in the material graph instead, where the same multiply /
overlay maths is a few nodes and no bulk data is locked.

severity rationale: impact=editor process kill in a shared editor, destroying unsaved state for
every attached agent, reached by retrying a soft error the verb itself invites x reach=any caller
composing procedural textures -> Critical

## Fix

Both defects confirmed against the source. The unlock guard was `if (BaseData)
BaseMip.BulkData.Unlock();` — but `FBulkData::LockReadOnly` (BulkData.cpp:963-978) sets
`LOCKSTATUS_ReadOnlyLock` *before* returning `GetDataBufferReadOnly()`, which is null for an
empty / GPU-resident payload. A null return therefore means LOCKED, and the guard skipped the
unlock in exactly that case. `check(IsUnlocked())` at the top of the next `LockReadOnly` is the
Fatal that followed. The reason the read returned null at all is the same one already recorded on
`B-texture-normal-from-height-crash`: the *built platform* mip is the compressed / GPU-side copy
and has no readable CPU payload.

Fixed subtractively rather than by adding `IsUnlocked()` checks: **no `FBulkData` lock is left in
the texture handlers to leak.** Every pixel read and write now goes through
`TextureSourceMip::FScopedMipLock`, which locks the editor `FTextureSource` (CPU-resident,
uncompressed) via the engine's own `FTextureSource::FMipLock` RAII pair — a refused lock leaves
`LockState == None` and the destructor no-ops, so "released on every exit path" is a property of
the type, not a discipline at 22 call sites. The guard's constructor also runs
`FTextureCompilingManager::FinishCompilation` first: an in-flight async build holds a read lock on
the same `FTextureSource`, and `LockMipInternal` refuses a ReadWrite lock under a read lock by
returning null — which is what produced the soft error one call after `CreateEmptyTexture` started
that build (ticket item 2). Error text now names the texture, its role (`base` / `overlay` /
`output` / …) and the reason, and states the failure is not retryable (items 3 and 4).

All 22 lock sites in the handler were converted, not only `combine_textures`. The audit found the
same leak at `resize_texture` and `channel_extract`, plus four *null-dereference* crash sites where
the platform read had no null check at all (`invert`, `desaturate`, `adjust_curves` memcpy from
`nullptr`; `channel_pack`'s indexed read). Loop bounds now come off the locked mip rather than
`UTexture2D::GetSizeX()` (the platform size, which an LOD bias can shrink below the source), and
`resize_texture` / `combine_textures` / `channel_extract` create their output *before* taking any
lock — `FTextureSource::Init` asserts Fatal on a locked source and `NewObject` re-allocates in
place when the output name resolves to an existing asset.

`Handlers/Asset/TexturePixelStats.cpp` was deliberately left alone: its `FTextureSource` lock is
linear with no early return between lock and unlock (null there means the lock was NOT taken), and
routing it through the guard would add a compilation flush to the read-only `texture.describe` path
for no defect fixed.

Files changed (all under `Plugins/PinWright/`):
- `Source/PinWright/Private/Handlers/Asset/TextureSourceMipLock.h` — new; the scoped lock.
- `Source/PinWright/Private/Handlers/Asset/TextureSourceMipLock.cpp` — new.
- `Source/PinWright/Private/Handlers/Material/TextureHandler.cpp` — 14 verbs converted; every
  `FBulkData::LockReadOnly` / `Unlock` and every raw `Source.LockMip` / `UnlockMip` removed.
- `Source/PinWright/Private/Tests/Assets/TestCombineTexturesSourceLock.cpp` — new; two regressions.

Verification a reviewer should run:
1. `PinWright.texture.*` automation group (compile first — this change was not built by the fix
   agent). The two new tests are `PinWright.texture.combine_textures.ReadsEditorSource` (inputs
   carry a colour in `Source` while their platform mips are resident-but-flat; a revert to the
   platform read yields black and fails) and
   `PinWright.texture.combine_textures.FailedLockReleasesSourceLock` (a combine that fails on the
   overlay must leave the base unlocked — proven by a following in-place `texture.invert`, which
   needs a ReadWrite lock the leaked read lock would refuse).
2. Live: `texture.create_noise_texture` twice (2048², seamless), then `texture.combine_textures`
   over the two with `blendMode: "Overlay"`. It must now SUCCEED rather than return
   `[TEXTURE_ERROR] Failed to lock texture data`. Issue the identical call a second and third time:
   the editor must stay up.
3. Live negative: `combine_textures` with an `overlayTexture` that has no editor source. The error
   must name the overlay and say retrying will not help, and a subsequent `texture.invert` on the
   base must still succeed.

## History
- `#1-filed` `OPEN` reporter — Hit while authoring asphalt and concrete source textures for the FPS environment on UE 5.8 / EAContentExamples58, after a critic pass found every concrete and asphalt surface was falling through to a borrowed sandstone map and the remedy was to generate real ones. Sequence was exactly the two calls above: four `texture.create_noise_texture` calls (2048², seamless, Worley and Perlin) all returned `saved:true` with the `.uasset` files present on disk; the first `combine_textures` over two of them returned `[TEXTURE_ERROR] Failed to lock texture data`; I verified the files on disk and re-issued the identical call, and the editor died on `IsUnlocked()`. Deepest PinWright frame `TextureHandler.cpp:2189` inside `ExecuteTextureAction`, reached from `AutoHandler_354_` at `:2929`. The inference that call 1 leaked the lock rests on the assertion being *already-locked* rather than *cannot-lock*, and on nothing else in the session having touched those two assets. Dedup: grepped the board for `combine_textures`, `LockReadOnly`, `IsUnlocked` and `BulkData`; the hits were all unrelated `*-no-disk-write` persistence tickets in other namespaces, none about texture bulk-data locking or this crash.
- `#2-fixed` `IN-REVIEW` developer — Replaced every FBulkData platform-mip lock in the texture handler with a scoped FTextureSource lock (`TextureSourceMip::FScopedMipLock`), so the leak is impossible by construction and the soft error's cause (reading the compressed platform mip) is gone too. See the Fix section for files, the wider audit, and the verification to run. Not compiled or run by the implementing agent.
