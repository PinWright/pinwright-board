---
id: B-niagara-edit-with-open-asset-editor-slate-crash
title: "`niagara.add_emitter` returns success and then kills the editor on the next Slate redraw — the open Niagara System Editor's title bar dereferences the emitter handle the mutation just invalidated, and no `niagara.*` verb has an EDITOR_OPEN guard"
status: DONE
severity: Critical
category: bug
tags: [niagara, add_emitter, remove_emitter, crash, access-violation, editor-open-guard, slate, stale-pointer, multi-agent]
encounters: 1
lastSeen: 2026-08-28T09:31:00+05:00
---

# A structural `niagara.*` edit made while the system's asset editor is open kills the process at the next Slate redraw, on a freed emitter handle

`niagara.add_emitter` rebuilds a `UNiagaraSystem`'s emitter-handle array. If that
system's asset editor is open, the open `SNiagaraOverviewGraphTitleBar` still
holds the emitter it was constructed against. The RPC **returns success** —
`emitterCount: 2, compiled: true` — and the process dies a moment later, on the
next ordinary Slate frame, when the title bar's visibility delegate dereferences
the handle the mutation freed.

`EXCEPTION_ACCESS_VIOLATION reading address 0x00000008000000b8`. The whole editor
process dies, taking every other agent's unsaved work with it.

The crash is not in the mutation and not in any capture — it is in a **redraw**,
which means nothing in the call's own response can ever report it and no
error is attributable to it.

## Root cause (guilty source line)

**The mutation:** `Plugins/PinWright/Source/PinWright/Private/Handlers/Niagara/NiagaraHandler.cpp:89`,
inside the `niagara.add_emitter` handler registered at `:39`:

```cpp
    NewHandle = System->AddEmitterHandle(*Emitter, HandleName, EmitterVersion);
```

`AddEmitterHandle` reshapes `System->GetEmitterHandles()`. The sibling
`niagara.remove_emitter` does the same at `NiagaraHandler.cpp:206`
(`System->RemoveEmitterHandlesById(ToRemove)`) and is the obvious second
offender.

**The missing guard, which is the actual defect.** No `niagara.*` handler
performs an open-asset-editor check before mutating. Verified across this tree:
`EDITOR_OPEN` is emitted from exactly two places, both in the **material**
cluster — `Handlers/Material/MaterialFinders.h:144` and `:262` — and
`NiagaraHandler.cpp` contains no `FindEditorForAsset` call at all. So a Niagara
system with a live `FNiagaraSystemToolkit` is mutated underneath that toolkit
with nothing checking, nothing refusing, and nothing warning.

**This is the confirmation an existing ticket explicitly asked for.**
`B-material-graph-mutators-bypass-editor-open-guard` (DONE, High) closes with a
section headed "Related — possible systemic exposure (UNVERIFIED, investigate
separately before ticketing)" which reads: *"Niagara mutators
(`NiagaraEditHandler`, `NiagaraAdvancedEditHandler`, `NiagaraCurveHandler`,
`NiagaraGraphHandler`) and Blueprint graph handlers also `LoadObject` + mutate
assets backed by working-copy editors (`FNiagaraSystemToolkit`,
`FBlueprintEditor`) with no analogous guard. Confirm which editors clobber-on-save
before filing."* This ticket is that confirmation — **and the outcome is worse
than that ticket predicted.** It predicted a silent clobber (the open editor's
working copy overwriting your edit, lossy but survivable). What actually happens
is the inverse: the open editor's *widget* reads your edit's freed data and takes
the process down. The "confirm clobber-on-save before filing" gate should be
dropped; a process kill does not need a clobber to justify a guard.

## Evidence

Verified on disk in this checkout during filing.
`Saved/Logs/EAContentExamples58-backup-2026.08.27-09.49.42.log:7141`, crash at
`2026.08.27-09.49.42` UTC (14:49 local; logs are UTC+0, this machine is UTC+5):

```
Unhandled Exception: EXCEPTION_ACCESS_VIOLATION reading address 0x00000008000000b8
```

Callstack `:7143` onward, innermost first:

```
UNiagaraEmitter::GetEmitterData()                          NiagaraEmitter.cpp:2538
SNiagaraOverviewGraphTitleBar::IsUsingDeprecatedEmitter()  SNiagaraOverviewGraphTitleBar.cpp:386
SNiagaraOverviewGraphTitleBar::GetSystemSubheaderVisibility() SNiagaraOverviewGraphTitleBar.cpp:221
TBaseSPMethodDelegateInstance<...EVisibility...>::Execute() DelegateInstancesImpl.h:266
SWidget::Prepass_Internal()                                SWidget.cpp:1826  (x36)
FSlateApplication::DrawPrepass()                           SlateApplication.cpp:1413
FSlateApplication::PrivateDrawWindows()                    SlateApplication.cpp:1465
FSlateApplication::DrawWindows()
```

