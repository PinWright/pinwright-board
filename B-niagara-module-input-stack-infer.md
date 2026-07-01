---
id: B-niagara-module-input-stack-infer
title: "niagara.set_module_input wrongly rejects EmitterUpdate/ParticleSpawn modules with INVALID_STACK when scriptUsage omitted"
status: IN-REVIEW
severity: High
category: bug
tags: [niagara, stack, set-module-input, script-usage, invalid-stack, discoverability]
---

# `niagara.set_module_input` wrongly rejects modules with `INVALID_STACK` when the optional `scriptUsage` is omitted

`niagara.set_module_input` documents `scriptUsage` as **optional**
(`niagara.set_module_input.md` line 17:
`scriptUsage (string, optional): Script usage`). But for any module that
does not live in the `ParticleUpdateScript` stage (e.g. a `SpawnRate`
module in `EmitterUpdateScript`, or an `InitializeParticle` /
`AddVelocity` module in `ParticleSpawnScript`), omitting `scriptUsage`
causes the call to be **wrongly rejected** with:

```
[INVALID_STACK] Module '<entryId>' is not in a valid stack group.
```

The module IS in a valid stack group — it just isn't in the
`ParticleUpdateScript` stack that the handler silently defaults to. The
caller passed a complete, valid, documented argument set (`assetPath`,
`emitter`, `entryId`, `inputName`, `value`) with the optional
`scriptUsage` left off, exactly as the wiki permits, and the call failed.

## Root cause (source-confirmed)

`NiagaraEditHandler.cpp::ResolveStackOutputNode` (line 503) defaults the
usage to `ENiagaraScriptUsage::ParticleUpdateScript` when `scriptUsage`
is empty, then looks the module up in that stack's groups. A module that
actually lives in EmitterUpdate / ParticleSpawn / EmitterSpawn / SystemX
is not found there, so the `SourceGroupIndex` predicate
(`ApplyModuleMutation`, lines 660-668) fails and returns the
`INVALID_STACK` "not in a valid stack group" error.

The handler already knows how to infer the correct owning stack from the
resolved module node — `FindOwningStackOutputNode` (lines 527-570) walks
the module's downstream links to the owning `UNiagaraNodeOutput`. But
that inference is **gated to `SetModuleScript` only**
(`ApplyModuleMutation` line 645):

```cpp
if (Operation == ENiagaraEditOperation::SetModuleScript && Payload.Target.ScriptUsage.IsEmpty())
{
    OutputNode = FindOwningStackOutputNode(Target);
}
```

So `SetModuleInput` (and `SetStackEnabled`) never get the inference and
fall through to the ParticleUpdate-defaulting `ResolveStackOutputNode`.

This is the **same defect** that `niagara.set_module_script` had —
filed/verified in `F-niagara-set-module-script` history `#6-reverify-invalid-stack`
(same `INVALID_STACK` "not in a valid stack group" wording) and fixed in
`#7-infer-owning-stack` by calling `FindOwningStackOutputNode` when
`scriptUsage` is omitted. The fix simply was not extended to the
`SetModuleInput` operation path that shares `ApplyModuleMutation`.

## Why it matters

This is the common authoring path. To set a spawn rate, a particle
lifetime, or an upward velocity on a freshly-built emitter, the caller
must override inputs on EmitterUpdate and ParticleSpawn modules — none of
which are in ParticleUpdate. The documented optional-`scriptUsage` form
fails for all of them, and the error text ("not in a valid stack group")
actively misdirects: it implies the module/entryId is malformed or
orphaned, not that an optional param was defaulted to the wrong stage.
The author has to read the handler source to discover that `scriptUsage`
is effectively required outside ParticleUpdate.

## Repro (verbatim, replay-confirmed)

1. `niagara.create_system` `{name:"NS_OracleReplay", savePath:"/Game/FuzzVFX"}` -> ok
2. `niagara.create_emitter` `{name:"E_OracleReplay", savePath:"/Game/FuzzVFX"}` -> ok
3. `niagara.add_emitter` `{systemPath:".../NS_OracleReplay.NS_OracleReplay", emitterPath:".../E_OracleReplay.E_OracleReplay", name:"OracleWisp"}` -> ok
4. `niagara.add_module` `{assetPath:".../NS_OracleReplay.NS_OracleReplay", emitter:"OracleWisp", modulePath:"/Niagara/Modules/Emitter/SpawnRate.SpawnRate", scriptUsage:"EmitterUpdateScript"}` -> `nodeId:"6DAB3F964202631D10C0EABF9B6A0F5D"`
5. **FAIL** `niagara.set_module_input` `{assetPath:".../NS_OracleReplay.NS_OracleReplay", emitter:"OracleWisp", entryId:"6DAB3F964202631D10C0EABF9B6A0F5D", inputName:"SpawnRate", value:20}` (no `scriptUsage`)
   -> `[INVALID_STACK] Module '6DAB3F964202631D10C0EABF9B6A0F5D' is not in a valid stack group.`
