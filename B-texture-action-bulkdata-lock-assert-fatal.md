---
id: B-texture-action-bulkdata-lock-assert-fatal
title: "`ExecuteTextureAction` calls `FBulkData::LockReadOnly` without checking IsUnlocked — a repeated texture RPC kills the whole editor"
status: OPEN
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

## Fix

`TextureHandler.cpp:2189` — check `IsUnlocked()` before `LockReadOnly()` and return
`TEXTURE_ERROR` instead of asserting; and on every early-return path out of
`ExecuteTextureAction`, release any lock the function took (RAII guard rather than manual
unlock), so a failed call cannot leave the payload locked for the next one.

## Workaround

None available to the caller: the crashing call is not necessarily the one that broke the
lock. Agents sharing an editor should save after every compile rather than batching
saves, since any co-tenant's texture call can end the process.
