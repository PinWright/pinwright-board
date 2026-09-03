---
id: B-texture-action-bulkdata-lock-assert-fatal
title: "`ExecuteTextureAction` calls `FBulkData::LockReadOnly` without checking IsUnlocked — a repeated texture RPC kills the whole editor"
status: IN-REVIEW
severity: Critical
category: bug
tags: [texture, crash, shared-editor]
---

# `ExecuteTextureAction` calls `FBulkData::LockReadOnly` without checking IsUnlocked — a repeated texture RPC kills the whole editor

A texture RPC that has already failed twice with a soft `TEXTURE_ERROR: Failed to lock
texture data` escalates on the next attempt into a fatal engine assertion, taking the
editor down with it. In a shared editor this kills every agent connected to it, not just
the caller who issued the texture call.

## Evidence (this checkout, 2026-09-02)

`Saved/Logs/EAContentExamples58.log`, all times UTC (machine is UTC+5):

    [22.32.18:690] LogPinWrightSubsystem: Warning: Automation request failed (TEXTURE_ERROR): Failed to lock texture data
    [22.32.20:202] LogPinWrightSubsystem: Warning: Automation request failed (TEXTURE_ERROR): Failed to lock texture data
    [22.32.45:280] LogWindows: Error: appError called: Assertion failed: IsUnlocked()
        [File:.../Runtime/CoreUObject/Private/Serialization/BulkData.cpp] [Line: 965]

Callstack, top frames:

    UnrealEditor-CoreUObject.dll!FBulkData::LockReadOnly()            BulkData.cpp:965
    UnrealEditor-PinWright.dll!ExecuteTextureAction()                 TextureHandler.cpp:2189
    UnrealEditor-PinWright.dll!RunTextureAction()                     TextureHandler.cpp:2703
    UnrealEditor-PinWright.dll!AutoHandler_354_()                     TextureHandler.cpp:2929
    UnrealEditor-PinWright.dll!FRpcDispatcher::ProcessRequest()       RpcDispatcher.cpp:813

Two soft failures at 22:32:18 and 22:32:20 left the bulk data locked; the third call at
22:32:45 hit `LockReadOnly` on the still-locked payload and asserted.

## Why the soft error is the real defect

The two `TEXTURE_ERROR` returns are the handler noticing it could not get the data — but
it evidently leaves the lock state inconsistent on that path, so the *next* caller
crashes rather than getting a third clean error. The symptom therefore lands on whoever
happens to call next, which in a multi-agent editor is usually a different stream from
the one that caused it.

## Impact

Fatal, editor-wide. Every agent connected to that editor loses in-memory work that was
compiled but not yet saved. This instance cost the AI stream two rewritten Blueprint
functions (`AIC_Enemy.UpdateDwell`, `AIC_Enemy.UpdateSquadRole`) that had returned
`compiled:true` fourteen seconds before the crash and were never flushed to disk.

## Fix proposed at filing

`TextureHandler.cpp:2189` — check `IsUnlocked()` before `LockReadOnly()` and return
`TEXTURE_ERROR` instead of asserting; and on every early-return path out of
`ExecuteTextureAction`, release any lock the function took (RAII guard rather than manual
unlock), so a failed call cannot leave the payload locked for the next one.

## Workaround

None available to the caller: the crashing call is not necessarily the one that broke the
lock. Agents sharing an editor should save after every compile rather than batching
saves, since any co-tenant's texture call can end the process.
## Fix

True as filed, with one correction to the mechanism: an `IsUnlocked()` check before
`LockReadOnly` would have been the wrong fix. `FBulkData::LockReadOnly` (BulkData.cpp:963-978)
sets `LOCKSTATUS_ReadOnlyLock` and only then returns `GetDataBufferReadOnly()`, which is null for
a compressed / GPU-resident payload — so the soft failures at 22:32:18 and 22:32:20 DID take the
lock, and the handler's `if (BaseData) ... Unlock()` guard skipped the release in precisely that
case. The third call then asserted. Guarding the acquire would have converted the crash into a
permanent soft failure for that asset without ever releasing the leaked lock.

Fixed by removing the lock rather than checking it. Every read and write in `TextureHandler.cpp`
now goes through `TextureSourceMip::FScopedMipLock` over the editor `FTextureSource`
(CPU-resident, uncompressed), built on the engine's `FTextureSource::FMipLock` RAII pair; there is
no `FBulkData` lock left in the file. Full rationale, the wider audit (the same leak at
`resize_texture` / `channel_extract`, four additional null-dereference sites, the create-before-lock
reordering) and the reviewer's verification steps are recorded on the sibling ticket
`B-combine-textures-leaks-bulkdata-lock-then-crashes`, which covers the same root cause.

Files changed (all under `Plugins/PinWright/`):
- `Source/PinWright/Private/Handlers/Asset/TextureSourceMipLock.h` / `.cpp` — new; the scoped lock.
- `Source/PinWright/Private/Handlers/Material/TextureHandler.cpp` — 14 verbs converted.
- `Source/PinWright/Private/Tests/Assets/TestCombineTexturesSourceLock.cpp` — new; two regressions.

Verification a reviewer should run: compile the plugin (the implementing agent did not build), run
the `PinWright.texture.*` automation group, then live-drive `texture.combine_textures` three times
in a row over two freshly created noise textures — the first call must succeed rather than return
`Failed to lock texture data`, and the editor must survive the repeats. Also confirm a genuine
refusal (an overlay with no editor source) names the offending texture and side, and that a
following `texture.invert` on the base still succeeds, which is what proves nothing stayed locked.

## History
- `#1-fixed` `IN-REVIEW` developer — Root cause confirmed by reading `FBulkData::LockReadOnly`: the lock is taken before the possibly-null pointer is returned, so the pointer-conditional unlock skipped exactly the leaking path. Replaced all platform-mip `FBulkData` locking in `TextureHandler.cpp` with a scoped `FTextureSource` lock helper. Not compiled or run by the implementing agent.
