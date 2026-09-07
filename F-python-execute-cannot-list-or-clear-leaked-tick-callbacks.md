---
id: F-python-execute-cannot-list-or-clear-leaked-tick-callbacks
title: "`python.execute` has no way to list or clear the tick/timer callbacks a script registered, so a leaked callback outlives PIE, the script, and every later slot — with no route to stop it but killing the editor"
status: OPEN
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
