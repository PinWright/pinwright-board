---
id: B-capture-asset-preview-no-safe-close-mode
title: "`render.capture_asset_preview` has no safe `closeAfterCapture` value: true crashes the editor inside the shared `CaptureSubject::CloseAssetEditor()` teardown for BOTH the mesh and Niagara providers, false leaks every asset editor it opens"
status: OPEN
severity: Critical
category: bug
tags: [render, capture_asset_preview, capture-subject, crash, access-violation, editor-teardown, asset-editor, resource-leak, multi-agent]
encounters: 1
lastSeen: 2026-08-27T18:47:15+05:00
---

# `render.capture_asset_preview` has no safe configuration — `closeAfterCapture: true` faults in the shared teardown, `closeAfterCapture: false` leaks asset editors without bound

`render.capture_asset_preview` works by opening a real asset editor, capturing its
preview viewport, and then either closing that editor or leaving it open. Both
outcomes are unsafe, so the verb has **no configuration a caller can pick that is
known good**:

- **`closeAfterCapture: true`** (the default) — `EXCEPTION_ACCESS_VIOLATION`
  inside `PinWrightCaptureSubject::CloseAssetEditor()`. The whole editor process
  dies. Observed twice in one session through **two different providers**, which
  is what proves the fault is in the shared teardown rather than in any one
  subject kind.
- **`closeAfterCapture: false`** — nothing ever closes the editors this verb
  opens, and there is no cap on how many may be open. Measured **37**
  `LogAssetEditorSubsystem: Opening Asset editor for ...` lines in one session's
  log against **0** closes.

The plugin's own wiki already warns that an asset editor left open crashes UE 5.8
at shutdown, so all three states — close, leave open, leave open until exit — are
documented-or-observed unsafe.

This is a shared-editor verb: when it takes the process down it takes every other
agent's unsaved work with it. On the session that produced this ticket, four
editor kills cost roughly forty minutes and blocked eight agents each time.

## Root cause (guilty source line)

Both crashes pass through the same function.
`Plugins/PinWright/Source/PinWright/Private/Handlers/Render/CaptureSubject.cpp:1405`:

```cpp
    AssetEditorSubsystem->CloseAllEditorsForAsset(Asset);
    // Read back rather than assumed: an asset editor can refuse its own close, and a close reported
    // but not performed leaves the shutdown-crash precondition in place while the response says it
    // is gone.
    return AssetEditorSubsystem->FindEditorForAsset(Asset, false) == nullptr;
```

(The crash frames report `CaptureSubject.cpp:1409`, the `return` on the next
statement — the compiler's line attribution for the frame. `CloseAllEditorsForAsset`
at `:1405` is the call that faults.)

**The existing pre-close gate does not cover this.** `CloseAssetEditor`
(`CaptureSubject.cpp:1355`) already carries an elaborate guard at
`CaptureSubject.cpp:1392-1403` — `CountPreviewSceneViewportHolders(Asset)` — which
refuses the close when any other live `TSharedPtr<FSceneViewport>` exists,
because `SEditorViewport`'s destructor asserts `check(SceneViewport.IsUnique())`.
That guard is for a **different** crash (a `check()` on a shared viewport
pointer). The crashes here are `EXCEPTION_ACCESS_VIOLATION`s inside the
**toolkit's own destructor chain**, downstream of `CloseAllEditorsForAsset`,
which that guard does not and cannot see. A fixer reading `CloseAssetEditor` will
find a function that looks carefully defended and is not defended against this.

## Evidence: two crashes, two providers, one shared frame

Both callstacks are in this checkout's own logs. Verified on disk 2026-08-27; the
plugin frames carry this tree's absolute paths.

**Crash A — Niagara provider.** `Saved/Logs/EAContentExamples58-backup-2026.08.27-08.26.21.log:6048`,
`EXCEPTION_ACCESS_VIOLATION reading address 0x0000000000000000`, on
`/Game/Atlantis/VFX/NS_Bubbles_Stream`. Frames `:6094-6114`, innermost first:

```
UNiagaraStackPropertyRow::FinalizeInternal()   NiagaraStackPropertyRow.cpp:209
UNiagaraStackEntry::Finalize()                 NiagaraStackEntry.cpp:227
UNiagaraStackEntry::Finalize()                 NiagaraStackEntry.cpp:233  (x5, recursive)
UNiagaraStackViewModel::Reset()                NiagaraStackViewModel.cpp:208
FNiagaraEmitterHandleViewModel::Cleanup()      NiagaraEmitterHandleViewModel.cpp:53
FNiagaraSystemViewModel::Cleanup()             NiagaraSystemViewModel.cpp:283
FNiagaraSystemToolkit::~FNiagaraSystemToolkit() NiagaraSystemToolkit.cpp:101
FAssetEditorToolkit::CloseWindow()             AssetEditorToolkit.cpp:538
UAssetEditorSubsystem::CloseAllEditorsForAsset() AssetEditorSubsystem.cpp:311
PinWrightCaptureSubject::CloseAssetEditor()    CaptureSubject.cpp:1409
PinWrightCaptureSubjectNiagara::Release()      CaptureSubjectProviders_Niagara.cpp:197
PinWrightCaptureSubject::ReleaseSubject()      CaptureSubject.cpp:253
AutoHandler_349_()                             RenderHandler.cpp:1445
FRpcDispatcher::ProcessRequest()               RpcDispatcher.cpp:646
```

**Crash B — Mesh provider.** `Saved/Logs/EAContentExamples58-backup-2026.08.27-08.39.32.log:3512`,
`EXCEPTION_ACCESS_VIOLATION reading address 0x000000003f800000`. Frames `:3528-3545`:

```
TBaseRawMethodDelegateInstance<...FAssetEditorToolkit...>::ExecuteIfSafe()
                                               DelegateInstancesImpl.h:498
SStandaloneAssetEditorToolkitHost::ShutdownToolkitHost()
                                               SStandaloneAssetEditorToolkitHost.cpp:409
SStandaloneAssetEditorToolkitHost::OnToolkitHostingFinished()
                                               SStandaloneAssetEditorToolkitHost.cpp:430
FAssetEditorToolkit::CloseWindow()             AssetEditorToolkit.cpp:538
UAssetEditorSubsystem::CloseAllEditorsForAsset() AssetEditorSubsystem.cpp:311
PinWrightCaptureSubject::CloseAssetEditor()    CaptureSubject.cpp:1409
PinWrightCaptureSubjectMesh::ReleaseMeshSubject() CaptureSubjectProviders_Mesh.cpp:636
PinWrightCaptureSubject::ReleaseSubject()      CaptureSubject.cpp:253
AutoHandler_349_()                             RenderHandler.cpp:1445
FRpcDispatcher::ProcessRequest()               RpcDispatcher.cpp:646
```

**What the pair proves.** Every frame from `CaptureSubject.cpp:1409` outward is
identical. Only the provider that calls it differs —
`CaptureSubjectProviders_Niagara.cpp:197` versus
`CaptureSubjectProviders_Mesh.cpp:636`, both verified at those exact lines in
this tree, both calling `PinWrightCaptureSubject::CloseAssetEditor(...)`. So
**the verb is unsafe on every subject kind it serves, not just Niagara.** Treat
any capture of a static or skeletal mesh preview as carrying the same risk.

Crash B's fault address is worth noting for whoever debugs it: `0x3f800000` is
the IEEE-754 bit pattern of `1.0f` being dereferenced as a pointer — the
signature of a freed-and-reused allocation or a type confusion, not a null
dereference.

## The other half: `closeAfterCapture: false` leaks editors without bound

Taking the documented workaround leaves every asset editor open. Measured on
`Saved/Logs/EAContentExamples58-backup-2026.08.27-09.49.42.log` in this tree:

```
grep -a -c "LogAssetEditorSubsystem: Opening Asset editor"  ->  37
grep -a -ci "clos.*editor"                                  ->   0
```

