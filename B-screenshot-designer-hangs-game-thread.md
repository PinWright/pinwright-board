---
id: B-screenshot-designer-hangs-game-thread
title: "widget.screenshot_designer wedged the game thread for 10+ min on a C++-parented Widget Blueprint; the editor had to be killed and unsaved edits were lost"
status: OPEN
severity: Critical
category: bug
tags: [widget, screenshot-designer, umg, hang, game-thread, data-loss]
encounters: 1
costly: 1
lastSeen: 2026-09-24T01:25:00Z
---

# widget.screenshot_designer wedged the game thread on a C++-parented Widget Blueprint

UE 5.8, host `X:\src\unreal\unreal-fpv-new`, plugin `8748c637`, editor started with
`-RunningUnattendedScript`, no PIE. Sequence on `/App/App/UI/LobbyAndMenu/Elements/W_AppUserPanel`
(parent class is project C++ `UAppUserPanel`, a `UCommonUserWidget`):

1. `widget.add` x2 and `widget.set` x2 (new SizeBox + `W_ImageButton` child), then
   `blueprint.compile` (UpToDate, 0 errors) at 01:24:36Z. Package dirty, not saved.
2. `widget.screenshot_designer {widgetPath, filename, max_size: 1400, visibilityOverrides:
   {SB_SchoolLogin: Visible, LessonBlock: SelfHitTestInvisible, SB_Login: Collapsed,
   VB_Buttons: Collapsed}}`.

The call returned `stream read failed: timed out`. From then on the process was `Not Responding`
with about one core busy (CPU time 643 s to 1096 s over ~8 min), the frame counter stopped, and
nothing further was logged: not even `LogAssetEditorSubsystem: Opening Asset editor`, so the
hang sits before or inside the designer open. The gateway's I/O thread answered
`EDITOR_NOT_READY: game thread has not completed a tick for 128 s ... No PinWright RPC is in flight`,
which is wrong: the in-flight RPC was `widget.screenshot_designer`. The editor was killed by PID
after ~10 min; the unsaved widget edits were lost and had to be redone. Log preserved at
`Saved/Logs/PDS-hang-screenshot_designer-44296.log` in the host.

Control: after a restart, `editor.open_asset` on the same saved asset (and on
`W_LyraFrontEnd`, which contains it) opened the real UMG designer and the editor stayed responsive,
so a person opening the asset does not hit this; the hang is specific to the capture path.
Not the same as `B-geometry-offscreen-runs-native-construct` (that one asserts in
`NativeConstruct` via `resolve_geometry`; here there is no assert and the designer preview does not
run `NativeConstruct`), nor `B-screenshot-designer-leaves-designer-open-compile-crash`.

**Workaround:** do not use designer captures on this asset family; verify layout in PIE with
`editor.screenshot`. Save before any capture.
**Fix (proposed):** bound the designer-open/preview-resolve retry in
`ResolveDesignerPreviewTargetWithRetry` with a wall-clock limit that returns an error, and have
the stall reporter name the in-flight method.

## History
- `#1-hang-on-capture` `OPEN` reporter — Filed from the school-login UMG pass on
  `W_AppUserPanel`; costly (editor killed, edits redone, ~15 min lost).
