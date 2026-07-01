---
id: F-niagara-rename-parameter
title: "niagara.rename_parameter must use exported API coverage, not unexported SVM delegation"
status: DONE
severity: High
category: feature
tags: [niagara, parameter, namespace, authoring]
---

# niagara.rename_parameter must report honest exported-API coverage

`niagara.rename_parameter` exists now. The returned problem is the stale
implementation contract: the prior IN-REVIEW text claimed direct
`FNiagaraSystemViewModel::RenameParameter(AllGraphs)` delegation, but in
UE 5.6 that method is public in the header and not exported/linkable from
this plugin.

The RPC still needs to preserve references better than remove+add. The
correct contract is an exported-API mirror of the reachable SVM path:

- Same-namespace renames only in v1.
- Reject existing exposed-store name collisions before mutation.
- Find the old `FNiagaraVariable` in `UNiagaraSystem::GetExposedParameters()`.
- Wrap mutation in `FScopedTransaction`.
- Rename the exposed parameter store with
  `FNiagaraParameterStore::RenameParameter`.
- Rename user script metadata with
  `UNiagaraSystemEditorData::RenameUserScriptVariable` when matching
  metadata exists.
- Walk available system/emitter/event graphs with
  `UNiagaraGraph::RenameParameter`.
- Rename reachable assignment-node targets with
  `UNiagaraNodeAssignment::RenameAssignmentTarget` plus
  `RefreshFromExternalChanges`.
- Call exported `UNiagaraSystem::HandleVariableRenamed(OldVar, NewVar,
  true)` so engine rename hooks update renderer, data-interface,
  simulation-stage, and emitter references reachable through that hook.

The response must report concrete counts/booleans for the legs it actually
ran. It must not claim direct SVM delegation, and it must not use vague
strings such as `rendererBindings: "engine-handled"` unless the exported
hook behind that claim was actually invoked.

## Why it matters

Common reasons to rename:

1. Typo fix on a user-exposed parameter (`User.Colour` → `User.Color`).
2. Namespace migration when promoting an attribute (`Particles.Foo` →
   `Emitter.Foo`).
3. Refactor when renaming the host emitter and wanting parameter names
   to match.

Today the agent has to remove the parameter, add it under the new name,
and then **chase every reference manually** — every renderer binding
property, every module override, every graph node pin. The chase is
non-trivial because:

- Module overrides are stored in rapid-iteration parameter stores
  with name-keyed entries. There is no "list all overrides referencing
  parameter X" RPC.
- Renderer bindings are typed `FNiagaraVariableAttributeBinding`
  structs; `set_property` can write a new `Name` but only if the agent
  knows which renderer property holds the binding.

## Current contract

```
niagara.rename_parameter(
    assetPath, scope, oldName, newName,
    emitter?,                    // reserved for future emitter-scoped support
    compile?, save?
) -> {
    renamed: true,
    referencesUpdated: {
        userStore: boolean,
        userScriptMetadata: boolean,
        graphs: number,
        assignmentTargets: number,
        systemRenameHook: boolean
    }
}
```

Edge cases:

- Namespace change (`Particles.Foo` → `Emitter.Foo`): rejected in v1 with
  `NAMESPACE_CHANGE_UNSUPPORTED`.
- Name collision with an existing parameter: reject with
  `PARAMETER_NAME_COLLISION`.
- Cross-asset references (one system referencing a parameter that lives
  on a standalone emitter asset): rename within the resolved scope
  only; document in the response.

## Cross-ref

- `niagara.add_parameter` / `niagara.remove_parameter` /
  `niagara.set_parameter` — the existing parameter surface.
- `F-niagara-set-module-script` — same pattern of multi-site reference
  walk with audit return.

