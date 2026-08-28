---
id: B-niagara-editor-open-guard-missing-on-mutators
title: "Only add_emitter and remove_emitter carry the EDITOR_OPEN guard; four other niagara.* mutator files have none"
status: DONE
severity: High
category: bug
tags: [niagara, editor-open-guard, crash-adjacent, slate, multi-agent, shared-editor, EDITOR_OPEN]
encounters: 1
lastSeen: 2026-08-28T09:31:00+05:00
---

# The open-asset-editor guard covers two verbs out of a family

`B-niagara-edit-with-open-asset-editor-slate-crash` is fixed for `niagara.add_emitter` and
`niagara.remove_emitter`: `Handlers/Niagara/NiagaraEditorOpenGuard.h` refuses with `EDITOR_OPEN` when
any asset editor holds the target open, because reshaping the emitter-handle array underneath a live
`FNiagaraSystemToolkit` faults on the next Slate redraw.

The guard is asset-class-agnostic (`UObject*`) and adopting it is a one-line call, but four other
mutator files still have none: `NiagaraEditHandler.cpp`, `NiagaraAdvancedEditHandler.cpp`,
`NiagaraCurveHandler.cpp`, `NiagaraGraphHandler.cpp`.

Whether each of those verbs can actually fault an open toolkit is **not established** -- the proven
crash is specifically the emitter-handle reshape. A structural graph edit under an open Niagara editor
is at minimum in the same neighbourhood, and the guard costs one line. This should be decided per
verb rather than blanket-applied: refusing a harmless edit because a toolkit is open is its own
ergonomic cost, and in a shared editor the toolkit is often someone else's.

**Related, worth doing at the same time:** the guard duplicates
`PinWright::Material::IsMaterialEditorOpen` in shape. Consolidating both into one plugin-wide
`EDITOR_OPEN` guard is close to a rename and would put every namespace on the same behaviour.

## History
- `#1-scope-note-from-the-fix` `OPEN` reporter -- Raised by the agent that fixed
  `B-niagara-edit-with-open-asset-editor-slate-crash`, which deliberately scoped itself to the two
  emitter-handle mutators the crash was proven on. Source-level claim; no crash reproduced on the
  other four files.
- `#2-classified-32-verbs-guarded-two` `IN-REVIEW` developer -- Classified all 32 verbs in the four
  files against one criterion read out of UE 5.8 source: does the mutation invalidate something a
  live toolkit widget caches by bare pointer or value snapshot AND reach no notification the toolkit
  is subscribed to. 28 of 32 fail that test and are deliberately left unguarded -- the engine
  mutators they call broadcast `OnRenderersChanged` / `OnSimStagesChanged` /
  `FNiagaraParameterStore::OnLayoutChange` / `GRAPHACTION_RemoveNode`, or
  `UNiagaraSystem::PostEditChangeProperty` reaches `FNiagaraSystemViewModel::RefreshAll`, and the
  surviving stack entries hold `TWeakObjectPtr`. Two families do not. **Guarded (2):**
  `niagara.add_event_handler` / `niagara.remove_event_handler` in `NiagaraAdvancedEditHandler.cpp`
  -- `UNiagaraStackEventHandlerPropertiesItem` snapshots `EventHandlerScriptProps` into a
  `UNiagaraStackEventWrapper` once and writes the whole copy back from that wrapper's
  `PostEditChangeProperty`, so an out-of-band write is reported as succeeding and then silently
  reverted, and no notification can repair it. **Fixed by notification instead of refusal (2):**
  `niagara.reset_module_input` (override-pin branch) and `niagara.clear_module_overrides` in
  `NiagaraEditHandler.cpp` `MarkAsGarbage()` an override `UEdGraphPin` while being the only two
  graph mutators in that file that ended at `NotifyNiagaraObjectChanged`; `UEdGraphNode::RemovePin`
  notifies only via `NotifyGraphNeedsRecompile`, which `UNiagaraGraph::NotifyGraphChanged`
  early-returns on before broadcasting `OnGraphChanged`, leaving the freed pin live in
  `UNiagaraStackFunctionInput::OverridePinCache` and `SGraphPin::GraphPinObj`. Both now also call
  `NotifyNiagaraGraphChanged`, which drops those caches inside the transaction and keeps the edit
  working rather than refusing it. `NiagaraCurveHandler.cpp` and `NiagaraGraphHandler.cpp` needed no
  change. `NiagaraEditorOpenGuard.h` gained a defaulted `HazardClause` parameter so a refusal names
  what would actually break instead of always citing the emitter-handle reshape; the two existing
  call sites are unchanged. Tests added to
  `Tests/Niagara/TestNiagaraEditorOpenGuard.cpp`:
  `PinWright.niagara.editor_open_guard.EventHandlerMutatorsRefuseWhileAssetEditorOpen` (both event
  verbs refuse, handler count unmoved, `niagara.add_simulation_stage` on the same open asset is
  **not** refused -- the anti-blanket assertion -- and the add succeeds once the editor closes) and
  `PinWright.niagara.clear_module_overrides.BroadcastsGraphChangedForOpenEditorCaches` (counts
  `UEdGraph::OnGraphChanged` broadcasts; 0 before the fix).
