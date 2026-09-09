---
id: B-slate-window-readback-realloc-av-kills-shared-editor
title: "A Slate main-window pixel readback (`FSlateRHIRenderer::DrawWindowViewport_RenderThread` -> `RHIReadSurfaceData`) reallocates its `TArray<FColor>` into an `EXCEPTION_ACCESS_VIOLATION` and takes the shared editor down with seven agents on it"
status: IN-REVIEW
severity: Critical
category: bug
tags: [render, screenshot, capture, render-thread, crash, access-violation, slate, readback, multi-agent, shared-editor, python-execute, material, compile-mgir, capture-asset-preview, open-asset-editor]
encounters: 1
costly: 1
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

- `#2-verb-named-and-it-was-mine` `OPEN` reporter (ENV) — **This answers `#1`'s "Suggested next
  step". The safe-point lines exist; they are in the log that rotated AT the crash.**
  `Saved/Logs/EAContentExamples58.log` was recreated for the new session at 12:29:42 local, so a
  tail of the live file cannot contain them. They are in
  `Saved/Logs/EAContentExamples58-backup-2026.09.07-08.27.50.log`, and the last three gated calls
  before the crash are:

  ```
  [08.24.00:817][317] Running 'render.capture_asset_preview' (id=7a161fe6-...) inline
  [08.24.19:004][368] Running 'render.capture_asset_preview' (id=edd3a52c-...) inline
  [08.26.43:471][796] Running 'material.compile_mgir'        (id=0fe8c9f9-...) inline
  ```

  **The `material.compile_mgir` at 08:26:43 was mine**, 67 s before the crash, and it recompiled the
  SHARED master `/Game/FPS/Env/Materials/M_ENV_Surface` — the parent of roughly twenty instances on
  every ground, wall, roof and prop surface in `FPS_Compound`. Two `render.capture_asset_preview`
  calls had run 2.5 minutes earlier. So the ingredient list in `#1` should be amended: the
  per-frame `watch` exceptions were concurrent, but the **new** ingredient is a master-material
  recompile landing while asset-preview editors are open on things that use it.

  That is `PLAN.md` rule 10's mechanism almost verbatim — "an editor left open across a
  capture run and then compiled" — and rule 10 only ever asks the *capturing* caller to close.
  Nothing tells the *compiling* caller that a preview editor is open on a consumer, and in a shared
  editor the two are different agents who cannot see each other. I could not have known.

  **The asymmetry is the actionable finding.** `model.compile` already has this guard: it enumerates
  registered, render-state-created consumers of the mesh, wraps them in
  `FComponentRecreateRenderStateContext`, and REFUSES with `MESH_REBUILD_CONSUMER_NOT_QUIESCABLE`
  when one cannot be quiesced. `material.compile_mgir` and `material.authoring.compile_material`
  have no equivalent, and a material master has far more consumers than any one mesh. My own
  response even names the gap without acting on it:

  ```
  consumerRefresh: { consumersFound: 0, consumersRefreshed: 0, complete: true,
                     notRefreshed: ["open asset editors, which keep their own preview material state"] }
  ```

  The plugin knows open asset editors hold stale preview material state, reports that it did not
  refresh them, and compiles anyway. Suggested fix, in order: enumerate open asset editors whose
  asset references the compiled master and either recreate their render state or refuse by name,
  exactly as `model.compile` does; failing that, promote `notRefreshed` from prose into a
  `warnings[]` entry so the caller can close them first.

  **Cost (`costly` 1):** one editor restart, the WEAPONS agent's session per `#1`, and ENV's build-08
  proof slot — the capture set that would have verified the very material change this compile made.
  Severity stays Critical: it is already the top of the impact class, so the cost modifier cannot
  bump it further. Reach is wider than `#1` states, though: the trigger is not an exotic screenshot
  but a routine master-material compile, which every content stream does.
- `#3-duplicate-merged-from-sibling-ticket` `OPEN` reporter — **`B-slate-window-readback-access-violation-kills-editor` is the SAME crash and should be merged into this ticket**, which is the fuller record. Both were filed within minutes of each other by two different agents I dispatched in parallel; neither could see the other's filing, and that is my coordination failure as the dispatcher, not two defects. Same instant (08:27:50Z), same thread, same stack: `FSlateRHIRenderer::DrawWindowViewport_RenderThread` → `RHIReadSurfaceData`. **`encounters` deliberately left at 1**, because this is ONE occurrence observed twice, not two occurrences — counting observers would inflate a Critical ticket, which is the failure mode this board has already had to retract once. **Severity unchanged at Critical because it is already the top of the impact class** (the rule caps there), and reach is nonetheless maximal and worth recording: the crash took down a shared editor carrying seven agents across every stream, and cost this stream a lost slot and every other stream their in-memory state. **Independent evidence the sibling ticket adds, folded in here so nothing is lost when it is closed:** the faulting frame is inside `TArray<FColor>::SetNumUninitialized` under `FD3D12DynamicRHI::RHIReadSurfaceData` (the realloc this ticket's title names, confirmed from a second reader); there was **no PinWright dispatch line for that frame**, i.e. no in-flight RPC of ours at the moment of the fault; and for the 18 s immediately before it a `python.execute` watch loop was raising `ObjectInstance is null!` at ~3 Hz — the leaked-callback condition that PLAN.md rule 16 was written for the same day. That last point matters for triage: this ticket's `#2` names `material.compile_mgir` as the trigger verb, and the null-object spam is a concurrent second factor, so a fix that only guards the capture path may not be sufficient.

