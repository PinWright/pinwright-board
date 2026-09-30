---
id: B-python-pie-rpcs-run-locally
title: "UFUNCTIONs called through python.execute (and object.call_function) during PIE run every Server/Client RPC locally in the calling world: a client-side Server RPC never reaches the server, and a game RPC that re-sends from its own _Implementation recursed until a stack-overflow SIGSEGV killed the editor"
status: OPEN
severity: High
category: bug
tags: [python, python.execute, object.call_function, pie, multi-pie, listen-server, rpc, net, callspace, FEditorScriptExecutionGuard, GAllowActorScriptExecutionInEditor, crash, silent-false-success]
encounters: 1
costly: 1
lastSeen: 2026-09-30T09:44:29Z
---

# Scripted UFUNCTION calls turn PIE RPCs into local calls

UE 5.8.2 Linux, host `/sdb-disk/src/unreal/unreal-fpv-wt2` (PDS), plugin `2580e7f4`, editor started
offscreen. Listen-server PIE with one client (`editor.play {numClients: 2, netMode: "listen"}`), race on a
PIE map. The task drove race state from `python.execute` scripts that call game UFUNCTIONs on the
client's and the host's objects.

Two symptoms, one cause:

1. **A Server RPC called from a client world silently does nothing on the server.** The call returns
   normally and the script reports success. The `_Implementation` runs on the client's own copy of the
   object instead; the server never sees the call.
2. **A game RPC that re-sends from its own implementation recursed until the editor died.** From a
   client world the script called `URaceProgressComponent::SetTotalTime` on the client's player-state
   component. That function sends `Server_SetTotalTime` when `IsLocallyControlledClient()` is true
   (`PC->IsLocalController() && !PC->HasAuthority()`), and `Server_SetTotalTime_Implementation` calls
   `SetTotalTime` again. Over the network that is safe: the implementation runs on the server, where the
   check is false. Run locally on the client, the check stays true and the pair recurses forever:

```
LogPinWrightSafePoint: Running 'python.execute' (id=01a0f1b3-...) inline: no world is inside UWorld::Tick ...
LogCore: === Critical error: ===
Unhandled Exception: SIGSEGV: invalid attempt to write memory at address 0x00007ffd976b5e20
libUnrealEditor-CoreUObject.so!UObject::FindFunctionChecked(FName) const [.../ScriptCore.cpp:1536]
libUnrealEditor-App.so!URaceProgressComponent::Server_SetTotalTime(float) [.../RaceProgressComponent.gen.cpp:1249]
libUnrealEditor-App.so!URaceProgressComponent::Server_SetTotalTime_Implementation(float) [.../RaceProgressComponent.cpp:74]
libUnrealEditor-App.so!URaceProgressComponent::execServer_SetTotalTime(UObject*, FFrame&, void*) [.../RaceProgressComponent.gen.cpp:1295]
libUnrealEditor-CoreUObject.so!UFunction::Invoke(UObject*, FFrame&, void*) [.../Class.cpp:7595]
libUnrealEditor-CoreUObject.so!UObject::ProcessEvent(UFunction*, void*) [.../ScriptCore.cpp:2232]
libUnrealEditor-App.so!URaceProgressComponent::Server_SetTotalTime(float) [.../RaceProgressComponent.gen.cpp:1250]
... (the same five frames repeat until the stack is exhausted)
LogExit: Executing StaticShutdownAfterError
LogCore: FUnixPlatformMisc::RequestExit(1, GenericPlatformmallocCrash::Malloc.OutOfMemory)
```

The fault address is on the stack (`0x7ffd...`), so this is a stack overflow from the recursion. The
`Malloc.OutOfMemory` exit reason is printed during the shutdown after the error.

## Cause (verified in engine source)

- The Python plugin wraps every UFUNCTION it invokes in `FEditorScriptExecutionGuard`
  (`Engine/Plugins/Experimental/PythonScriptPlugin/Source/PythonScriptPlugin/Private/PyUtil.cpp:644`,
  `InvokeFunctionCall`, right before `InObj->ProcessEvent(...)`). The guard sets
  `GAllowActorScriptExecutionInEditor` for the whole call, including every C++ call nested inside it.
- `AActor::GetFunctionCallspace` returns `FunctionCallspace::Local` first thing when that global is set
  (`Engine/Source/Runtime/Engine/Private/Actor.cpp:5469-5474`), before looking at the world, the net mode
  or the role. `UActorComponent::GetFunctionCallspace` defers to the owning actor
  (`ActorComponent.cpp:1249-1258`), so components behave the same.
- Result: while a Python-invoked UFUNCTION is on the stack, every `Server`/`Client`/`NetMulticast` RPC on
  any actor or component in any PIE world executes locally and is never sent.

PinWright cannot change the engine guard, but it owns both routes an agent uses to call UFUNCTIONs:

- `python.execute` runs straight into the guard above. The PIE annotation it already adds
  (`pieActive: true` plus a `log` warning about sentinel values, `docs/wiki-src/python.md`) says nothing
  about RPCs, and `python.md` has no word on networking.
- `object.call_function` takes the **same** guard unconditionally, around both `ProcessEvent` calls
  (`Source/PinWright/Private/Handlers/Reflection/ObjectCallFunctionHandler.cpp:134` and `:275`). The
  comment says it is there so reflective calls run on editor-world actors, but it also applies to PIE
  objects, so the typed route has the same RPC behavior as Python.

## Expected

- `object.call_function`: take `FEditorScriptExecutionGuard` only when the target lives in an editor world
  (`WorldType::Editor` / `EditorPreview`). For a PIE or game world, call `ProcessEvent` without it, so
  RPCs follow the normal callspace and a Server RPC from a client object reaches the server. That gives
  agents one faithful way to call game RPCs.
- `python.execute`: when PIE is active, extend the existing `pieActive` warning to say that UFUNCTIONs
  called from the script run every RPC locally in the calling world, and point to `object.call_function`
  or to `editor.console_command {world: "server"}` for server-side changes. Document the same thing in
  `python.md` next to the other "calls that crash the editor" notes, including the re-send recursion as a
  known way to die.

**Workaround:** call the server-side function on the **server** world's copy of the object (the listen
server's player state for that pilot) instead of the client-side wrapper, or set replicated state
directly on the server object. Never call a function from a client-world script if it sends a Server
RPC.

## History
- `#1-rpc-recursion-kills-editor` `OPEN` reporter - Filed from a PDS race-results repro (listen PIE, host + 1 client, UE 5.8.2 Linux, offscreen editor, plugin `2580e7f4`). Server RPCs called from client-world scripts had no server-side effect, and `SetTotalTime` from the client world recursed through `Server_SetTotalTime` until SIGSEGV. Root cause traced to the engine's `FEditorScriptExecutionGuard` in `PyUtil.cpp:644` plus `AActor::GetFunctionCallspace`'s early return. `object.call_function` takes the same guard, so it is not a way around it today. Costly: one editor crash and restart, plus a rebuilt PIE session.