- `#3-runtime-verified-per-verb-classification-holds` `DONE` verifier — 2026-08-28, rebuilt DLL at plugin HEAD `b79ba53e`, editor pid 14932. All calls below were made against a live `FNiagaraSystemToolkit` opened with `editor.open_asset` on a scratch Niagara system, and the toolkit was left open and redrawing for ~3 minutes afterwards.
  **The two newly guarded verbs refuse, with their own hazard text.** `niagara.add_event_handler` and `niagara.remove_event_handler` both answer `EDITOR_OPEN` — and the message is the event-handler one, naming the `UNiagaraStackEventWrapper` snapshot of `EventHandlerScriptProps`, the whole-copy write-back and the silent revert. It is not the emitter-handle reshape clause. So the defaulted `HazardClause` parameter `#2` added works end to end, and a refusal now names what would actually break. (Minor: `remove_event_handler` validates arguments before reaching the guard — omitting `eventHandlerIndex`/`eventHandlerId` answers `INVALID_ARGUMENT` first. Harmless, since neither path mutates, but a caller probing for the guard needs valid arguments to see it.)
  **The anti-blanket assertion holds behaviourally.** `niagara.add_simulation_stage` on the same open asset is NOT refused — it created a real stage (`stageIndex: 0`, a `NiagaraSimulationStageGeneric` with a returned id) — and `niagara.remove_simulation_stage` removed it, also unrefused. That is the deliberate 28-of-32 classification actually exercised rather than reasoned about, and neither faulted the open toolkit.
  **The notification fix was exercised, not merely read.** `niagara.set_module_input` wrote a real override pin on `GravityForce.Gravity` under the open toolkit (`pinId` returned, `value: "(X=0.0,Y=0.0,Z=-420.0)"`), then `niagara.clear_module_overrides` reported `cleared: 1` naming `GravityForce.Gravity`. That is the code path `#2` describes actually running: an override `UEdGraphPin` was `MarkAsGarbage`d while a live `UNiagaraStackFunctionInput::OverridePinCache` / `SGraphPin::GraphPinObj` could still have been holding it. With `NotifyNiagaraGraphChanged` now dropping those caches inside the transaction, the edit took effect and the editor kept drawing. Zero asserts, zero access violations across the whole session; a `register_slate_post_tick_callback` probe recorded 22,870 ticks with the longest game-thread stall 2.30 s.
  **Scope re-verified against source at this HEAD** rather than trusted from `#2`: `NiagaraEditorOpenGuard.h` is included by `NiagaraHandler.cpp` and `NiagaraAdvancedEditHandler.cpp` only. Guarded there: `add_emitter` (`:204`), `remove_emitter` (`:384`), `add_event_handler` (`NiagaraAdvancedEditHandler.cpp:122-127`, hazard `EventHandlerStackSnapshot`), `remove_event_handler` (`:233-238`). Unguarded by design in the same file: `add_simulation_stage` (`:292`), `remove_simulation_stage` (`:392`), `add_data_interface` (`:460`), `remove_data_interface` (`:548`). `NiagaraEditHandler.cpp`, `NiagaraCurveHandler.cpp` and `NiagaraGraphHandler.cpp` carry no guard call at all, as intended; the two `NiagaraEditHandler.cpp` verbs fixed by notification instead are `reset_module_input`'s override-pin branch (`NotifyNiagaraGraphChanged` at `:3247`, inside the transaction after `RemoveOverridePinAndChainedNodes` at `:3235`) and `clear_module_overrides` (`:3367`). The static-switch branch of `reset_module_input` (`:3149-3160`) deliberately does not call it, and frees no pin.
  **Left undone, wants its own ticket rather than this one:** the consolidation with `PinWright::Material::IsMaterialEditorOpen` that the body proposes. The two guards are still separate implementations of the same `EDITOR_OPEN` shape.
