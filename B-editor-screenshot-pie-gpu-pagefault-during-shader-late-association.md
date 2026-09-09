---
id: B-editor-screenshot-pie-gpu-pagefault-during-shader-late-association
title: "`editor.screenshot` on the PIE path forces a readback while the RHI is still late-associating shaders, and the GPU page-faults (Aftermath MMU fault in a fragment shader), killing the shared editor"
status: IN-REVIEW
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

## History

- `#1-capture-readiness-gate` `IN-REVIEW` developer — This ticket carried no `## History` section when the fix landed, so the section starts here rather than at the filing entry. Added a shared capture-readiness gate, `Source/PinWright/Private/Utils/CaptureReadinessGate.h/.cpp` (`PinWrightCaptureReadiness`): a bounded 20 s PUMPING drain of `GShaderCompilingManager` + `FAssetCompilingManager` reusing the plugin's existing pump (`Utils/AssetCompilePump.h::AdvanceOnGameThread`), a `shadersCompiling` block (`measured`, `shaderCompilerAvailable`, `pendingAtEntry`, `ready`, `timedOut`, `shaderJobsAtEntry`/`Remaining`, `assetCompilationsAtEntry`/`Remaining`, `pumpRounds`, `drainMs`, `budgetMs`, `readinessWarning`), and a refusal message. Wired into `Source/PinWright/Private/Handlers/Editor/ViewportHandler.cpp` (`editor.screenshot`, before `StartJob` so both branches are covered and a refusal is a structured error rather than a bare code on a ticket) and `Source/PinWright/Private/Handlers/Render/RenderHandler.cpp` (`render.capture_open_level`, immediately before `CaptureEditorViewportToPng`); both publish the block under `viewport.shadersCompiling`. **Shader work still pending after the drain refuses the readback** with the new `CAPTURE_NOT_READY` code (`Handlers/ErrorCodes.h`, documented in `Docs/error-code-catalog.md`); **asset compilation still pending warns and shoots**, because a queued texture cannot page-fault the GPU and refusing there would turn every capture in a busy editor into an error. `GShaderCompilingManager->IsCompiling()` is folded in beside `GetNumRemainingJobs()` because the latter does not count shader maps awaiting the game-thread finalize pass, which is exactly the late-association window. Docs: `Docs/wiki-src/render.md`, `Docs/wiki-src/editor.md`. Tests `PinWright.render.capture_readiness.PendingShaderWorkRefusesTheReadback`, `.BlockIsPublishedAndAbsentWhenUnmeasured`, `.BothVerbsGateBeforeTheReadback` (`Source/PinWright/Private/Tests/Render/TestCaptureReadinessGate.cpp`), driving the injected `FScopedPendingWorkProbe` seam so the drain loop, block and refusal decision run their production paths without a GPU. Counterfactual: revert `ShouldRefuseReadback` (or the drain loop's timeout bookkeeping) in `Utils/CaptureReadinessGate.h/.cpp` and `PendingShaderWorkRefusesTheReadback` fails, because a probe reporting three permanently pending shader jobs then reports `ShouldRefuseReadback() == false` and the verb would issue the readback into a compiling RHI exactly as it did on 2026-09-08; move either `DrainBeforeReadback()` call below its capture call and `BothVerbsGateBeforeTheReadback` fails on the source-offset comparison.

