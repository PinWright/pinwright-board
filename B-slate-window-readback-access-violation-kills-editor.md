---
id: B-slate-window-readback-access-violation-kills-editor
title: "A Slate WINDOW screenshot readback faults on the render thread inside `TArray<FColor>::SetNumUninitialized` and kills the shared editor - a different path from the hit-proxy `ColorRT` assert"
status: IN-REVIEW
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
- `#2-duplicate-of-realloc-ticket` `OPEN` reporter — **Duplicate. The canonical ticket for this crash is `B-slate-window-readback-realloc-av-kills-shared-editor`**, which is the fuller record (it carries the triggering verb, a second History entry from the ENV stream naming `material.compile_mgir`, and the cost field). Same instant 08:27:50Z, same render-thread `EXCEPTION_ACCESS_VIOLATION`, same stack through `FSlateRHIRenderer::DrawWindowViewport_RenderThread` → `RHIReadSurfaceData`. Both tickets were filed within minutes by two agents I dispatched in parallel, neither able to see the other's filing; the duplication is my coordination failure as the dispatcher, not two defects. **All evidence unique to this ticket has been copied into `#3` of the canonical**, so this one can be closed without loss. I have NOT changed `status` myself — only the user closes tickets — and I have NOT incremented `encounters` anywhere, because one crash observed by two agents is one occurrence and counting observers would inflate a Critical ticket. Recommend: close this as a duplicate and keep the realloc ticket.
- `#3-closed-as-duplicate-fixed-by-canonical-change` `IN-REVIEW` developer — **Closed as a duplicate of `B-slate-window-readback-realloc-av-kills-shared-editor`, which `#2` already identified as canonical; no separate code was written for this ticket and none is needed.** The single change made on the canonical ticket covers this stack in full: the shared helper `PinWrightScreenshotUtils::TakeSlateScreenshot` (`Source/PinWright/Private/Utils/ScreenshotUtils.h` / `.cpp`) now owns the readback destination in a buffer with static storage duration and disarms the Slate renderer's pending screenshot state on every exit path, and all four former `FSlateApplication::Get().TakeScreenshot` call sites route through it — `Source/PinWright/Private/Handlers/Editor/EditorWindowHandlers.cpp:1033`, `Source/PinWright/Private/Handlers/Drive/DriveEditorChrome.cpp:581`, `Source/PinWright/Private/Handlers/UI/WidgetDesignerScreenshotHandler.cpp:323`, `Source/PinWright/Private/Utils/ScreenshotUtils.cpp:585`. The regression test is `PinWright.render.slate_screenshot.LeavesNoPendingRendererState` in `Source/PinWright/Private/Tests/Render/TestSlateScreenshotPendingState.cpp`; counterfactual and mechanism are recorded once, on the canonical ticket's `#4`, and are not restated here. **This ticket's own two distinguishing observations are answered by that fix**: the fault inside `TArray<FColor>::SetNumUninitialized` was the destination array being grown through a pointer whose storage had already been freed, not a garbage rect (`FSlateApplication::TakeScreenshot` measures the rect from the widget's arranged geometry, and the engine clamps it at `SlateRHIRenderer.cpp:1184-1190`), so the "keep window screenshots to a constant width/height" workaround in the body was aimed at the wrong input and is now moot; and the missing dispatch line for the faulting frame is explained rather than merely noted — the readback was armed by an earlier call and executed on a later Slate frame, so there genuinely was no RPC of ours in flight. **`encounters` left at 1 and severity left at Critical**, per `#2`: one crash observed by two agents is one occurrence. Verify against the canonical ticket, not this one.
