---
id: B-screenshot-designer-leaves-designer-open-compile-crash
title: "widget.screenshot_designer leaves the UMG Designer open, and a later blueprint.compile_bpir on that widget kills the editor in SDesignerView::UpdatePreviewWidget -> UUserWidget::RebuildWidget"
status: OPEN
severity: Critical
category: bug
tags: [widget, screenshot_designer, compile_bpir, designer, crash, access-violation, editor-open-guard, close_asset, multi-agent]
encounters: 3
lastSeen: 2026-09-03T05:00:00Z
---

# A capture verb opens the Designer; the next BPIR compile on that widget crashes the editor

## Symptom

`widget.screenshot_designer` opens the target Widget Blueprint in the UMG Designer as a
documented side effect (its own page: *"Cold Designer state opens the asset, switches to Designer,
invokes `SlatePreview`…"*). It never closes it. The Designer tab then ticks
`SDesignerView::UpdatePreviewWidget` every frame for the rest of the session. A subsequent
`blueprint.compile_bpir` on that same widget rebuilds its generated class under the live preview,
and the editor dies on an ordinary Slate paint:

```
Unhandled Exception: EXCEPTION_ACCESS_VIOLATION reading address 0x0000000000000038
  UUserWidget::RebuildWidget()               UserWidget.cpp:1214
  UWidget::TakeWidget_Private()              Widget.cpp:993
  UWidget::TakeWidget()                      Widget.cpp:975
  SDesignerView::UpdatePreviewWidget()       SDesignerView.cpp:2305
  SDesignerView::Tick()                      SDesignerView.cpp:2401
  SWidget::Paint()                           SWidget.cpp:1511
  ... Slate paint stack ...
  SWindow::PaintSlowPath()                   SWindow.cpp:2120
```

The process ran `StaticShutdownAfterError` and exited; the gateway went to connection-refused,
taking every other agent in the shared editor with it. Log:
`Saved/Logs/EAContentExamples58.log` around `19.44.57` (crash), the preceding
`19.44.49 LogBlueprint: Compiling Blueprint` lines are a *different* stream's compiles landing on
the same frame.

## Sequence that produced it

1. `widget.screenshot_designer {widgetPath:"/Game/FPS/UI/WBP_HUD", target:"preview"}` — twice, for
   layout review. Both succeeded; the Designer stayed open (no `editor.close_asset` was called,
   and the verb does not do it).
2. Eight `blueprint.compile_bpir` calls against `/Game/FPS/UI/WBP_HUD`, each adding one function
   and each implicitly running a full Blueprint compile. Seven returned
   `compiled:true, status:"UpToDate"`.
3. The eighth returned `COMPILE_FAILED` (an unrelated FText-namespace error) and therefore ran the
   rollback path.
4. The editor crashed on the next paint, in the Designer preview rebuild.

I cannot say from the log whether the fatal frame belongs to the failed compile's rollback
specifically or to any of the recompiles — only that the crashing stack is the Designer preview
rebuilding a widget class PinWright had been recompiling under it.

## Why this is worse than the general "edit with the asset editor open" class

`B-niagara-edit-with-open-asset-editor-slate-crash` (DONE) established the pattern: a structural
edit while an asset editor is open dies on the next Slate redraw because the open editor holds
state the mutation invalidated, and the mutating verb has no `EDITOR_OPEN` guard. This is the UMG
instance of the same class, with one aggravating difference: **the open editor was opened by
PinWright itself, by a verb whose entire purpose is read-only review.** A caller doing
inspect → capture → edit — the loop `widget.iteration-loop` prescribes — walks into it by
following the documentation.

`B-widget-add-userwidget-preview-crash` (DONE) is a different trigger (`widget.add` of custom
`UUserWidget` rows) reaching a similar stack; nothing here used `widget.add`.

## What should happen

Any of these closes it, in rough order of preference:

1. `widget.screenshot_designer` closes the asset editor it opened, when it opened it (leave a
   pre-existing user-opened tab alone). The capture already restores every other piece of state it
   touches — designer-eye flags, runtime visibility — so leaving a live Designer tab behind is the
   one un-restored side effect.
2. `blueprint.compile_bpir` (and `blueprint.compile`) detects an open Designer for the target
   Widget Blueprint and either closes/refreshes it around the compile, or refuses with
   `EDITOR_OPEN` naming `editor.close_asset` — the guard `B-niagara-*` asked for, applied to UMG.
3. Failing both, the `widget.screenshot_designer` and `widget.iteration-loop` pages must say in
   plain terms: call `editor.close_asset` before any further authoring on a widget you captured.

**Workaround:** call `editor.close_asset` on the widget immediately after every
`widget.screenshot_designer`, before any `blueprint.compile_bpir` / `blueprint.compile` on it.

severity rationale: impact=editor crash that takes down a shared multi-agent editor and discards
every unsaved in-memory edit in it x reach=the documented inspect-capture-edit loop for UMG work
-> Critical.

## History
- `#1-filed` `OPEN` reporter — Hit on EAContentExamples58 (UE 5.8, shared editor, port 27145) building `/Game/FPS/UI/WBP_HUD`. Two `widget.screenshot_designer target:"preview"` calls for layout review left the WBP_HUD Designer open; eight subsequent `blueprint.compile_bpir` calls on the same widget authored its functions, the eighth returned `COMPILE_FAILED` on an unrelated FText-namespace error, and the editor then died with `EXCEPTION_ACCESS_VIOLATION reading address 0x38` at `UUserWidget::RebuildWidget` (UserWidget.cpp:1214) reached from `SDesignerView::UpdatePreviewWidget` (SDesignerView.cpp:2305) inside `SDesignerView::Tick` on a Slate paint — i.e. the Designer preview rebuilding the class that had just been recompiled beneath it. Evidence: `Saved/Logs/EAContentExamples58.log`, `Critical error` block at `19.44.57:839`, full callstack quoted above. Cost beyond the crash: the eight `compile_bpir` calls were never followed by a save (the verb reports no persistence field — filed separately as `E-compile-bpir-no-persistence-field`), so all eight authored functions were in memory only and are gone; `widget.export_xml`-visible tree and every `blueprint.add_variable` variable survived, because those verbs auto-save. Cross-refs: `B-niagara-edit-with-open-asset-editor-slate-crash` (DONE) — same class, different subsystem, and its requested `EDITOR_OPEN` guard was never extended to UMG; `B-widget-add-userwidget-preview-crash` (DONE) — similar stack, different trigger (`widget.add`, not used here). No plugin source read; diagnosis is from the callstack and the call sequence.
- `#2-blast-radius-second-agent` `OPEN` reporter — Same crash instance (19:44:57Z), witnessed from a second agent in the same shared editor authoring Niagara blood emitters under `/Game/FPS/VFX/Emitters/`. `encounters` deliberately NOT bumped: one occurrence, two witnesses. This entry records the blast radius the severity rationale predicts. Lost from that stream: three `asset.duplicate` calls that had each returned `success:true, pendingSave:true, existsOnDisk:false` (`E_Blood_Mist`, `E_Blood_Drips`, `E_Blood_Burst`) and one `niagara.set_static_switch` on each; nothing reached disk, confirmed by `Content/FPS/VFX/Emitters/` holding no `E_Blood_*` beyond the pre-existing `E_Blood_Spray.uasset`. The batch of switch calls issued immediately after the crash returned `EDITOR_NOT_RUNNING (connection refused)`. That stream never opened a widget, never called `widget.screenshot_designer` and never compiled a blueprint — it had no way to observe the hazard or defend against it. This argues for fix 1 or 2 over fix 3 in "What should happen": a documentation-only fix on the widget pages protects only the agent that reads them, while the editor is shared with agents working unrelated subsystems who will never see that page.
- `#3-third-witness-no-recovery-path` `OPEN` reporter — Same crash instance (19:44:57Z), third witness: a stream briefed to author four `E_Explosion_*` Niagara emitters (Flash, Fireball, Shockwave, Light) under `/Game/FPS/VFX/Emitters/`. Unlike #2 it lost no in-memory work, because it never landed a call — the crash preceded its first `asset.duplicate`, which returned `EDITOR_NOT_RUNNING (connection refused)`. The cost here is therefore a whole scheduled work unit that never started, which is a category the severity rationale does not yet count. New fact this entry adds: **the crash has no recovery path.** Five minutes after it, `Saved/PinWright/gateway-port` still reads `27145` with nothing listening, no `UnrealEditor.exe` for this project exists (the five live engine processes belong to `EAContentExamples57` and `unreal-fpv/PDS`), and two `CrashReportClientEditor` windows titled `EAContentExamples58 Crash Repor...` are parked unattended. Every asset-only brief in this wave forbids `editor_start`/`editor_restart` on the grounds that a shared-editor lifecycle decision is not an authoring agent's to make, so no participant could restart it themselves; recovery took an out-of-band intervention by the editor's owner and the gateway came back roughly ten minutes after the crash. So the outage is bounded by whoever owns the editor being available, not self-healing. `encounters` deliberately NOT bumped — one occurrence, now three witnesses. Consequence for prioritisation: the blast radius is not bounded by the in-flight edits #1 and #2 enumerate, it extends to every task queued behind the shared editor for as long as it stays down.
- `#4-fourth-witness-niagara-authoring` `OPEN` reporter — Fourth witness to the same 19:44:57 crash, from the Niagara asset-authoring workstream on the same shared editor. Loss here: five `UNiagaraEmitter` assets (`/Game/FPS/VFX/Emitters/E_FPS_MuzzlePistol_{Core,Petals,Smoke,Heat,Light}`) created by `asset.duplicate` seconds before the crash and never saved — `asset.duplicate` reports `existsOnDisk:false, pendingSave:true`, so a duplicate is in-memory only until an explicit `asset.save`. What survived did so by luck: an unrelated agent's save sweep at 19:41 had flushed the earlier emitters (`E_FPS_MuzzleAR_*`, `E_FPS_ShellCasing`, `E_FPS_ShellSmoke`, `E_FPS_TracerCore`) to disk. Reinforces the practical lesson rather than adding a new failure mode: on a shared editor, `asset.save` every asset the moment it is structurally complete, because the crash that discards it will be triggered by a subsystem you are not using. `encounters` deliberately NOT bumped — still one occurrence, now four witnesses.
- `#5-blast-radius-fifth-witness-zero-work-in-flight` `OPEN` reporter — Same crash instance (19:44:57Z), fifth witness: a Niagara agent tasked with authoring `E_Explosion_SmokeColumn`, `E_Explosion_Debris` and `E_Explosion_DustRing` under `/Game/FPS/VFX/Emitters/`. `encounters` deliberately NOT bumped: still one occurrence, now five witnesses. This entry records a blast-radius shape the earlier entries do not: **the crash landed before this stream issued its first RPC**, so nothing was lost but nothing could start either — the opening `asset.duplicate` of `/Niagara/DefaultAssets/Templates/Emitters/SimpleSpriteBurst` returned `EDITOR_NOT_RUNNING (connection refused)`, as did every retry. Confirmed on disk: `Content/FPS/VFX/Emitters/` holds no `E_Explosion_*` at all (26 pre-existing emitters from other streams, newest `E_ImpConcrete_Chunks.uasset` at 22:43:25 local, i.e. 92 s before the crash). Process evidence: no `UnrealEditor.exe` remains for this project (`Get-CimInstance Win32_Process` shows only an EAContentExamples57 automation run and four unrelated `PDS.uproject` game processes), nothing listens on the port named in `Saved/PinWright/gateway-port` (27145), and seven `CrashReportClientEditor` processes are queued. Reinforces the argument in `#2` for fix 1 or 2 over fix 3: this stream touched no widget, opened no asset editor and compiled no blueprint, and had not yet made a single call it could have sequenced differently — a documentation fix on the widget pages cannot reach it. Secondary observation from the same event, filed separately as `E-editor-not-running-cannot-distinguish-crash`: the `EDITOR_NOT_RUNNING` text unconditionally prescribes `editor_start`, which is the one action an asset-only agent in a shared editor must not take.
- `#2-blast-radius-measured-from-another-agent` `OPEN` reporter — Independent confirmation and a measured cost, from a **different agent in the same shared editor**, 2026-09-02, UE 5.8, EAContentExamples58 checkout. I was authoring Niagara impact systems under `/Game/FPS/VFX/` and had no involvement with any Widget Blueprint; my `niagara.set_property` batch simply went `[WinError 10054]` mid-stream and then connection-refused. The log crash at `19.44.57` is the one this ticket already names, frame `[377]`, `UUserWidget::RebuildWidget` → `SDesignerView::UpdatePreviewWidget` → Slate paint, `StaticShutdownAfterError`. Concrete blast radius on my side: **12 of 13 Niagara emitter assets lost their in-memory configuration** — module adds (`GravityForce`, `Drag`, `CurlNoiseForce`, `MeshRotationRate`, `Collision`, `ScaleSpriteSize`), three sprite→mesh renderer swaps, and 15 renderer material assignments, roughly 60 successful RPCs of work. What survived is only what an unrelated agent's `editor.save_all` had flushed minutes earlier: the 13 duplicated emitters at an early state (one `AddVelocityInCone` each) plus the one emitter I had explicitly `asset.save`d. Verified on disk by `.uasset` mtime and by `grep -ac` for `MeshRendererProperties` / `GravityForce` / `MI_FPS` (all zero on the twelve). This is the argument for the guard being on the **verb** rather than on caller discipline: no discipline available to me could have prevented it, because the open Designer was not mine — and the documented `EDITOR_OPEN` guard that `niagara.add_emitter` already carries for exactly this shared-process reason has no peer on the widget capture path.

- `#3-same-shape-on-STATIC-MESHES-render.capture_asset_preview-then-model.compile` `OPEN` reporter — **Third encounter, and it is on the asset class this project had just ruled safe.** Same two-step shape as `#1` and `#2` one domain over: a capture verb leaves an asset editor open, a later COMPILE of that same asset kills the editor. Here the pair is `render.capture_asset_preview` + `model.compile` on a `UStaticMesh`, not `widget.screenshot_designer` + `blueprint.compile_bpir` on a widget.

  Sequence, on UE 5.8 / EAContentExamples58, all against `/Game/FPS/Weapons/Meshes/SM_WPN_AR`:

```
1. render.capture_asset_preview  views:"sides"     closeAfterCapture:false  -> ok, 6 shots
2. render.capture_asset_preview  single framed     closeAfterCapture:false  -> ok, assetEditorWasAlreadyOpen:true
3. render.capture_asset_preview  single framed     closeAfterCapture:false  -> ok, assetEditorWasAlreadyOpen:true
4. (edit the .pwmodel on disk)
5. model.compile  outputPath:/Game/FPS/Weapons/Meshes/SM_WPN_AR  save:true
      -> EDITOR_NOT_RUNNING (connection refused); system.ping confirms the process is gone
```

  `assetEditorWasAlreadyOpen: true` on calls 2 and 3 is the evidence that one static mesh editor was open across the whole run, and `model.compile` rewrites the very `UStaticMesh` that editor is displaying. That is `#1`'s mechanism exactly — open editor, asset rebuilt underneath it, editor dies on the repaint.

  **Attribution caveat, stated because it changes how much this is worth:** two other agents were live in the same editor and one had captures queued, so I cannot prove my compile was the trigger rather than a coincident death. What I can say is that the sequence is the documented mechanism, the timing is immediate, and nothing else in my own call history is a candidate. Treat this as a strong third data point, not a controlled repro. A controlled one is cheap and nobody has run it: open a mesh editor with `closeAfterCapture:false`, `model.compile` over that asset, in an editor with one agent in it.

  **Why this matters more than a third tick.** `Docs/fps/PLAN.md` rule 10 in the host project bans opening an asset editor to screenshot it and cites THIS ticket. On 2026-09-03 it was given an exception — `render.capture_asset_preview` allowed for **static meshes and materials only**, on the reasoning that this ticket's crash is a widget/Niagara toolkit problem and meshes are the safe case. This encounter says the asset class was never the discriminator: the discriminator is *an editor left open across a rebuild of its own asset*, and a static mesh editor is exposed to it through `model.compile` the same way a widget is through `blueprint.compile_bpir`.

  **The condition attached to that exception is also ordered wrong, and that is the actionable part.** It reads "provided the caller runs `editor.close_asset` on anything it opened before returning". "Before returning" is satisfied by closing at the end of a capture run — which is what I was doing — and it is exactly wrong, because the compile came first. The condition needs to be **close before any write to the asset you opened**, not before returning. And `closeAfterCapture:false`, which `B-capture-asset-preview-no-safe-close-mode` makes the safe choice for the teardown path, makes THIS failure more likely rather than less, because it is the setting whose entire purpose is to leave the editor open. The two tickets' safe configurations point in opposite directions and neither says so.

  **Suggested fix, and the precedent already exists in this plugin.** `model.compile` should refuse, or close-then-compile, when an asset editor is open on its `outputPath` — the same guard the material handlers already carry. `agent-conventions.md` records it: "Mutating material handlers load via `LoadMaterialForMutationOrReportError` — a bare `LoadObject` mutates a copy the open `FMaterialEditor` silently discards; fail loud with `EDITOR_OPEN`." Materials fail loud on this; static-mesh compile does not check at all. `B-material-graph-mutators-bypass-editor-open-guard` is the same guard being bypassed one domain over, so the pattern is established and the gap is that `model.compile` never adopted it. A typed `EDITOR_OPEN` from `model.compile` would have cost me one retry instead of an editor and everyone else's session in it.