Precursor at `:7135`, 31 seconds before the fault:
`LogUObjectGlobals: Warning: Failed to find object 'Class NiagaraEmitter'`.

The last PinWright line before the crash is another agent's `actor.spawn_batch`
at `09.48.54`, so nothing else was touching Niagara.

**Measured shape of the callstack, because a sibling report got this wrong.** The
full stack is **120 frames**, of which **36** are `SWidget::Prepass_Internal`.
This is an ordinary-depth widget tree drawing one frame — it is **not** a
recursion-depth blowout and **not** `EXCEPTION_STACK_OVERFLOW`. Deep prepass
recursion is the *context* the fault happens in; the stale emitter pointer is
what is actually read. A fix that only caps how many asset editors may be open
would leave this crash live for anyone who edits a system with a **single**
editor open.

## Verbatim repro

Exactly two calls, both of which the current documentation tells you to make:

```
render.capture_asset_preview {subject: {kind: "niagara",
                                        path: "/Game/Atlantis/VFX/NS_Bubbles_Stream",
                                        closeAfterCapture: false}}

niagara.add_emitter {systemPath: "/Game/Atlantis/VFX/NS_Bubbles_Stream",
                     emitterPath: "/Niagara/DefaultAssets/Templates/Emitters/BlowingParticles.BlowingParticles",
                     name: "ForceTest", compile: true, save: false}
```

`add_emitter` returns `{emitterCount: 2, compiled: true}`. The process dies on
the next frame. Reproduced once; the sequence is short and deterministic.

Note what the first call is: it is the documented workaround for
`B-capture-asset-preview-no-safe-close-mode`. **The remedy for that crash is the
trigger for this one.**

## What it should do

Refuse the mutation. Every `niagara.*` verb that reshapes a system's structure —
`add_emitter`, `remove_emitter`, and anything else touching the emitter-handle
array — should take the same shape the material cluster already ships: check
`UAssetEditorSubsystem::FindEditorForAsset(Asset, /*bFocusIfOpen=*/false)` and
send `EDITOR_OPEN` with an actionable hint, exactly as
`Handlers/Material/MaterialFinders.h:144` does. That code, the error constant
(`Handlers/ErrorCodes.h:371`), and the precedent are all already in the tree; the
Niagara cluster simply never adopted them.

Refusing is strictly better than the alternatives here. A window left open faults
only later; a mutation under an open toolkit faults on the next frame with no
attributable error, and every unsaved edit in the process goes with it.

## Do not fix this one alone

This is one of three linked defects that together mean
`render.capture_asset_preview` has no safe usage:

- `B-capture-asset-preview-no-safe-close-mode` — `closeAfterCapture: true` faults
  inside `CaptureSubject::CloseAssetEditor()`; `closeAfterCapture: false` leaks
  every editor it opens (37 opened, 0 closed, measured).
- **This ticket** — with an editor left open by that workaround, any structural
  Niagara edit kills the process.

Fixing only the teardown leaves this crash live for anyone who passes `false`.
Fixing only this leaves the teardown fault live for everyone who does not. Verify
a fix by running both paths.

## Workaround

Treat "asset editor open" and "system structurally edited" as mutually exclusive:
do not author a Niagara system through `niagara.*` while a
`render.capture_asset_preview` has left its editor open. Prove Niagara from the
**level** instead — `niagara.spawn_actor` + `effect.activate_niagara` +
`effect.advance_simulation` + `render.capture_open_level` — which never opens an
asset editor. Note this makes the sibling ticket's stated workaround unusable for
any asset you are still editing, which is every asset while you are building it.

Separately, and generally: `save: false` on a `niagara.*` edit means the change
lives only in memory. With crashes this frequent, pass `save: true` on every edit
you would not want to redo, not just the last in a batch. On the session that
produced this ticket, the in-memory `Mass` / `Write Mass` / `Acceleration` /
bounds-mode edits made after the previous explicit save were all lost.

## Distinct from related tickets

- `B-material-graph-mutators-bypass-editor-open-guard` (DONE, High) — the parent.
  Same missing guard, different namespace, and it predicted a silent clobber
  rather than a crash. This ticket supplies the Niagara confirmation it asked for.
- `F-niagara-remove-emitter` and `F-niagara-set-module-script` both carry live
  `EXCEPTION_ACCESS_VIOLATION` tester entries on Niagara verbs
  (`UNiagaraSystem::AddEmitterHandle()` on a missing `GraphSource`;
  `UNiagaraEmitter::CreateWithParentAndOwner`), but both fault **inside the
  mutation itself**, with no asset editor open and no Slate frame involved. Both
  were subsequently verified non-crashing. Distinct fault, distinct precondition.
