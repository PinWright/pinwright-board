---
id: F-niagara-set-module-script
title: "Cannot swap the script reference behind an existing stack module entry"
status: DONE
severity: High
category: feature
tags: [niagara, stack, module, script-ref, authoring]
---

# Cannot swap the script reference behind an existing stack module entry

A Niagara stack module is a `UNiagaraNodeFunctionCall` that references a
`UNiagaraScript` asset (the actual module body). The references are
stored on the function-call node's `FunctionScript` UPROPERTY plus the
versioned `SelectedScriptVersion` Guid.

There is **no `niagara.set_module_script`**. Confirmed by
`call("niagara.set_module_script")` returning NOT FOUND.

`niagara.set_property` with the function-call node as target can edit
reflected fields, but `FunctionScript` is not a simple reassignment —
swapping it requires re-resolving pin mappings, dropping orphaned
override entries, and triggering script-source refresh on the owning
emitter graph. The generic property path doesn't do any of that.

## Why it matters

Common authoring scenarios that need script swap:

1. **Version upgrade** — module author shipped `CurlNoiseForce` v2 with
   the same input names but a different script asset path; existing
   systems need a one-shot update of every reference. Today: remove + re-add
   for each emitter, losing position and any user overrides.
2. **Substitution** — swap "Add Velocity" for "Add Velocity From
   Direction" (compatible input shape, different math). Same problem.
3. **Refactor module location** — an author renames
   `/Game/FX/Modules/Foo` to `/Game/FX/Common/Modules/Foo`. Asset
   redirectors handle load, but if redirector fixup runs and the system
   isn't open in the editor, the function-call node may be left with a
   stale-but-valid script path until the next manual edit.

## Workaround

`niagara.remove_module` + `niagara.add_module`, then re-establish
every input override. Loses:

- Stack position (the new module lands at end of group; need
  `move_module` to restore).
- All `set_module_input` overrides.
- All `set_static_switch` resolutions.
- Module enabled/disabled state.

Each of those is recoverable individually, but the multi-step recovery
is far more error-prone than an atomic swap.

## Proposal

```
niagara.set_module_script(
    assetPath, target, entryId,
    scriptPath: string,         // new UNiagaraScript asset path
    scriptVersion?: string,     // optional version Guid
    preserveOverrides?: boolean,// default true — keep input overrides whose names+types match the new script
    compile?: boolean,
    save?: boolean
) -> {
    swapped: true,
    previousScript: string,
    preservedOverrides: [string],
    droppedOverrides: [{name: string, reason: "input_removed"|"type_changed"}],
    addedDefaults: [string]     // new inputs that didn't exist on the old script
}
```

Implementation surface: `FNiagaraStackGraphUtilities::SetMessageScript`
is engine-private; the equivalent is
`UNiagaraNodeFunctionCall::FunctionScript = NewScript` + invalidate the
cached compile info + `RefreshFromExternalChanges` on the owning graph +
re-allocate pins. Match input pins by name on the new vs old signature,
drop orphans, populate new defaults. Reuse the established view-model
cache to invalidate any open editor.

Edge cases:

- New script has `Usage` ≠ old script's `Usage`: reject with
  `INCOMPATIBLE_SCRIPT_USAGE` (e.g. cannot swap a `Module` into a
  `DynamicInput` slot).
- New script's `ModuleUsageBitmask` doesn't include the current stack
  group: reject with `INCOMPATIBLE_STACK_GROUP`.
- Versioned scripts: if `scriptVersion` is omitted, use the script's
  current exposed version.

## Cross-ref

- `niagara.add_module` / `niagara.remove_module` — the multi-step
  workaround surface.
- `F-search-api-niagara-modules` — discovery half; pairs with this
  ticket for "find a compatible script then swap".
- `F-niagara-event-handler-simstage-authoring` (DONE) — established
  pattern of script-source refresh + view-model cache invalidation.