Thirty-seven asset editors opened across one session (`SM_Fish_A`, `SM_Fish_B`,
`SM_Column_Doric`, `SM_Coral_Brain`, `SM_Coral_Fan`, `SM_Coral_Tube`, …), none
closed. Nothing in the plugin closes an editor opened with
`closeAfterCapture: false`, and there is no cap on how many may be open at once.

**Correcting the session log on this point.** The defect log entry that recorded
this (D30) attributed the 09.49.42 crash to Slate prepass recursion / stack
exhaustion over the accumulated widget tree. **That diagnosis is wrong, and this
tree's own log disproves it**: that crash is `EXCEPTION_ACCESS_VIOLATION` at
`0x00000008000000b8` (not `EXCEPTION_STACK_OVERFLOW`), its callstack is 120
frames total with only 36 `SWidget::Prepass_Internal` frames (not "hundreds"),
and its innermost frame is `UNiagaraEmitter::GetEmitterData()` on a freed emitter
handle. That crash belongs to `B-niagara-edit-with-open-asset-editor-slate-crash`
(filed alongside this one), not here. **The leak is real and measured; it has no
crash of its own attributed to it yet.** Do not fix it on the strength of a
crash it did not cause — fix it because 37 unclosed editors is a leak, and
because the plugin's wiki already says an open asset editor faults UE 5.8 at
shutdown.

## Verbatim repro

```
render.capture_asset_preview {subject: {kind: "niagara",
                                        path: "/Game/Atlantis/VFX/NS_Bubbles_Stream"}}
```

`closeAfterCapture` defaults to `true` (`RenderHandler.cpp:332`). Survives one or
two calls; dies on a later one. Reproduced twice in one session on that asset,
and once through the mesh provider on the static meshes above.

A capture that returns clean is not proof the editor survived: a shot set that
had **already returned `assetEditorClosed: true`** was followed by the process
dying roughly 90 seconds later on unrelated work.

## What it should do

Three changes, and **none of them alone makes the verb safe** — see the section
below.

1. **Make the close survivable.** The fault is downstream of
   `CloseAllEditorsForAsset` in the toolkit destructor chain, so the fix is
   either to drop whatever the provider still holds into the toolkit's object
   graph before closing (the way the existing `CountPreviewSceneViewportHolders`
   gate does for `FSceneViewport`), or to defer the close out of the release path
   onto a later tick when no provider state is live. Extending the existing gate
   to count toolkit-owned view-model references is the natural first probe.
2. **Bound the open editors.** A pool of one — close the *previous* subject's
   editor when opening the next — fixes the leak without ever tearing down the
   editor whose rows are still live, and would sidestep (1) for the common
   sequential-capture case.
3. **Stop the verb being the only route.** See `## Related` — the level route
   (`niagara.spawn_actor` / `actor.spawn` + `render.capture_open_level`) opens no
   asset editor at all and is what this session ended up using. It should be
   named in the docs as the safe path for repeated captures.

## Do not fix one of these and stop

This defect and its two siblings are **one fix**, and each one's documented
workaround is another one's trigger:

- **This ticket** — `closeAfterCapture: true` faults in `CloseAssetEditor`. Its
  workaround is `closeAfterCapture: false`.
- **`B-niagara-edit-with-open-asset-editor-slate-crash`** — with an editor left
  open by `closeAfterCapture: false`, any structural `niagara.*` edit to that
  system kills the process at the next Slate redraw, on a stale emitter handle.
  So this ticket's workaround walks you into that one.
- **The leak documented above** — `closeAfterCapture: false` also accumulates
  editors without bound, which is what puts an editor in the state the sibling
  ticket faults on in the first place.

A fix that only makes the close safe leaves the leak and the stale-widget crash
live for anyone who passes `false`. A fix that only caps open editors leaves the
teardown fault live for everyone who does not. **Verify a fix by running both
paths, not one.**

## Related

- `B-niagara-edit-with-open-asset-editor-slate-crash` — the third member of this
  family; must be fixed with this one.