- `B-editor-quit-crash-open-asset-editors` (IN-REVIEW, High) — open asset editors
  plus an AV, but at `FEngineLoop::Exit`, in `~FStaticMeshEditor` delegate
  teardown. Different trigger, different stack, and its fix does not reach a
  mid-session redraw.

severity rationale: impact=editor crash on the next redraw after a call that reported success, killing every agent in the shared process and losing all unsaved work, with no error attributable to it x reach=triggered by the documented workaround for a sibling Critical defect, on the normal path for authoring any Niagara system -> Critical

## History
- `#1-initial-repro` `OPEN` reporter — Found building the Atlantis level (map as forcing function; see host `CLAUDE.md` § "What this project is for"), 2026-08-27, UE 5.8, PinWright at this checkout's HEAD. Two-call deterministic repro: `render.capture_asset_preview {closeAfterCapture:false}` on `/Game/Atlantis/VFX/NS_Bubbles_Stream`, then `niagara.add_emitter` of `BlowingParticles` — `add_emitter` returned `{emitterCount:2, compiled:true}` and the process died on the next Slate frame. Crash re-verified on disk in THIS checkout during filing: `EAContentExamples58-backup-2026.08.27-09.49.42.log:7141`, `EXCEPTION_ACCESS_VIOLATION reading address 0x00000008000000b8`, innermost frame `UNiagaraEmitter::GetEmitterData() NiagaraEmitter.cpp:2538` under `SNiagaraOverviewGraphTitleBar::IsUsingDeprecatedEmitter() :386`, precursor `LogUObjectGlobals: Warning: Failed to find object 'Class NiagaraEmitter'` at `:7135`. **Measured the callstack to settle a competing diagnosis**: 120 frames total with 36 `SWidget::Prepass_Internal` — an ordinary-depth widget tree, not a recursion blowout, and an access violation rather than a stack overflow. A sibling session entry had attributed this same crash to Slate stack exhaustion over 37 accumulated asset editors; that is the context, not the cause, and a fix that only caps open editors would leave this live for a single open editor. Source-confirmed the missing guard at HEAD in this tree: `niagara.add_emitter` registers at `NiagaraHandler.cpp:39` and mutates via `System->AddEmitterHandle(...)` at `:89` (sibling `remove_emitter` via `RemoveEmitterHandlesById` at `:206`), while `EDITOR_OPEN` is emitted from only two sites in the whole plugin — `Handlers/Material/MaterialFinders.h:144` and `:262` — and `NiagaraHandler.cpp` contains no `FindEditorForAsset` call at all. This is the confirmation `B-material-graph-mutators-bypass-editor-open-guard` explicitly deferred ("Confirm which editors clobber-on-save before filing"), with a worse outcome than it predicted: not a clobber, a process kill. Worked around by proving Niagara from the level (`niagara.spawn_actor` + `effect.activate_niagara` + `effect.advance_simulation` + `render.capture_open_level`); defect untouched.
- `#2-editor-open-guard-on-structural-edits` `IN-REVIEW` developer — Added the Niagara open-asset-editor guard the ticket asked for. New header `Plugins/PinWright/Source/PinWright/Private/Handlers/Niagara/NiagaraEditorOpenGuard.h` supplies `PinWrightNiagara::FindOpenAssetEditorForNiagaraAsset` / `IsNiagaraAssetEditorOpen` / `RejectStructuralEditWhileAssetEditorOpen`, taking the same `UAssetEditorSubsystem::FindEditorForAsset(Asset, /*bFocusIfOpen=*/false)` shape the material cluster ships in `Handlers/Material/MaterialFinders.h`. Both emitter-handle mutators in `Handlers/Niagara/NiagaraHandler.cpp` now call it immediately after loading the system and before any mutation — `niagara.add_emitter` (before `AddEmitterHandle`) and `niagara.remove_emitter` (before `RemoveEmitterHandlesById`) — so the refusal, not the crash, is what the caller observes. The check keys off the **asset**, never off who opened the editor, so a toolkit another agent left open in the shared process refuses the write. Error code is the already-registered `EDITOR_OPEN` (`ErrorCodes::ERR_EDITOR_OPEN`, `Handlers/ErrorCodes.h:371`); no new code was needed. The message names the open editor via `IAssetEditorInstance::GetEditorName()`, names `editor.close_asset` as the way out, and says the editor may belong to another caller. The guard lives in its own header rather than in `NiagaraHandler.cpp` for two reasons: the remaining `niagara.*` mutators (`NiagaraEditHandler`, `NiagaraAdvancedEditHandler`, `NiagaraCurveHandler`, `NiagaraGraphHandler`) can adopt it without a second copy, and `NiagaraHandler.cpp` still emits raw error-code literals, so introducing `ErrorCodes::ERR_` into that file would flip it to "adopting" and fail `PinWright.core.error_codes.RegistryAdoptingFilesUseConstantsOnly` on its ~28 pre-existing literals. Regression test `PinWright.niagara.editor_open_guard.StructuralEditsRefuseWhileAssetEditorOpen` in `Source/PinWright/Private/Tests/Niagara/TestNiagaraEditorOpenGuard.cpp`: builds a transient system with one emitter handle plus a probe emitter (`NiagaraEditTestUtils`), registers a stub `IAssetEditorInstance` through `UAssetEditorSubsystem::NotifyAssetOpened`, asserts both verbs answer `EDITOR_OPEN` and that the handle count does not move, then unregisters and asserts the identical calls succeed and do move it. **The test asserts the guard, not the crash, and is built so it cannot provoke the crash**: the stub editor is not a Slate widget, so a reverted guard fails assertions instead of killing the suite host. Not compiled or run — the orchestrator owns builds. Scope note: this fixes only the two emitter-handle mutators in `NiagaraHandler.cpp`; the sibling `niagara.*` mutator files still have no guard, and the linked `B-capture-asset-preview-no-safe-close-mode` teardown fault is untouched.
- `#3-runtime-verified-verbatim-repro-now-refuses` `DONE` verifier — 2026-08-28, rebuilt DLL at plugin HEAD `b79ba53e`, editor pid 14932. **This ticket's verbatim two-call repro was executed and no longer kills the process.**
  Call 1, the documented workaround for the sibling ticket: `render.capture_asset_preview {subject:{kind:"niagara", path:<system>, closeAfterCapture:false}}` — response confirmed the window was genuinely left open (`assetEditorClosed:false`, `assetEditorCloseDeferred:false`), so the precondition was real and not assumed. Call 2: `niagara.add_emitter {emitterPath:"/Niagara/DefaultAssets/Templates/Emitters/BlowingParticles.BlowingParticles", name:"ForceTest", compile:true, save:false}` — answered `EDITOR_OPEN`, naming the open 'Niagara' asset editor, the mechanism (`FNiagaraEmitterHandleViewModel::EmitterHandle` is a raw pointer into the handle array), `editor.close_asset` as the way out, and that the editor may belong to another caller sharing the process. No `emitterCount`, no `compiled: true`, no mutation, and no death on the following frames. `niagara.remove_emitter` refuses identically with the same clause.
  **Counterfactual measured, so this is a guard and not a blanket refusal.** After `editor.close_asset` on the same system, the byte-identical `add_emitter` succeeded and moved the count (`emitterCount: 2`). The guard keys on the asset's open state only.
  **Held open afterwards, because the fault was a redraw and not the call.** The same pair was also run with the toolkit opened via `editor.open_asset` rather than a capture, and the toolkit was then left open for ~3 minutes of ordinary Slate frames while four further structural edits ran against it. Zero asserts, zero access violations, no `EXCEPTION_ACCESS_VIOLATION` anywhere in the session log, and a `register_slate_post_tick_callback` probe showed the editor ticking throughout (22,870 ticks, longest stall 2.30 s). The precursor this ticket records — `LogUObjectGlobals: Warning: Failed to find object 'Class NiagaraEmitter'` — did not appear either.
  Source at this HEAD: `NiagaraEditorOpenGuard.h` exists and supplies all three entry points, keyed on `UAssetEditorSubsystem::FindEditorForAsset(Asset, /*bFocusIfOpen=*/false)` (`:57`), emitting `ErrorCodes::ERR_EDITOR_OPEN` (`:100`); `add_emitter` calls it at `NiagaraHandler.cpp:204`, ahead of the emitter load (`:209`), the transaction (`:236`) and `AddEmitterHandle` (`:254`); `remove_emitter` calls it at `:384`, ahead of handle resolution and `RemoveEmitterHandlesById` (`:456`).
  Run against a scratch duplicate of `NS_Bubbles_Stream` rather than the shipped asset, deliberately: had the guard failed, the mutation would have landed on map content before the process died.
  Scope unchanged and correctly tracked elsewhere — the sibling `niagara.*` mutator files are `B-niagara-editor-open-guard-missing-on-mutators` (also verified today), and the capture teardown is `B-capture-asset-preview-no-safe-close-mode` (also verified today), so the "do not fix this one alone" condition is satisfied: all three members were run in one session.
