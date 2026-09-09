---
id: F-python-execute-cannot-list-or-clear-leaked-tick-callbacks
title: "`python.execute` has no way to list or clear the tick/timer callbacks a script registered, so a leaked callback outlives PIE, the script, and every later slot — with no route to stop it but killing the editor"
status: IN-REVIEW
severity: High
category: feature
tags: [python, tick, callback, lifetime, pie, shared-editor, hazard, wiki]
encounters: 1
lastSeen: 2026-09-07T08:27:50Z
---

# A callback registered by `python.execute` can only be removed by the handle that registered it

`unreal.register_slate_post_tick_callback(fn)` (and its `pre_tick` and timer siblings) returns a
handle, and `unregister_slate_post_tick_callback(handle)` is the only way to remove it. The handle
lives in the calling script's namespace. `python.execute` runs each call in a fresh inline module
(`Intermediate/PinWright/Python/InlinePython_<hash>.py`), so when that call returns, **the handle is
gone and the callback is not**. There is no verb — and no `unreal` API — that enumerates registered
callbacks or clears them.

Consequences, in order of how bad they get:

1. The callback keeps ticking after the script ends.
2. It keeps ticking after **PIE ends**, and if it captured a PIE actor every tick now throws.
3. It keeps ticking through **every later slot**, in a shared editor, for streams that have nothing
   to do with it.
4. The only remedy is restarting the editor, which is exactly what a shared editor cannot afford.

## What it cost here (this checkout, 2026-09-07)

A callback named `watch`, registered from an inline `python.execute` during an AI slot, threw
`Actor: Internal Error - ObjectInstance is null!` on its captured PIE actor handle starting
**06:54:54.657Z** — 0.2 s after `UWorld::CleanupWorld for T_AI`, in the PIE teardown frame — and
went on throwing **94,358 times over 93 minutes**, through four other streams' slots, until the
editor died at 08:27:50Z in `FD3D12DynamicRHI::RHIReadSurfaceData` on the render thread.

**Whether the leak caused that access violation is unproven** and this ticket does not claim it did.
The leak is the defect on its own terms: 30 exceptions a second for an hour and a half, in a process
four other agents were working in, with no supported way for any of them to find or stop it. The
log line names the inline script by content hash — `InlinePython_A46C7846…` — and those files are
deleted after execution, so even the forensic route back to the offending source is gone.

## Ask

Two pieces, the first much more important than the second:

1. **`python.callbacks {action: "list" | "clear"}`** — list the tick/timer callbacks currently
   registered from the Python layer (handle, kind, and the inline-script hash or module that
   registered it), and clear them, individually or all. "Clear all" is the one that turns an
   editor restart into a one-line recovery.
2. **Auto-clear on PIE end**, or at minimum a warning: when PIE tears down and Python-registered
   tick callbacks are still live, log one line naming them. Silence is what let this run for 93
   minutes — every one of those 94,358 lines named the symptom and none named the cause.

Retaining the inline script for callbacks that outlive their call, instead of deleting it, would
also make the hash in the error message resolvable.

## Wiki

`python.execute` should carry a hazard note: a callback or timer registered inside the call
**outlives the call**, nothing lists or clears it, and losing the handle means losing the only way
to remove it — so keep the handle in a module (not in the inline script), unregister in a `finally`
and on the first exception inside the callback, and never register from a script that ends before
PIE does. The equivalent rule is now `Docs/fps/PLAN.md` rule 16 in this project, which is the wrong
place for it: it is a property of the verb, not of this map.

## Workaround

Discipline only, and it does not help once the handle is lost. Register from a **module** kept in
`sys.path` so a later `python.execute` can `import` it and call its `stop()`; have the callback
unregister itself on any exception and when the PIE world stops resolving; and prove it is gone
before releasing a shared-editor slot. If a callback leaks anyway with no module holding the handle,
there is nothing to do but restart the editor.

