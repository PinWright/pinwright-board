---
id: F-niagara-reset-module-input
title: "Cannot reset a module input override to its default (no clear-overrides op)"
status: DONE
severity: High
category: feature
tags: [niagara, stack, module, override, authoring]
---

# Cannot reset a module input override to its default

`niagara.set_module_input` writes an override into the stack module's
parameter store (rapid-iteration parameters) or the caller-pin default
(static switches via `niagara.set_static_switch`). There is **no
`niagara.reset_module_input` / `niagara.clear_module_overrides`**.
Confirmed by `call("niagara.reset_module_input")` and
`call("niagara.clear_overrides")` returning NOT FOUND.

Adjacent ops in the Blueprint surface exist: BPIR's `compile_bpir` can
re-emit a node body without a stale override, and the editor UI has a
"Reset to default" right-click on every overridden input. Niagara MCP
has neither path.

## Why it matters

Override iteration is the core inner loop of stack authoring:

1. Add a module with `niagara.add_module`.
2. Override an input with `niagara.set_module_input` to try a value.
3. Decide the override is wrong — want to **remove the override** and
   fall back to the module's authored default.

Step 3 is impossible. The agent must either:

- Set the override back to the value the module authored as its default
  (requires reading the module asset's pin defaults, which is doable
  but indirect, and breaks if the module author later changes the
  default).
- Remove the module and re-add it (loses position in the stack and any
  other valid overrides on the same module).
- `python.execute` calling
  `FNiagaraStackGraphUtilities::RemoveRapidIterationParametersForModule`.

This also blocks "clean state" workflows — e.g. before saving a
canonical template, agents often want to clear all overrides and let
defaults flow through. There's no bulk path either.

## Proposal

Two RPCs:

```
niagara.reset_module_input(
    assetPath, target, entryId, inputName,
    compile?, save?
) -> { reset: true, previousValue: any }
```

Removes the override entry from the rapid-iteration parameter store (or
the caller-pin default for static switches), returning the value that
was there. Module's authored default takes effect on next compile.

```
niagara.clear_module_overrides(
    assetPath, target, entryId,
    compile?, save?
) -> { cleared: number, inputs: [string] }
```

Bulk clear for one module entry — every override input falls back to
default. Returns the list of input names that were cleared.

Implementation surface: rapid-iteration parameter removal goes through
`UNiagaraScript::RapidIterationParameters` + the editor utility
`FNiagaraStackGraphUtilities::FindOrAddModuleInputNode` reversed via
`RemoveModuleInputOverride`. Static-switch reset writes the default-value
string back on the caller pin. Reuse the session-scoped
`FNiagaraSystemViewModel` cache so any open editor refreshes.

## Cross-ref

- `niagara.set_module_input` — the asymmetric counterpart.
- `niagara.set_static_switch` — static-switch reset shares the same
  caller-pin path described in the static-switch wiki note.
- `F-niagara-event-handler-simstage-authoring` (DONE) — view-model cache
  + dirty-management pattern.

## History
- `#1-no-reset-module-input` `OPEN` reporter — Confirmed `niagara.reset_module_input` / `niagara.clear_overrides` don't exist. `set_module_input` writes but never clears, leaving no path back to authored defaults except setting the override to a matching value (fragile) or remove+re-add (loses other valid overrides). Blocks the standard inner loop of override iteration. Proposes per-input reset plus bulk clear, both implemented via `RapidIterationParameters` removal and the static-switch caller-pin path. Reuses the established view-model cache.
- `#2-reviewed-and-confirmed` `OPEN` reviewer — Verified against source. `NiagaraEditHandler.cpp:1867` registers `niagara.set_module_input` with no clear/reset flag; grep across `Source/` finds zero `RemoveRapidIteration*` / `RemoveOverridePin` / `RemoveOverride` usages, and no `niagara.reset_*` / `niagara.clear_*` handlers exist. Workarounds (write-default-back, remove+re-add, `python.execute`) all match the report. On the API-shape question: paired-verb is the established house style in this surface (`add_parameter`/`remove_parameter`, `add_renderer`/`remove_renderer`, `add_module`/`remove_module`, `add_event_handler`/`remove_event_handler`, `add_data_interface`/`remove_data_interface`) — none overload a setter with a `clear` flag, and a `clear:true` mode would make the required `value` semantically meaningless. Proposed two-RPC shape is correct. Severity High is defensible given this is the inner-loop authoring asymmetry; Medium would also be honest since two workarounds exist, but left as filed.
- `#3-implemented-reset-override-pin` `IN-REVIEW` developer — Added `niagara.reset_module_input` and `niagara.clear_module_overrides` in `NiagaraEditHandler.cpp`. Implementation mirrors `set_module_input`'s **override-pin** write path (not rapid-iteration parameters as the ticket described): walks the module's input-map pin to its upstream `UNiagaraNodeParameterMapSet` and removes the named pin via inline reimpl of `RemoveNodesForStackFunctionInputOverridePin` (the engine helper is not NIAGARAEDITOR_API). Static-switch dispatch via `NiagaraStaticSwitch::FindByName` resets caller-pin DefaultValue. Helpers exposed via `NiagaraResetModuleInputHelpers.h` for test reach.
- `#4-fix-review-issues` `IN-REVIEW` developer — Fixed critical link-blocker: `FNiagaraStackGraphUtilities::GetStackFunctionOverrideNode` is not `NIAGARAEDITOR_API`; inlined the override-node walk in `NiagaraResetModuleInput::FindStackFunctionOverrideNode` using the existing `GetParameterMapPin` file-local helper and `FindObject<UClass>` for the `UNiagaraNodeParameterMapSet` class (which lives in a private engine header). Deleted the dead anonymous-namespace duplicates of `FindStackFunctionOverrideNode` and `RemoveOverridePinAndChainedNodes` (no call sites). Added `AddInfo` for deferred `RemoveOverridePinAndChainedNodes` integration coverage plus a `NiagaraStaticSwitch::FindByName(nullptr,...)` null-graph assertion to the test. Trimmed WHAT-comments from `RemoveOverridePinAndChainedNodes` to WHY-only.
- `#5-verify-static-switch-reset` `DONE` tester — Verified: `niagara.reset_module_input` on `/Game/UltraDynamicSky/Particles/Puddle_Ripple` entry `8CBE34B74E80AB6F1A9DE8B98442D668` input `Loop Behavior` returned `success:true`, `reset:true`, `kind:"switch"`, `previousValue:"NewEnumerator1"`; follow-up `niagara.inspect` showed that module's `Loop Behavior` static-switch `value` changed from `1` to default `0`.
- `#6-rerun-static-switch-reset` `DONE` tester — Verified: `niagara.inspect` on `/Game/UltraDynamicSky/Particles/Puddle_Ripple` showed entry `8CBE34B74E80AB6F1A9DE8B98442D668` input `Loop Behavior` at override value `1` with default `0`; `niagara.reset_module_input` returned `success:true`, `reset:true`, `kind:"switch"`, `previousValue:"NewEnumerator1"`, and follow-up `niagara.inspect` showed `Loop Behavior` value `0`. Restored the unsaved sampled switch value to `1` with `niagara.set_static_switch` and `save:false` after verification.