## History
- `#1-no-set-module-script` `OPEN` reporter — Confirmed `niagara.set_module_script` doesn't exist. The function-call node's `FunctionScript` UPROPERTY can technically be reflected via `set_property`, but a real swap requires pin re-resolution, override mapping, and script-source refresh — none of which the generic property path performs. Workaround is remove+re-add which loses stack position, overrides, static-switch resolutions, and enabled state. Proposes `niagara.set_module_script` with `preserveOverrides` flag, returning kept/dropped override lists for the caller to audit.
- `#2-reviewed-and-confirmed` `OPEN` reviewer — Verified against source: no `set_module_script` handler in `Handlers/Niagara/`. `target.kind: "module"` resolves to `UNiagaraNodeFunctionCall` and `FunctionScript` is `UPROPERTY(EditAnywhere, Category="Function")` on that node (UE 5.6 `NiagaraNodeFunctionCall.h:61`), so `niagara.set_property` could mechanically assign the pointer — but as the ticket notes, that bypasses pin reallocation, override remap, and view-model refresh. Even the legacy `niagara.graph.add_module` only does `FuncNode->FunctionScript = ModuleScript;` for fresh-node creation, which is insufficient for mutating an existing node with overrides. Severity High is justified for batched version-upgrade scenarios; remove+re-add is destructive (loses overrides, position, enabled state, static-switch resolutions). No duplicate ticket. Ticket retained as-is.
- `#3-implemented-set-module-script` `IN-REVIEW` developer — Added `niagara.set_module_script` in `NiagaraEditHandler.cpp` with snapshot/replay override flow. Uses `UNiagaraNodeFunctionCall::RefreshFromExternalChanges` after FunctionScript reassignment. Compatibility validated via Usage equality + IsSupportedUsageContextForBitmask. Helpers `SnapshotInputOverrides` and `EnumerateScriptInputs` exposed via `NiagaraSetModuleScriptHelpers.h` for test reach.
- `#4-fix-review-issues` `IN-REVIEW` developer — Moved KillSystemInstances before ModuleNode->Modify to avoid rendering partial state; eliminated duplicate SnapshotInputOverrides call via OutOverrideSnapshot out-param on ApplyModuleMutation; added EnumerateScriptInputsRealGraph, FunctionScriptSwapCounterfactual, and RejectsIncompatibleUsagePredicate tests to TestNiagaraSetModuleScript.cpp; documented why SwapPreservesMatchingOverride/DropsTypeMismatchedOverride/RejectsIncompatibleStackGroup dispatcher-path tests are deferred; skipped SnapshotInputOverrides reuse refactor (T4 owns FindStackFunctionOverrideNode export).
- `#5-crash-during-set-module-script` `OPEN` tester — Crashed: live verification confirmed `niagara.set_module_script` is registered, then attempted `niagara.set_module_script` on `/App/ChemicalPlantEnv/CloudPlane/NS_Smoke.NS_Smoke` entry `87F0A10C40B1E128540E599959D56A21`; the RPC returned `fetch failed`, follow-up `asset.search` and `/health` could not connect, and `Saved/Logs/PDS.log` shows `EXCEPTION_ACCESS_VIOLATION` in `UNiagaraEmitter::CreateWithParentAndOwner`.
- `#6-reverify-invalid-stack` `OPEN` tester — Returned: fresh-editor retry no longer crashed, but `niagara.set_module_script` rejected the same existing module entry with `INVALID_STACK: Module '87F0A10C40B1E128540E599959D56A21' is not in a valid stack group`; expected a same-script no-save swap to succeed on the stack module. Test: `niagara.set_module_script` with assetPath `/App/ChemicalPlantEnv/CloudPlane/NS_Smoke.NS_Smoke`, emitter `Fountain`, entryId `87F0A10C40B1E128540E599959D56A21`, scriptPath `/Niagara/Modules/Emitter/SpawnRate.SpawnRate`, `preserveOverrides:true`, `compile:false`, `save:false`.
- `#7-infer-owning-stack` `IN-REVIEW` developer — Fixed `niagara.set_module_script` to infer the owning stack output from the resolved module entry when `scriptUsage` is omitted, so existing emitter-stack modules are validated against their real stack instead of the ParticleUpdate default. Added dispatcher regression `EditorAutomationRpcGateway.niagara.set_module_script.InfersOwningEmitterUpdateStack` for a same-script no-save swap on an EmitterUpdate module; explicit `scriptUsage` behavior remains unchanged.
- `#8-verify-fix` `DONE` tester — Verified: replayed the #6 test (`niagara.set_module_script` on `/App/ChemicalPlantEnv/CloudPlane/NS_Smoke.NS_Smoke`, emitter `Fountain`, entry `87F0A10C40B1E128540E599959D56A21`, scriptPath `/Niagara/Modules/Emitter/SpawnRate.SpawnRate`, `preserveOverrides:true`, `compile:false`, `save:false`, no `scriptUsage`). Got `success:true, swapped:true, previousScript:/Niagara/Modules/Emitter/SpawnRate.SpawnRate, preservedOverrides:[], droppedOverrides:[], addedDefaults:[InputMap]` — no more `INVALID_STACK`. Schema check via `niagara.set_module_script?` confirms `scriptUsage` remains optional.