## History
- `#1-python-callbacks-verb` `IN-REVIEW` developer — The engine's Python handle lists (`PySlateUtil::PythonPre/PostTickCallbackHandles`, PySlate.cpp) are file-scope statics in a Private module with no accessor, so registrations are captured at the call instead: new `Content/Python/pinwright_callbacks.py` wraps `unreal.register_slate_post_tick_callback` / `register_slate_pre_tick_callback` / `register_python_shutdown_callback` (and their `unregister_*` pairs), keeps the engine handle, and mirrors metadata into C++ through the new reflected `Public/PinWrightPythonCallbackLibrary.h` + `Private/PinWrightPythonCallbackLibrary.cpp`. Records live in C++ (`Private/Utils/PythonCallbackRegistry.h/.cpp`) so the two readers that must not re-enter the interpreter — the EndPIE warning and python.execute's own leak count — are pure C++ reads; that file also owns the one-shot shim install, the `FEditorDelegates::EndPIE` warning naming every survivor (id, kind, slot, invocations, source) in the new `LogPinWrightPythonCallbacks` category, and `FScopedRequestSlot`, which attributes registrations to a request and counts survivors by allocation watermark rather than request id. New verb `python.callbacks` (`Private/Handlers/System/PythonCallbacksHandler.cpp`) takes `action: list|clear` plus `ids: [...]` or `all: true` — clear is never implied — and reports `cleared`/`failed`/`notFound` measured by snapshotting the registry before and after, never by trusting the script's own report. `Private/Handlers/System/PythonExecuteHandler.cpp` installs the shim before the sys.modules snapshot (so the restore cannot drop it), wraps the interpreter call in `FScopedRequestSlot`, and adds `leakedCallbacks` to the response plus a `Warning` log entry when it is nonzero, or a different one when tracking could not install so an absent field never reads as zero. `Private/Handlers/ErrorCodes.h` gains `ERR_PYTHON_CALLBACK_TRACKING_UNAVAILABLE`; `Private/PinWrightModule.cpp` drops the EndPIE binding on shutdown. Documented on `docs/wiki-src/python.md` under a new `## Callbacks a script leaves behind` section (above the first H3) plus a `### python.callbacks` method section and a `leakedCallbacks` note on `### python.execute`, including what the shim cannot see: callbacks registered before tracking started, world timers, and non-Python registrations. Regression test `PinWright.python.callbacks.LeakedTickCallbackIsListedAndCleared` (`Private/Tests/Infra/TestPythonCallbackTracking.cpp`). Counterfactual: it registers a counting probe through `python.execute`, broadcasts `FSlateApplication::Get().OnPostTick()` by hand and reads the probe's own counter back from Python, then clears through `python.callbacks` and broadcasts again — if `RequestClear` stopped reaching `unreal.unregister_slate_post_tick_callback`, or the shim stopped holding the engine handle it needs, the record would still vanish from PinWright's registry and every other assertion would still pass while the counter advanced to 2, so `the engine no longer invokes the cleared callback` fails.
- `#2-verifier-gap-fixes` `IN-REVIEW` developer — Five gaps from verification, all closed. (1) `Private/Handlers/System/PythonExecuteHandler.cpp` deleted the private-scope temp script before the leak count was read, so every leaked record's `source` named a file that no longer existed — the ticket's own forensic dead end. `CountSurviving()` now runs before the delete, the script is retained exactly when the count is nonzero, and the response reports it as `retainedScript` (named in the warning too). (2) `Private/Utils/PythonCallbackRegistry.cpp` never memoised a failed install, so every `python.execute` on a broken host re-ran the import and wrote another traceback; the failure is now sticky, and `EnsureTrackingReady(bRetryFailedInstall)` lets only the explicit `python.callbacks` verb retry — so fixing the cause still costs no editor restart. (3) `Private/Dispatch/SafePoint.cpp` gains `python.callbacks` to the tick-unsafe table: its payload is fixed, but both paths still put a Python frame on the handler's stack. (4) `BuildInstallScript` interpolated the plugin content path into the command, and `ExecPythonCommandEx` routes to `RunFile` if `.py` appears anywhere in it (`PythonScriptPlugin.cpp:828`) — a plugin installed under a `.py` directory would have silently never installed the tracker; the `sys.path` insert is dropped (the engine already registers `<Plugin>/Content/Python` from `RegisterModulePaths`) and the script is now the constant `__import__('pinwright_callbacks').install()`. (5) `Shutdown()` left `bShimInstalled`/`Records` set while the Python side kept `_ENTRIES` and its `_pinwright_tracked` markers, so a module unload+reload made pre-reload callbacks invisible and unclearable; `Shutdown()` now clears the C++ state and `Content/Python/pinwright_callbacks.py` gained `_readopt()`, which re-files surviving entries under fresh ids — guarded by a new `IsTracked` probe (registry + reflected library) so it is a no-op on an ordinary first install and cannot duplicate a record. `_ENTRIES` values became dicts carrying a one-element id cell that the tick wrapper reads through, so re-adoption also follows the invocation counter. Second regression test `PinWright.python.callbacks.LeakedPrivateScriptStaysOnDisk` locks (1): counterfactual — restore the unconditional delete and both `retainedScript` and the on-disk existence check fail while every other assertion in the file still passes. `Private/Handlers/ErrorCodes.h` `=` alignment corrected.
