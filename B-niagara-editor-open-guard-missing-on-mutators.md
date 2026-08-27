---
id: B-niagara-editor-open-guard-missing-on-mutators
title: "Only add_emitter and remove_emitter carry the EDITOR_OPEN guard; four other niagara.* mutator files have none"
status: IN-REVIEW
severity: High
category: bug
tags: [niagara, editor-open-guard, crash-adjacent, slate, multi-agent, shared-editor, EDITOR_OPEN]
encounters: 1
lastSeen: 2026-08-27
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