- `#2-mechanism-reattributed-flush-mitigation-added` `IN-REVIEW` developer — **`#1` attributed this to the wrong mechanism; that claim is withdrawn from the code, the docs and the tests.** Verified in engine source: `Starting late shader associations...` / `ignored N pipelines based on frame fences` / `Created late shader associations` are emitted only by `CreateShaderAssociations` (`D3D12NvidiaExtensions.cpp` ~62-229), whose single call site is `RHICoreNvidiaAftermath.cpp` ~851-857 **inside the Aftermath GPU-crash-dump decode handler** ("Allow association after a fault"); the `Shader debug data not found` lines that follow in the reported log are the same handler. Those lines are therefore **decoder output after the fault, not a pre-fault state**, and no compile gate could have prevented what they accompany. Aftermath's own verdict — `PageFault` / `AddressTranslationError` / `Read` / fragment shader — points at a **resource-lifetime or descriptor-residency fault around the readback**. A PSO-precache drain was also considered and rejected as dead code: precache is hardcoded off under `WITH_EDITOR` (`PipelineStateCache.cpp` ~4951-4959). **What the gate is now, and is documented as:** a frame-quality precondition — a material whose shader map has not landed renders as the DEFAULT material, and such a frame reads settled, non-blank and clean on every other honesty field, which `warmup` cannot catch because a stand-in material is stable. It refuses with `CAPTURE_NOT_READY` rather than shipping stand-ins. **Drain-loop defect fixed:** it waited while `ShaderJobs > 0 || AssetCompilations > 0`, so any normal post-map-load asset queue burned the full 20 s on the game thread on EVERY screenshot and then shot the frame anyway; it now waits on shader work only and reports asset counts through `assetCompilationWarning` without blocking (`Source/PinWright/Private/Utils/CaptureReadinessGate.h/.cpp`). **Placement fixed:** the gate now runs after the pose apply and the warm-up settle and immediately before the readback — on the level paths via the capture's `BeforeFinalFrame` hook, composed onto (never replacing) any caller's hook, so `effect.step_and_capture`'s world freeze still runs first (`Source/PinWright/Private/Handlers/Render/RenderHandler.cpp`, `Source/PinWright/Private/Handlers/Editor/ViewportHandler.cpp`). **`ui.screenshot` was ungated and is now gated and publishes both blocks** (`Source/PinWright/Private/Handlers/UI/UiHandler.cpp` ~189). **The mitigation aimed at the ACTUAL mechanism:** one shared readback preamble, `PinWrightScreenshotUtils::FlushBeforeReadback` (`Source/PinWright/Private/Utils/ScreenshotUtils.h/.cpp`, beside the landed Slate-readback helper rather than duplicating it), calling `FlushRenderingCommands()` — render-queue drain plus `ImmediateFlush(FlushRHIThreadFlushResources)`, which flushes the RHI thread and its pending resource deletions — at the last point before the pixels are requested, reported as `viewport.readbackFlushed`. **What the flush does and does not prove: it makes the source RT and its descriptors settled before the copy is set up, which is a plausible mitigation for an `AddressTranslationError` at a readback. It is NOT proven. The fault was seen once, on one scene, nothing in this tree reproduces it, and no test can assert the absence of a GPU fault. The live tester must verify on the reporter's scene — PIE with tracers at `TracerEvery = 1`, `SetGlobalTimeDilation(0.03)`, camera freshly moved to a new vantage — and a clean run there is evidence, not proof.** Docs corrected in `Docs/error-code-catalog.md`, `Docs/wiki-src/render.md`, `Docs/wiki-src/editor.md`, `Docs/wiki-src/ui.md`. Tests: `PinWright.render.capture_readiness.PendingShaderWorkRefusesTheReadback` (now also pins `PumpRounds == 0` on an asset-only queue — counterfactual: widen the loop back to any-pending-work and it fails, restoring the 20 s stall on every capture), `.BlockIsPublishedAndAbsentWhenUnmeasured`, `.EveryVerbGatesThenFlushesThenReadsBack` (renamed from `BothVerbsGateBeforeTheReadback`; searches the call name rather than `DrainBeforeReadback()` with empty parens, asserts gate < flush < readback and that the level paths gate from `BeforeFinalFrame` — counterfactual: hoist either gate back to handler entry and it fails), and the new `.SharedReadbackPreambleFlushesBeforeThePixels` (counterfactual: delete the `FlushBeforeReadback` call from `CaptureGameViewportToPngFile` and the ordering assertion against both readback branches fails). Ticket stays IN-REVIEW: the frame-quality half is testable and tested; the crash half is unproven by construction and needs the live tester.
