---
id: B-slate-window-readback-access-violation-kills-editor
title: "A Slate WINDOW screenshot readback faults on the render thread inside `TArray<FColor>::SetNumUninitialized` and kills the shared editor - a different path from the hit-proxy `ColorRT` assert"
status: OPEN
severity: Critical
category: bug
tags: [render, screenshot, screenshot_window, readback, crash, render-thread, access-violation, shared-editor, multi-agent, slate]
encounters: 1
lastSeen: 2026-09-07T08:27:50+00:00
---

# A Slate window readback access-violates on the render thread and takes the shared editor down

Observed on the shared FPS editor at **2026-09-07 08:27:50 UTC**, from
`Saved/Logs/EAContentExamples58-backup-2026.09.07-08.27.50.log:293313-293336`.

```
08:27:50:375  LogRHI: Error: Breadcrumbs 'RDGExecute_RenderThread'
                - SlateUI Title = EAContentExamples58 - Unreal Editor
                - RenderGraphExecute - Slate
                - Frame 124953
08:27:50:375  LogWindows: Error: Unhandled Exception:
                EXCEPTION_ACCESS_VIOLATION reading address 0x0000008000000084
```

Callstack, rendering thread, innermost first:

```
mi_usable_size()                                          mimalloc alloc.c:549
mi_heap_realloc_zero_aligned_at()                         alloc-aligned.c:122
FMallocMimalloc::Realloc()                                MallocMimalloc.cpp:227
TSizedHeapAllocator<32,FMemory>::ResizeAllocation()        ContainerAllocationPolicies.h:863
UE::Core::Private::ReallocGrow<3,...>()                    Array.h:625
TArray<FColor,...>::SetNumUninitialized()                  Array.h:2587
FD3D12DynamicRHI::RHIReadSurfaceData()                     D3D12RenderTarget.cpp:633
FRHICommandListImmediate::ReadSurfaceData()                RHICommandList.h:4619
TRDGLambdaPass<FReadbackTextureParameters,
  `FSlateRHIRenderer::DrawWindowViewport_RenderThread'::<lambda_2>>::Execute()
FRDGBuilder::ExecutePass() / ExecuteSerialPass() / Execute()
FSlateRHIRenderer::DrawWindows_RenderThread()              SlateRHIRenderer.cpp:1493
RenderingThreadMain()
```

## Why this is not `B-capture-open-level-hitproxy-colorrt-assert-kills-editor`

That ticket's fault is `FViewport::GetRawHitProxyData` -> `FRHIRenderPassInfo` ->
`Assertion failed: ColorRT`, on a **scene** viewport, and its fix took the capture pump
off the hit-proxy path. This one is:

- a **Slate WINDOW** readback (`FSlateRHIRenderer::DrawWindowViewport_RenderThread`'s
  `FReadbackTextureParameters` RDG pass), not a scene viewport and not a hit-proxy map;
- an **access violation inside the destination array's realloc**, not a null render
  target - `TArray<FColor>::SetNumUninitialized` is growing the CPU-side destination
  buffer in `RHIReadSurfaceData` when it faults, so the failing input is the readback
  **rect / element count**, not the source texture;
- `0x0000008000000084` is not a plausible heap pointer, so either the array's allocation
  is already corrupt or the requested count is garbage.

The breadcrumb names the main editor window by title, so the thing being read back is the
whole editor window - the `editor.screenshot_window` / `editor.screenshot` family, not
`render.capture_asset_preview` (which reports `renderer:"sceneViewportReadPixels"`).

## What the log shows around it

The last four PinWright dispatches before the fault, all `inline`:

```
08:24:00:817  Running 'render.capture_asset_preview' (id=7a161fe6-...)
08:24:19:004  Running 'render.capture_asset_preview' (id=edd3a52c-...)
08:26:43:471  Running 'material.compile_mgir'        (id=0fe8c9f9-...)
   (no dispatch line for the faulting frame)
```

**No `LogPinWrightSafePoint: Running '...'` line accompanies the faulting frame**, which is
itself worth a look: either the window-screenshot path does not go through the safe-point
gate, or the readback was enqueued by an earlier call and executed on a later frame - and in
both cases the gate cannot serialise it against anything else.

For 18 seconds immediately before the crash (08:27:25 -> 08:27:43, frames 913-952) a
`python.execute` "watch" loop was raising, at ~3 Hz, once per frame:

```
LogPython: Error: File ".../InlinePython_A46C78464B54C50D61F82BA22F2FAD93.py", line 14, in watch
LogPython: Error: Exception: Actor: Internal Error - ObjectInstance is null!
```

That is a caller polling a **stale actor handle** every frame. Recorded as adjacency, not
as a claimed cause: a dangling `UObject` being touched per frame is the kind of thing that
corrupts a heap that a render-thread realloc then trips over, but nothing here proves it.

## Expected

A window readback either succeeds or fails; it never access-violates on the rendering
thread. Concretely: `RHIReadSurfaceData`'s destination `TArray<FColor>` should be sized
from a rect the caller validated against the window's **current** backbuffer extent, and
the verb should refuse (retryably) if the window resized or was destroyed between the
request and the RDG pass. A shared-editor verb must not be able to kill the process.

## Cost

One shared editor, mid-session, with at least four streams live in it (WEAPONS mesh work,
a Blueprint stream running `compile_bpir`, a material stream running `material.compile_mgir`
at 08:26:43, and whatever owned the python watch loop). Cost to this reporter was nil - the
pistol mesh work was already compiled and written to disk at 08:15:56 UTC and re-verified
afterwards - but `material.compile_mgir` had run 67 s earlier with no `asset.save` behind it
in the log, so that stream probably lost work.

## Workaround

None from the caller's side. Until it is understood, prefer
`render.capture_asset_preview` / `render.capture_open_level` over the window-screenshot
family for evidence, since those read a scene viewport rather than the editor window, and
keep window screenshots to a **constant** width/height - a varying readback size is the
only caller-visible input to the element count that faults here.

## History

- `#1-filed` `OPEN` WEAPONS (pistol mesh, bystander) - Filed at the moment the editor
  became unreachable. My last successful PinWright call was a `model.validate`; my five
  captures in this session were all `render.capture_asset_preview`
  (`renderer:"sceneViewportReadPixels"`, `warmup.settled:true`, `assetEditorClosed` chased
  with an explicit `editor.close_asset` after each), which is not the path in the callstack.
  I did not make the call that faulted and cannot name it - the log carries no dispatch line
  for that frame, which is half of what makes this worth filing. Evidence:
  `Saved/Logs/EAContentExamples58-backup-2026.09.07-08.27.50.log`, lines 293313-293336 for
  the fault and 288957-292777 for the dispatch history. The editor was restarted by someone
  else at 08:29:45 UTC (`Saved/Logs/EAContentExamples58.log` mtime); I did not restart it.
