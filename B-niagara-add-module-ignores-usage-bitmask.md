---
id: B-niagara-add-module-ignores-usage-bitmask
title: "niagara.add_module accepts a module into a stack its ModuleUsageBitmask excludes (e.g. SpawnRate into ParticleUpdate), and an omitted scriptUsage defaults to ParticleUpdate, so an emitter-only module lands in a particle stack silently"
status: IN-REVIEW
severity: Medium
category: bug
tags: [gap-analysis-2026-09-28, niagara, add-module, script-usage, module-usage-bitmask, validation]
encounters: 1
lastSeen: 2026-09-29T18:18:40Z
---

# `niagara.add_module` does not check the module's usage bitmask against the target stack

The AddModule branch of `ApplyModuleMutation`
(`Source/PinWright/Private/Handlers/Niagara/NiagaraEditHandler.cpp:986-1015`) resolves the
output node, loads the module script and calls `FNiagaraStackGraphUtilities::AddScriptModuleToStack`
with no stage check. `niagara.set_module_script` in the same function already refuses the same
mismatch with `INCOMPATIBLE_STACK_GROUP` via
`UNiagaraScript::IsSupportedUsageContextForBitmask(ModuleUsageBitmask, StackUsage)` (`:1435-1442`),
and the Niagara editor's add-module menu filters by the same bitmask, so `add_module` is the one
authoring path that lets an incompatible module in.

It compounds with the default: on an emitter target, `TryResolveStackScriptUsage` (`:415-440`)
falls back to `ParticleUpdateScript` when `scriptUsage` is absent or unrecognised. So
`niagara.add_module {modulePath: "/Niagara/Modules/Emitter/SpawnRate.SpawnRate"}` with no
`scriptUsage` puts SpawnRate, whose `ModuleUsageBitmask` is `1536` (EmitterSpawn | EmitterUpdate
only, read from `C:\UE_5.8\Engine\Plugins\FX\Niagara\Content\Modules\Emitter\SpawnRate.uasset`),
into the ParticleUpdate stack and reports success.

Found while fixing `PinWright.niagara.move_module.RelayoutsNodePosY`, whose test fixture places
SpawnRate in ParticleUpdate through the engine helper directly (see
`B-niagara-tests-spawnrate-wrong-path` `#3-move-module-fixture-chain`). Not reproduced live
through the verb, and whether a later `compile:true` reports the misplaced module was not checked.

**Fix:** in the AddModule branch, after loading `ModuleScript`, refuse with
`ERR_INCOMPATIBLE_STACK_GROUP` when `GetLatestScriptData()->ModuleUsageBitmask` does not support
`OutputNode->GetUsage()`, naming the stages the module does support. Add a failure-direction test
(SpawnRate into ParticleUpdate refused, into EmitterUpdate accepted) and document the error on the
`niagara.add_module` wiki page.

## History
- `#1-add-module-no-stage-check` `OPEN` reporter — `niagara.add_module` never checks `ModuleUsageBitmask` against the target stack (`NiagaraEditHandler.cpp:986-1015`), unlike `niagara.set_module_script` (`:1435-1442`), and an omitted `scriptUsage` defaults to ParticleUpdate (`:439`), so an emitter-only module such as SpawnRate (bitmask 1536) lands in ParticleUpdate with a success result. Static reading only; not reproduced through the verb.
- `#2-bitmask-gate-and-derived-usage` `IN-REVIEW` developer — `NiagaraEditHandler.cpp`: new shared `CheckModuleUsageAllowsStack(Script, StackUsage)`, used by both `add_module` and `set_module_script` (which previously had its own inline check). It refuses `INCOMPATIBLE_STACK_GROUP` with a message that names the module path, the requested stack and the allowed stacks from `GetSupportedUsageContextsForBitmask`. Omitted `scriptUsage`: `FindOnlyAllowedStackOutputNode` places the module in the only stack of the target graph that its bitmask allows. When several stacks qualify, the ParticleUpdate default stays and the gate refuses it if disallowed. So a wrong stack now fails loudly and never lands silently. `scriptUsage` was not made required, although rpc-design §3 prefers that when no safe default exists. With the gate in place the default can no longer be unsafe: it either names an allowed stack or is refused. Making the parameter required would break every existing ParticleUpdate caller without adding safety. An explicit `scriptUsage` is still resolved before the module loads, so `OUTPUT_NODE_NOT_FOUND` keeps precedence over `MODULE_SCRIPT_NOT_FOUND`, which `Assets.Niagara.Edit` `ModuleSurfacesStructuredErrors` relies on. New test `PinWright.niagara.add_module.UsageBitmaskGatesTargetStack` (`Tests/Niagara/TestNiagaraAddModuleUsage.cpp`) covers three cases. SpawnRate with explicit ParticleUpdate is refused with the code and with module, stack and allowed names in the message. SpawnRate with `scriptUsage` omitted is refused. The ParticleUpdate chain is unchanged after both refusals, and SpawnRate into EmitterUpdate succeeds and grows that stack by one. Wiki `docs/wiki-src/niagara.md` has a new `### niagara.add_module` section; CHANGELOG has a Changed line. Compiles with `-SingleFile` on 5.8; not run.