- `#2-no-repro-not-shared-cause` `OPEN` developer — Investigated, not fixed; status stays OPEN. (1) **Not the same root cause as `B-geometry-offscreen-runs-native-construct`.** The Designer preview is built with designer flags (`FWidgetBlueprintEditor::UpdatePreview` -> `CreateUserWidgetFromBlueprint`), and the engine skips `NativeOnInitialized` / `NativeConstruct` for design-time widgets (`UserWidget.cpp:173`, `:1228`), so no C++ parent's runtime lifecycle runs on the capture path; the offscreen fix (`#2` there) does not touch this verb. (2) **No repro on the Linux host (UE 5.8, same asset family, same plugin code as `8748c637` in `Handlers/UI/*` and `Dispatch/*`)**, three attempts, each returned in seconds with a 1400x787 PNG and `visibilityOverrideCount:4`: the saved asset with the exact `#1` `visibilityOverrides` and `max_size:1400`; the same after `widget.add` x2 (SizeBox + a C++-button WBP child) and `blueprint.compile` with the package dirty and unsaved; and that dirty sequence again on an editor started with `-RunningUnattendedScript`. (3) **The `#1` evidence points away from the capture body.** `widget.screenshot_designer` is in the tick-unsafe table (`Dispatch/SafePoint.cpp:107`), so every run logs `LogPinWrightSafePoint: Running 'widget.screenshot_designer' (id=...) inline` before the handler starts, and `NoteRpcDispatchBegin` publishes the method before that; a wedge inside the handler (including inside `ResolveDesignerPreviewTargetWithRetry`) would have made the stall reporter name it. It said `No PinWright RPC is in flight`, and `Opening Asset editor` was never logged, so either the request was still queued for its safe point (game thread wedged elsewhere) or the handler had already returned. The retry loop the proposed fix targets is already bounded at 12 attempts; a wall-clock check between attempts cannot interrupt a single call that never returns, so that fix was not applied. **Needed to proceed:** from the preserved `Saved/Logs/PDS-hang-screenshot_designer-44296.log` on the Windows host, whether the `Running 'widget.screenshot_designer' ... inline` line (and any `Deferring 'widget.screenshot_designer'` Verbose line) is present, and the last 50 lines before the stall; or a native stack of the game thread if it recurs (`procdump -ma <pid>` on Windows before killing).
- `#3-reporter-names-queued-request` `OPEN` developer — Root cause still not pinned; status stays OPEN. (1) **Every loop the capture path owns is already bounded by count** (`FindWidgetEditorHostWindowWithRetry` and `ResolveDesignerPreviewTargetWithRetry`, 12 attempts each; `ResolveDesignerViewWidget` parent walk, depth 32; `PopulateSlateGeometryMap` walks a finite arranged-children tree). There is no plugin-side spin or unbounded wait. Anything left that could freeze is a single engine call on the handler stack (`OpenEditorForAsset`, `FSlateApplication::Tick`, `ForceRedrawWindow`, `FSlateRenderer::FlushCommands`, the `FWidgetRenderer` draw and readback). A wall-clock check between attempts cannot interrupt any of those, and the `#1` evidence says no handler was on the stack anyway, so no clock guard was added: it could not prove the freeze away. (2) **The concrete defect fixed is the misleading stall report.** When the game thread was wedged with no handler running, `ping` said "No PinWright RPC is in flight ... engine-internal" while `widget.screenshot_designer` sat queued behind the wedge. Socket-originated tools/call requests are marshalled with `AsyncTask` and normally deferred to the safe point at Verbose, so a request that never started leaves no Log line. Worse, the proxy's stream read timeout disconnects the caller, which drops its completion, so the transport had nothing left to name. `FSocketHttpServer` now keeps an `AwaitingRequests` record (method plus handoff time) from the handoff to the dispatcher until `ResolveCompletion`, independent of the completion and cleared only by the answer or server stop. When no handler is in flight, the stall report names the oldest record as `awaitingMethod` / `awaitingRequestId` / `awaitingSeconds`, and the message says the request is queued and the thread is not inside its handler (`Transport/SocketHttpServer.{h,cpp}`, `Transport/McpRequestCore.{h,cpp}`, docs `wiki-src/unattended.md`, `error-code-catalog.md`). Tests: `PinWright.transport.liveness.Ping.StalledNamesAwaitingRequest` (unit) and `PinWright.transport.liveness.Awaiting.SurvivesClientDisconnect` (end to end through a real socket server: hand-off, caller disconnects, stale heartbeat, a fresh ping names the request). (3) **Scenario pinned as a test:** `PinWright.widget.screenshot_designer.NativeParentDirtyWithVisibilityOverrides` is a WBP parented to the native `UTestWidgetConstructProbe`, with a SizeBox and child, compiled, dirty and unsaved, Designer closed, captured with four `visibilityOverrides` and `max_size:1400`. It must return success at 1400 on the long axis, report 4 overrides and zero `NativeConstruct` calls. It passes on the unfixed Linux tree, as `#2` predicts. It exists so a Windows suite run of the same shape fails or stops on it by name. **Needed to proceed:** the next recurrence's `ping` body (`awaitingMethod` present means the wedge is outside the handler; `inFlightMethod: widget.screenshot_designer` means it is inside), plus a native game-thread stack (`procdump -ma <pid>` before killing), and/or the `Running 'widget.screenshot_designer' ... inline` check against `Saved/Logs/PDS-hang-screenshot_designer-44296.log` that `#2` asked for.