6. **PASS** same call **plus** `scriptUsage:"EmitterUpdateScript"`
   -> `{success:true, operation:"set_module_input", entryId:"6DAB3F964202631D10C0EABF9B6A0F5D", inputName:"SpawnRate", pinId:"E1D0D517403E778B3A58A496BA1656FE", index:0}`

The only difference between the failing and passing call is the presence
of the optional `scriptUsage`.

**Workaround:** always pass `scriptUsage` matching the module's stage
(`EmitterUpdateScript` / `ParticleSpawnScript` / etc.) on
`set_module_input`.

**Fix:** extend the `FindOwningStackOutputNode` inference at
`ApplyModuleMutation` line 645 to also cover `SetModuleInput` (and
`SetStackEnabled`) when `Payload.Target.ScriptUsage.IsEmpty()`, mirroring
the `SetModuleScript` fix from `F-niagara-set-module-script`
`#7-infer-owning-stack`. Either that, or make the doc/error honest:
if the optional param is genuinely required outside ParticleUpdate, the
error should say "module is in the EmitterUpdate stack; pass
scriptUsage:EmitterUpdateScript" rather than "not in a valid stack group".

## Cross-ref

- `F-niagara-set-module-script` (DONE) — same `INVALID_STACK` symptom on
  the sibling RPC; `#7-infer-owning-stack` is the exact fix pattern to
  reuse here. The shared helper `FindOwningStackOutputNode` already exists.

## History
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed via mcp__editor-automation__call: `niagara.set_module_input` on a `SpawnRate` module in the `EmitterUpdateScript` stack returns `[INVALID_STACK] Module '6DAB3F964202631D10C0EABF9B6A0F5D' is not in a valid stack group.` when the documented-optional `scriptUsage` is omitted, but `success:true` when `scriptUsage:"EmitterUpdateScript"` is added (only difference). Source-confirmed: `NiagaraEditHandler.cpp` defaults usage to `ParticleUpdateScript` in `ResolveStackOutputNode` (line 503) and only infers the owning stack via `FindOwningStackOutputNode` for `SetModuleScript` (line 645), so `SetModuleInput` mis-resolves the stack and the group-index check (lines 660-668) wrongly reports INVALID_STACK. Same defect/fix as `F-niagara-set-module-script` `#7-infer-owning-stack`; the inference was never extended to the SetModuleInput path. Wiki marks `scriptUsage` as optional (`niagara.set_module_input.md` line 17) with no hint it is effectively required outside ParticleUpdate.
- `#2-infer-owning-stack` `IN-REVIEW` developer — Fixed by widening the `FindOwningStackOutputNode` inference gate in `ApplyModuleMutation` (`Source/EditorAutomationRpcGateway/Private/Handlers/Niagara/NiagaraEditHandler.cpp`, the former line-645 `Operation == SetModuleScript && ScriptUsage.IsEmpty()` guard) to a `bCanInferOwningStack` predicate covering `SetModuleScript`, `SetModuleInput`, and `SetStackEnabled` when `Payload.Target.ScriptUsage.IsEmpty()`. This mirrors the proven `F-niagara-set-module-script` `#7-infer-owning-stack` fix and reuses the existing `FindOwningStackOutputNode` helper, so a non-ParticleUpdate module (e.g. `SpawnRate` in `EmitterUpdateScript`) now resolves to its real owning stack output instead of defaulting to ParticleUpdate and failing the group-index predicate with INVALID_STACK. Explicit-`scriptUsage` behavior is unchanged. Regression test added: `Source/EditorAutomationRpcGateway/Private/Tests/Niagara/TestNiagaraSetModuleInput.cpp` — dispatcher-path tests `EditorAutomationRpcGateway.niagara.set_module_input.InfersOwningEmitterUpdateStack` and `EditorAutomationRpcGateway.niagara.set_stack_enabled.InfersOwningEmitterUpdateStack` add a SpawnRate module to the EmitterUpdate stack of a duplicated-fixture system, invoke the RPC with `scriptUsage` omitted, and assert the result is success and NOT `INVALID_STACK` (each would fail if the inference widening were reverted), mirroring the sibling test in `TestNiagaraSetModuleScript.cpp`. Plus a `set_module_input` registration test.
