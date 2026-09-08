---
id: B-editor-screenshot-pie-gpu-pagefault-during-shader-late-association
title: "`editor.screenshot` on the PIE path forces a readback while the RHI is still late-associating shaders, and the GPU page-faults (Aftermath MMU fault in a fragment shader), killing the shared editor"
status: OPEN
severity: Critical
category: bug
tags: [render, screenshot, pie, readback, crash, gpu, page-fault, aftermath, shader-compilation, warmup, shared-editor, multi-agent]
encounters: 1
costly: 1
lastSeen: 2026-09-08T15:26:08Z
---

# A PIE screenshot taken while shaders are still late-associating takes the GPU down

`editor.screenshot` issues its game-viewport readback immediately. It has no warmup or settle step,
unlike `render.capture_open_level`, whose response carries a
`viewport.warmup { settled, settleRounds, meanLuminanceDelta, pixelChangeMeasured, settleMs }` block
and demonstrably re-draws until the frame stops changing. When the readback lands in the same frame
window as the RHI's "late shader associations" pass, the draw references shader resources that are
not mapped yet and the GPU faults.

## Observed (this checkout, 2026-09-08)

    editor.screenshot {filename:"b10_burst_final.png", width:1280, height:720,
                       exposure:{mode:"fixed", ev100:-3}}
    -> Editor stream ended early (stream read failed: timed out)

Log, in order, one second apart:

    15:26:07.602  LogD3D12RHI: Starting late shader associations...
    15:26:08.603  LogD3D12RHI: Late shader associations ignored 888 pipelines based on frame fences
    15:26:08.624  LogD3D12RHI: Created late shader associations, took 1022ms
    15:26:08.626  LogRHICore: Error: Shader debug data not found (2432512719049712685)   [x4]
    15:26:08.629  LogNvidiaAftermath: Error: ... Status: PageFault, Engine Reset: True
                    Type: AddressTranslationError   Access: Read   Engine: Graphics
                    Fault Name: MMU Fault Error
                    Shader GPU PC Address: fragment_02 @ 0x00000320
    15:26:08.630  LogD3D12RHI: Error: PageFault at VA "0xFFFFDF383E3F1000" (GPU 0)
                    Last completed frame ID: -1 (cached: 11216) - Current frame ID: 11221

Aftermath dump written to `Saved/Logs/D3D12.0.2026.09.08-19.26.07.nv-gpudmp`. Afterwards
`editor.status` returned `EDITOR_NOT_READY` with the game thread not having ticked for 131 s, then
224 s — the editor never recovered, and every agent on the shared editor lost its session.

**This is not VRAM exhaustion.** The same error block reports `Local Used 3684.94 MB` against
`Local Budget 4763.38 MB`, and `System Used 300.34 MB` against `47541.73 MB`. It is a page fault at
an unmapped virtual address, with `Engine Reset: True`.

## Scene state at the moment of capture

Worth stating because it is what put fresh shaders in flight — and because none of it should be able
to take the GPU down:

- PIE running, three AI characters engaging a target dummy.
- `TracerEvery = 1` on each weapon (PIE instances only), so every shot spawns a tracer.
- `SetGlobalTimeDilation(0.03)`, to widen the muzzle-flash window enough to photograph.
- The camera had just been moved to a new vantage by setting the PIE view target's location and
  control rotation, so the frame was newly composed and its materials newly relevant.

The combination is "first frame that draws this VFX from this angle" — precisely when shaders get
compiled and late-associated.

## Ask

1. **Give `editor.screenshot` the settle step `render.capture_open_level` already has.** Before the
   readback, wait for the shader compilation queue to drain (`GShaderCompilingManager->IsCompiling()`
   / `FinishAllCompilation`) and for the frame to stop changing, and report a `warmup` block the way
   the level path does. A capture verb that draws a frame nobody has drawn before is exactly the
   caller that cannot know to wait.
2. Failing that, at minimum **report** that shaders were still compiling when the frame was taken,
   so a caller can retry rather than ship an incomplete frame — or, as here, lose the editor.

## Workaround

Draw the frame before capturing it: move the camera, let several frames tick at normal time
dilation, and only then call `editor.screenshot`. Avoid capturing the first frame after a camera
move that reveals new VFX. This is guesswork, not a fix — there is no way to ask whether shaders are
still compiling.

## Notes

- **Distinct from the two Slate readback tickets.** `B-slate-window-readback-access-violation-kills-editor`
  and `B-slate-window-readback-realloc-av-kills-shared-editor` are both the 2026-09-07 08:27:50Z
  event: a CPU-side `EXCEPTION_ACCESS_VIOLATION` inside `TArray<FColor>::SetNumUninitialized` on the
  render thread, in `FSlateRHIRenderer::DrawWindowViewport_RenderThread`. This one is a GPU-side MMU
  page fault reported by Aftermath with an engine reset, on the `editor.screenshot` game-viewport
  path, a day later. Same verb family, different failure.
- Two editor deaths in two days on a PIE-path capture readback is the pattern worth acting on even
  if each individual fault is a driver's to fix.
- Severity Critical per the board README's editor-crash clause, not by reach.