- `B-editor-quit-crash-open-asset-editors` (IN-REVIEW, High) — the adjacent
  known crash class (asset-editor toolkit destruction as an AV site). Its `#4`
  declares the crash "fixed and runtime-proven on UE 5.8" — but that proof covers
  `CloseOpenAssetEditors()` on the **`editor.quit`** path only. The same
  subsystem close called from `CaptureSubject::CloseAssetEditor()` still faults
  on UE 5.8, and for a Niagara toolkit rather than a static-mesh one. That
  ticket's fix does not reach this call site.
- `F-editor-close-all-asset-editors` (OPEN, **Low**) — asks for a verb to
  enumerate and bulk-close open asset editors, and its `#2` **demoted itself to
  Low** on the reasoning that it was "no longer justified by the quit path". The
  37-open-editors measurement above re-justifies it from a completely different
  path: unbounded accumulation during ordinary capture work inside a live
  session. Its Low rating rests on a premise this evidence falsifies.
- `F-render-runtime-spawned-actor` (OPEN, Medium) describes this verb as
  "**Static Mesh asset editors only**. Anything else is rejected with a typed
  error". **That reading is stale.** In this tree the verb registers at
  `RenderHandler.cpp:293` advertising "Static Mesh, Skeletal Mesh, animation
  asset and Niagara system", takes a `subject: {kind, path, animation,
  closeAfterCapture}` object at `RenderHandler.cpp:301-308`, and ran a Niagara
  provider to completion in the crash above. Every board ticket that models this
  verb predates the capture-subject provider architecture.

severity rationale: impact=editor crash taking down every agent sharing the process, with silent loss of all unsaved work, and no argument value that avoids it x reach=the documented, wiki-recommended way to prove any mesh or Niagara asset -> Critical

## History
- `#1-initial-repro` `OPEN` reporter — Found building the Atlantis level (map as forcing function; see host `CLAUDE.md` § "What this project is for"), 2026-08-27, UE 5.8, PinWright at this checkout's HEAD. **Four editor kills in one session.** Two of them are this ticket's teardown fault, and the pair is what proves it is not Niagara-specific: crash A at `2026.08.27-08.26.21` UTC (`EXCEPTION_ACCESS_VIOLATION` reading `0x0`, backup log `:6048`, frames `:6094-6114`) entered through `CaptureSubjectProviders_Niagara.cpp:197`, and crash B at `2026.08.27-08.39.32` UTC (reading `0x3f800000`, backup log `:3512`, frames `:3528-3545`) entered through `CaptureSubjectProviders_Mesh.cpp:636` — every frame from `CaptureSubject.cpp:1409` outward identical, only the provider differing. Both logs and both callstacks re-verified on disk in THIS checkout during filing, with the plugin frames carrying this tree's absolute paths; both provider call sites confirmed at those exact line numbers in current source. Guilty call is `AssetEditorSubsystem->CloseAllEditorsForAsset(Asset)` at `CaptureSubject.cpp:1405` (frames report `:1409`, the following `return`). Recorded that the existing pre-close gate at `CaptureSubject.cpp:1392-1403` (`CountPreviewSceneViewportHolders`) defends a *different* crash — the `SEditorViewport` destructor's `check(SceneViewport.IsUnique())` — and cannot see a toolkit-destructor AV, so the function looks defended and is not. The `closeAfterCapture:false` half re-measured in this tree: 37 `LogAssetEditorSubsystem: Opening Asset editor` lines against 0 closes in the 09.49.42 backup log. **Corrected the session log's D30 diagnosis**: it attributed the 09.49.42 crash to Slate prepass stack exhaustion, but that crash is an `EXCEPTION_ACCESS_VIOLATION` at `0x00000008000000b8` with 120 total frames and 36 `Prepass_Internal` frames, innermost `UNiagaraEmitter::GetEmitterData()` — it belongs to the sibling ticket. The leak stands on its own measurement; no crash is attributed to it. Also corrected the log's original Niagara-only framing, and noted that a capture returning `assetEditorClosed:true` does not mean the editor survived the following minute. Worked around by proving assets from the LEVEL instead (`niagara.spawn_actor` / `actor.spawn` + `render.capture_open_level`, then delete the preview actor), which opens no asset editor at all; defect untouched.
