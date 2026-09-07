---
id: B-slate-window-readback-realloc-av-kills-shared-editor
title: "A Slate main-window pixel readback (`FSlateRHIRenderer::DrawWindowViewport_RenderThread` -> `RHIReadSurfaceData`) reallocates its `TArray<FColor>` into an `EXCEPTION_ACCESS_VIOLATION` and takes the shared editor down with seven agents on it"
status: OPEN
severity: Critical
category: bug
tags: [render, screenshot, capture, render-thread, crash, access-violation, slate, readback, multi-agent, shared-editor, python-execute]
encounters: 1
lastSeen: 2026-09-07T08:27:50Z
---

# The editor's own window readback crashed the render thread; every agent on the shared editor lost its session

Observed on the shared FPS editor at **2026-09-07 08:27:50 UTC**, `Saved/Logs/EAContentExamples58.log`.

**Attribution, stated plainly: I did not run the verb that crashed.** My last RPC on that editor was
`editor.close_asset` at 08:14, thirteen minutes earlier; my two `render.capture_asset_preview` calls
(768-1200 px, asset-editor preview, both closed) completed at 08:12 and 08:14 and are *not* on this
callstack — this one is the **Slate window** path, `SlateUI Title = EAContentExamples58 - Unreal
Editor`, not a preview viewport. What follows is what the log records, not a claim about which
agent's call it was.

## What the log records

For the ~20 s before the crash, one agent was running a per-frame Python callback that threw on
every tick:

```
[08.27.31 .. 08.27.43] LogPython: Error: Traceback (most recent call last):
  File ".../Intermediate/PinWright/Python/InlinePython_A46C78464B54C50D61F82BA22F2FAD93.py", line 14, in watch
  Exception: Actor: Internal Error - ObjectInstance is null!
```

~40 occurrences, one or two per frame, frames 931-952. Then, on frame 953:

```
[08.27.50:375] LogRHI: Error: Breadcrumbs 'RDGExecute_RenderThread'
 - SlateUI Title = EAContentExamples58 - Unreal Editor
 - RenderGraphExecute - Slate
 - Frame 124953
[08.27.50:375] LogWindows: Error: === Critical error: ===
[08.27.50:375] LogWindows: Error: Unhandled Exception: EXCEPTION_ACCESS_VIOLATION
                                  reading address 0x0000008000000084
```

Callstack, render thread, top down:

```
mi_usable_size()                                  ThirdParty/mimalloc/2.0.0/src/alloc.c:549
mi_heap_realloc_zero_aligned_at()                 alloc-aligned.c:122
mi_realloc_aligned()                              alloc-aligned.c:188
FMallocMimalloc::Realloc()                        MallocMimalloc.cpp:227
FMemory::Realloc()                                UnrealMemory.cpp:685
TSizedHeapAllocator<32,FMemory>::ForAnyElementType::ResizeAllocation()
UE::Core::Private::ReallocGrow<3, ...>()          Array.h:625        (D3D12RHI)
TArray<FColor,...>::SetNumUninitialized()         Array.h:2587       (D3D12RHI)
FD3D12DynamicRHI::RHIReadSurfaceData()            D3D12RenderTarget.cpp:633
FRHICommandListImmediate::ReadSurfaceData()       RHICommandList.h:4619
TRDGLambdaPass<FReadbackTextureParameters, FSlateRHIRenderer::DrawWindowViewport_RenderThread ...>
FRDGBuilder::ExecutePass() / ExecuteSerialPass() / Execute()
FSlateRHIRenderer::DrawWindows_RenderThread()     SlateRHIRenderer.cpp:1493
```

The faulting address `0x0000008000000084` is not a small-offset null deref; it reads like a
corrupted or already-freed allocation header being handed back to mimalloc's `realloc`.

## Why this is PinWright's problem and not just the engine's

`FSlateRHIRenderer::DrawWindowViewport_RenderThread`'s readback pass only runs when something asked
Slate to hand back the window's pixels — the editor-window screenshot path that `editor.screenshot`
and the level-capture verbs sit on. It ran **while a Python tick callback was raising an exception
every frame**, i.e. while the game thread was in a state no safe point had declared quiescent. The
two safe-point gaps that already have tickets are the shape of this one:

- `B-safepoint-tick-gate-inert-on-simpletickobjects-path` — a tick-registered callback is not
  covered by the gate.
- `B-python-execute-reentrant-gc-crash` — `python.execute` reaching the render/GC path from inside
  a tick.

Neither covers a **window readback** landing on the same frame.

## What was expected

That a window-pixel readback either runs at a safe point or refuses, the way
`render.capture_asset_preview` refuses to destroy a toolkit on its own capture stack
(`assetEditorCloseDeferred`). A readback that can be scheduled onto a frame where a per-tick
callback is throwing is a readback with no gate.

## Impact

**Critical, and it is the multi-agent multiplier that makes it so.** One agent's screenshot took
down an editor that seven agents were working in: unsaved level state, open asset editors, and
every in-flight compile went with it. In my own case it landed between an authored `.pwmodel` edit
and the `model.compile` that would have verified it, so the source and the baked asset are out of
step until an editor is back up.

## Repro

Not reduced. Ingredients present at the moment of the crash, from the log alone:

1. A per-frame Python callback registered by `python.execute` that raises every tick.
2. A Slate **main-window** pixel readback (`editor.screenshot` or a level capture) on the same frame.

## Suggested next step

Name the verb. `LogPinWrightSafePoint: Running '<verb>' ... inline` is written for every gated
call and would settle attribution in one line; it is absent from the 3 MB tail I read, so either
the crashing call was not gated or the line is not emitted on this path. Establishing which is the
first piece of work here.

## History
- `#1-filed` `OPEN` reporter — Filed at the moment of failure by the WEAPONS AR-mesh agent, whose
  session was killed by it. Evidence is `Saved/Logs/EAContentExamples58.log` only: the crash
  callstack, the breadcrumb naming the Slate main window, and the ~40 per-frame `watch` exceptions
  immediately preceding. **No PinWright verb is attributed** — the reporter did not run one in that
  window and could not recover a `LogPinWrightSafePoint` line for the call that did. A second
  editor process started at 08:29:59.