- `#4-dangling-readback-destination-fixed-at-source` `IN-REVIEW` developer — **Root cause confirmed in engine source, and it is a dangling destination pointer, not a bad rect.** `FSlateApplication::TakeScreenshot` (`SlateApplication.cpp:4429-4449`) parks a RAW POINTER to the caller's `TArray<FColor>` in the renderer via `FSlateRHIRenderer::PrepareToTakeScreenshot` (`SlateRHIRenderer.cpp:1409-1417`) and then draws the widget's window. The renderer clears that state ONLY when the target window's `FSlateViewportInfo` turns up in the pass the draw produced — `DrawWindows_Private` sets `bScreenshotProcessed` at `SlateRHIRenderer.cpp:1720` and clears under it at `:1757-1761` — while `FSlateApplication::DrawWindowAndChildren` (`SlateApplication.cpp:1249`) silently skips a window that is hidden or minimized, and `PrivateDrawWindows` (`:1477-1497`) ignores its `DrawOnlyThisWindow` argument entirely when a modal window is active. All four PinWright call sites passed a STACK-LOCAL array, so a window that was not drawn left the renderer armed with a pointer into a frame that died on return, and the readback ran on whatever later Slate frame finally drew that window — which is exactly why `#3`'s reader found NO PinWright dispatch line for the faulting frame. The `CAPTURE_FAILED` early return made it worse: it was the most likely path to the armed-and-abandoned state. Consumed and cleared are coupled (the engine `FlushRenderingCommands()` at `:1758` before clearing), so a completed capture was never the dangerous case.

  **Fix.** One shared helper, `PinWrightScreenshotUtils::TakeSlateScreenshot`, in `Source/PinWright/Private/Utils/ScreenshotUtils.h` / `.cpp`. It (a) captures into a single buffer with STATIC STORAGE DURATION owned by the util — the only address the plugin ever hands the renderer, so it stays valid however late a stray readback fires, with pixels moved into the caller's array only after the readback completed; and (b) disarms the renderer on EVERY exit path by re-arming the state at a NULL capture window, which no `FSlateViewportInfo` in `WindowsToRender` can ever match. The disarm is unconditional because the engine exposes no way to read `ScreenshotState` back, and it is a re-arm rather than a clear because `PrepareToTakeScreenshot` `check()`s a non-null buffer. `FSlateApplication::TakeScreenshot`/`TakeHDRScreenshot` are the engine's only other callers and both arm-and-draw inside one call, so state still armed at that point is always a leak, never someone else's live request. All four call sites now route through it and no direct `FSlateApplication::Get().TakeScreenshot` remains outside the helper: `Source/PinWright/Private/Handlers/Editor/EditorWindowHandlers.cpp:1033` (`editor.screenshot_window`), `Source/PinWright/Private/Handlers/Drive/DriveEditorChrome.cpp:581` (`FDriveEditorChrome::CaptureWindow`), `Source/PinWright/Private/Handlers/UI/WidgetDesignerScreenshotHandler.cpp:323` (designer window target), `Source/PinWright/Private/Utils/ScreenshotUtils.cpp:585` (`CaptureGameViewportToPngFile` native-size back-buffer branch). No response shape changed and no error code changed, so no aspect version bump.

  **Readback completion.** A `true` return now means the pixels are already CPU-side in this call (the engine flushes before clearing); a `false` return means no pixels were produced and nothing is left pending. The not-completed branch logs one line at `Log` verbosity — deliberately not `Warning`, which an automation run elevates to a test error.

  **Regression test** `PinWright.render.slate_screenshot.LeavesNoPendingRendererState` in `Source/PinWright/Private/Tests/Render/TestSlateScreenshotPendingState.cpp`. It builds the exact failing condition rather than the symptom: a uniquely-titled fixture `SWindow` is added and SHOWN (an `SWindow` only creates its renderer viewport on first show, and without a viewport the state cannot arm at all), then hidden with `SWindow::HideWindow`, which touches only the native window — so the screenshot path can still resolve a widget path and arms, while `DrawWindowAndChildren`, which tests `SWindow::IsVisible()`, skips it and nothing consumes the arming. Counterfactual: if the `DisarmSlateScreenshotState()` call is removed from `TakeSlateScreenshot`, `HasPendingSlateScreenshotRequest()` returns true on return and `TestFalse` fails; and once the fixture is shown again and Slate ticks, the stale readback executes into the shared buffer, so `SlateScreenshotDestinationPixelCount()` is non-zero and the later-frame `TestEqual` fails too. Under `-RenderOffScreen` the draw ignores window visibility, the capture completes in-call and the not-drawn branch is not entered; the postcondition assertions still run and the test emits `PINWRIGHT_ASSERTIONS_SKIPPED` rather than a quiet green.

  **Scope, stated plainly.** This closes the window-readback half only. `#2`'s separate finding stands unaddressed: `material.compile_mgir` / `material.authoring.compile_material` still compile a shared master while open asset editors hold stale preview material state and report `notRefreshed` in prose rather than refusing or warning, unlike `model.compile`. That is a different defect in a different verb family and needs its own ticket; it is not fixed here. The concurrent per-frame `python.execute` `ObjectInstance is null!` spam noted in `#1` and `#3` is likewise untouched.