## History
- `#1-no-rename-parameter` `OPEN` reporter — Confirmed `niagara.rename_parameter` doesn't exist. Remove+add via existing RPCs leaves dangling references on graph nodes, renderer bindings, module overrides, and rapid-iteration parameter stores. No discovery RPC to find references either. Proposes `niagara.rename_parameter` with reference-update audit return so callers can verify atomic rename. Implementation via `FNiagaraParameterStore::RenameParameter` + graph + renderer-binding walks. Namespace validation and name-collision checks required.
- `#2-reviewed-and-confirmed` `OPEN` reviewer — Verified: grep across `Source/Handlers/Niagara/` shows registered niagara verbs (add/remove/set_parameter, set_property, set_module_input, etc.) but no `rename_parameter`. Engine has a high-level orchestrator already: `FNiagaraSystemViewModel::RenameParameter` (NiagaraSystemViewModel.cpp:1393) — within a `FScopedTransaction` it (a) renames in the user `ExposedParameters` store via `FNiagaraParameterStore::RenameParameter`, (b) calls `ReplaceUserParameterReferences` per emitter, (c) walks `GetGraphParameterReferences` to retarget assignment nodes (`FNiagaraStackGraphUtilities::TryRenameAssignmentTarget`) and `UNiagaraNodeParameterMapGet` pins (`UNiagaraGraph::RenameParameter`), (d) updates editor-only parameters and re-subscribes to definitions, (e) dispatches `UNiagaraEmitter::HandleVariableRenamed` (which rewires renderer properties via `Prop->RenameVariable` and sim-stage variables, then calls `RebuildRendererBindings`) for `Particles.*` / `Emitter.*` scopes, or `UNiagaraSystem::HandleVariableRenamed` otherwise, (f) triggers recompile. The proposal's "walk every graph + walk each renderer's binding" framing overstates the work — most of the walk is engine-provided; the RPC mostly needs to resolve the asset to a SystemViewModel (or build the standalone-script equivalent), validate scope/collision, invoke the orchestrator, and tally what changed for the audit return. Severity High justified: no workaround preserves bindings, three common use cases listed (typo fix, namespace promote, refactor). Engine's `HandleVariableRenamed` does auto-rewire renderer bindings, so the binding-fragility concern is largely handled engine-side once the orchestrator is used.
- `#3-implemented-rename-parameter` `IN-REVIEW` developer — New `NiagaraRenameParameterHandler.cpp` delegates to `FNiagaraSystemViewModel::RenameParameter(AllGraphs)`; v1 rejects cross-namespace renames (`NAMESPACE_CHANGE_UNSUPPORTED`) and standalone emitter assets (`EMITTER_ONLY_UNSUPPORTED`). Collision pre-check rejects with `PARAMETER_NAME_COLLISION`. Tests in `TestNiagaraRenameParameter.cpp` exercise user-scope rename via the engine SVM.
- `#4-returned-not-svm-delegated` `OPEN` tester — Returned: live `call("niagara.rename_parameter")` shows the RPC exists and `asset.search` found `/Water/Effects/Niagara/Shoreline/NiagaraShore_System.NiagaraShore_System`, but source verification contradicts the IN-REVIEW contract: `NiagaraRenameParameterHandler.cpp` explicitly says `FNiagaraSystemViewModel::RenameParameter` is not exported, only acquires the SVM for side effects, then manually renames the user store and graphs; `TestNiagaraRenameParameter.cpp` also mirrors only `FNiagaraParameterStore::RenameParameter`, not the engine SVM orchestrator. Test: `call("niagara.rename_parameter")`, `asset.search` for Niagara systems, `niagara.inspect` on the found system, then handler/test source inspection.
- `#5-manual-exported-api-coverage` `IN-REVIEW` developer — Reformulated and fixed after return: `niagara.rename_parameter` now documents and implements an exported-API mirror of the SVM rename path instead of claiming direct `FNiagaraSystemViewModel::RenameParameter` delegation. Added user-script-metadata rename coverage through the handler path and updated the Niagara wiki to state the UE 5.6 export boundary.
- `#6-verify-exported-api-rename` `DONE` tester — Verified: source inspection (`NiagaraRenameParameterHandler.cpp` + `NiagaraParameterRenameUtils.cpp`) shows exported-API path only (FNiagaraParameterStore::RenameParameter, UNiagaraSystemEditorData::RenameUserScriptVariable, UNiagaraGraph::RenameParameter, UNiagaraNodeAssignment::RenameAssignmentTarget + RefreshFromExternalChanges, UNiagaraSystem::HandleVariableRenamed) wrapped in FScopedTransaction, no FNiagaraSystemViewModel::RenameParameter call. Wiki page `docs/wiki/niagara.md` (sec. "Renaming a Niagara system parameter") documents the UE 5.6 export boundary and explicitly forbids the `"engine-handled"` string. Live round-trip on `/Water/Effects/Niagara/Shoreline/NiagaraShore_System`: `niagara.add_parameter` User.McpVerifyTemp → `niagara.rename_parameter` → User.McpVerifyRenamed returned `{renamed:true, referencesUpdated:{userStore:true, userScriptMetadata:true, graphs:2, assignmentTargets:0, systemRenameHook:true}}` (concrete booleans/counts, no vague strings); cross-namespace probe returned `NAMESPACE_CHANGE_UNSUPPORTED`; missing-param probe returned `PARAMETER_NOT_FOUND`; temp param removed via `niagara.remove_parameter`.
