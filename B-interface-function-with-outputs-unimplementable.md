---
id: B-interface-function-with-outputs-unimplementable
title: "A Blueprint Interface function that returns values cannot be implemented: compile_bpir counts its outputs as inputs and rejects it, and add_function override:true silently makes a K2Node_Event instead of a function graph"
status: IN-REVIEW
severity: High
category: bug
tags: [blueprint, add_function, compile_bpir, blueprint-interface, override, signature-check, silent-wrong-node]
encounters: 1
lastSeen: 2026-09-02T19:40:00Z
---

# A BP Interface function with return values has no working implementation route

Both routes to implement a Blueprint Interface function that declares outputs fail, one loudly
with a wrong diagnosis and one silently with the wrong node type.

## Repro

Author an interface and one implementer (UE 5.8, `EAContentExamples58`):

```
blueprint.create {name:"BPI_AIWeapon", savePath:"/Game/FPS/AI", blueprintType:"interface"}
blueprint.add_function {path:"/Game/FPS/AI/BPI_AIWeapon", functionName:"GetAmmoState",
  outputs:[{name:"AmmoInMag",type:"int"},{name:"AmmoReserve",type:"int"},{name:"bReloading",type:"bool"}],
  isPublic:true}                                                    -> success
blueprint.add_function {path:"/Game/FPS/AI/BPI_AIWeapon", functionName:"GetMuzzleLocation",
  outputs:[{name:"Muzzle",type:"vector"}], isPublic:true}           -> success
blueprint.compile {path:"/Game/FPS/AI/BPI_AIWeapon"}                -> compiled:true
blueprint.add_interface {path:"/Game/FPS/AI/BP_AIWeaponStub",
  interfaceClass:"/Game/FPS/AI/BPI_AIWeapon.BPI_AIWeapon_C"}        -> success, interfaceGraphs:[]
```

### Route 1 — `compile_bpir`: rejected, and the diagnosis is wrong

```
blueprint.compile_bpir {assetPath:"/Game/FPS/AI/BP_AIWeaponStub", code:
"entry function GetAmmoState() -> (int AmmoInMag, int AmmoReserve, bool bReloading) {
    return (AmmoInMag: $AmmoInMag, AmmoReserve: $AmmoReserve, bReloading: $bReloading)
}

entry function GetMuzzleLocation() -> (vector Muzzle) {
    %m = call GetMuzzleWorld(Target: self)
    return (Muzzle: %m)
}"}
```

```
[COMPILE_FAILED] Line -1: Function 'GetAmmoState' matches an overridable parent function, but the
authored signature does not match the parent override signature: Input count mismatch: expected 0,
actual 3; Line -1: Function 'GetMuzzleLocation' matches an overridable parent function, but the
authored signature does not match the parent override signature: Input count mismatch: expected 0,
actual 1
```

The counts give the bug away. `GetAmmoState` declares **zero inputs and three outputs**;
`GetMuzzleLocation` declares **zero inputs and one output**. "actual 3" and "actual 1" are the
**output** counts, so the signature comparator is putting the authored return values into the input
side and then comparing them against the parent's (correctly) empty input list. The parent's own
outputs are never compared at all. The check therefore rejects exactly the signature it should
accept, and no phrasing of the BPIR text can satisfy it — declaring the values as inputs would
compile a different, wrong function.

`Line -1` is a second, smaller defect in the same message: the error carries no usable source
position.

### Route 2 — `add_function {override:true}`: succeeds, produces the wrong node kind

```
blueprint.add_function {path:"/Game/FPS/AI/BP_AIWeaponStub", functionName:"GetAmmoState", override:true}
-> {"success":true, "override":true, "eventOverride":true, "compiled":true, "status":"UpToDate"}
```

`eventOverride:true` is the tell: it created a `K2Node_Event`, not a function graph. `blueprint.get`
afterwards lists `GetAmmoState` under `events` with `"eventType":"K2Node_Event"`, and it is absent
from `functions`. That node cannot return anything — in the Blueprint editor an interface function
**with** return values is implementable only as a function graph; only a `void` interface function
becomes an event. So the call reports success, the Blueprint compiles clean, and the interface
function is silently left unimplemented: any caller of `GetAmmoState` through the interface gets
default-constructed values at runtime with no error anywhere.

Nothing in the response distinguishes "I made the override you asked for" from "I made a node that
cannot possibly implement this". `compiled:true` / `status:"UpToDate"` actively reassure the caller.

## Expected

- `compile_bpir`'s parent-signature comparator should compare authored **outputs** against the
  parent's outputs and authored **inputs** against the parent's inputs, so a correctly-declared
  interface function implementation compiles. (`entry override GetAmmoState() -> (...)` should work
  too, if the explicit form is preferred for this case.)
- `add_function {override:true}` should choose the node kind from the parent function's signature:
  a function graph when it has return values, an event only when it is `void`. When it cannot make
  a valid implementation it should fail with a typed error, never return `success:true` with
  `eventOverride:true` for a function that has outputs.

## Impact

Blueprint Interfaces are the only cross-class call mechanism in a Blueprint-only project, and every
non-trivial one has at least one getter. In this session the interface was the whole point — the AI
stream needs to call a weapon it cannot cast to, because the weapon class is owned by another
stream and does not exist on disk yet.

**Workaround:** drop every return value from the interface and keep it `void`-only (those implement
fine as events through `entry event <Name>(...)`), then read state back by casting to the concrete
class — which defeats the reason for having the interface. The bogus `K2Node_Event` left behind by
route 2 must also be cleaned up with `blueprint.remove_event`; `blueprint.remove_function` does not
see it.

severity rationale: impact=silent false-success on a normal path (route 2 reports success and a
clean compile for a node that cannot implement the function) x reach=common (any Blueprint
Interface with a getter) -> High

## Fix

The observed input counts came from the upstream `blueprint.add_function` output-authoring defect, not from BPIR reversing signature directions: current BPIR and reflected-function code already classify inputs and outputs separately. This fix makes explicit override matching an order-independent, case-insensitive name-and-exact-`FEdGraphPinType` bijection within each direction, finds function graphs owned by `ImplementedInterfaces`, reuses those graphs in `blueprint.add_function` and BPIR, and makes replace mode delete only their body nodes while preserving entry/result terminators and interface membership. BPIR now also lays out every valid block's graph context, including reused interface graphs, without treating those graphs as rollback-owned. Void overrides remain eligible for `K2Node_Event`, with handler-created events now routed through `FKismetEditorUtilities::AddDefaultEventNode`.

The verifier follow-up extends `BlueprintGraphSnapshot::Capture` to every graph returned by `UBlueprint::GetAllGraphs`, so interface-owned, delegate, child, and extension graphs participate in the same node pre-image rollback as ordinary function/macro/ubergraphs. `PinWright.blueprint.compile_bpir.InterfaceGraphRollbackAfterFinalCompileFailure` invokes the real handler, mutates an interface-owned graph, forces the subsequent whole-Blueprint compile to fail, and requires graph identity, node count, serialized node state, and the original body sentinel to be restored.

Files changed:
- `Source/PinWright/Private/Handlers/Blueprint/BlueprintHandlerUtils.h`
- `Source/PinWright/Private/Handlers/Blueprint/BlueprintHandlerUtils.cpp`
- `Source/PinWright/Private/Handlers/Blueprint/BlueprintFunctionHandler.cpp`
- `Source/PinWright/Private/Compiler/BpirCompiler.cpp`
- `Source/PinWright/Private/Utils/BlueprintGraphSnapshot.h`
- `Source/PinWright/Private/Utils/BlueprintGraphSnapshot.cpp`
- `Source/PinWright/Private/Tests/Blueprint/TestBlueprintInterfaceFunctionOutputs.cpp`
- `Source/PinWright/Private/Tests/Blueprint/TestBlueprintOverrideSignature.cpp`
- `Source/PinWright/Private/Tests/Bpir/TestBpirInterfaceOutputOverride.cpp`
- `Source/PinWright/Private/Tests/TestUtils.h`
- `Docs/wiki-src/blueprint.md`

Tests added:
- `PinWright.blueprint.interface.OutputOverrideUsesFunctionGraph`
- `PinWright.blueprint.override.SignatureMatchesByNameAndDirection`
- `PinWright.bpir.compiler.integration.InterfaceNamedOutputsOverride`
- `PinWright.blueprint.compile_bpir.InterfaceGraphRollbackAfterFinalCompileFailure`

Deliberately unchanged: BPIR output descriptors remain outputs; void interface functions may remain event overrides; existing malformed interface assets are not migrated; `blueprint.add_interface` is unchanged; and the separate `Line -1` diagnostic issue is not addressed.

## History
- `#1-filed` `OPEN` reporter — Hit while building the FPS AI stream's weapon abstraction (`BPI_AIWeapon`, implemented by `/Game/FPS/AI/BP_AIWeaponStub`, to be implemented later by the WEAPONS stream's `BP_WeaponBase`). Two `void` interface events (`StartFire`, `StopFire`, `Reload`, `SetFireTarget`) implemented cleanly through `entry event <Name>(...)` in `compile_bpir` — only the two functions with return values failed. Route 1's error text is quoted verbatim above; the "expected 0, actual N" numbers match the authored **output** counts (3 and 1) exactly, which is what identifies the comparator as reading outputs into the input slot. Route 2's `eventOverride:true` plus a `blueprint.get` readback showing `GetAmmoState` under `events` (`eventType:"K2Node_Event"`) and absent from `functions` is the evidence for the silent-wrong-node half. No plugin source was read; the diagnosis is from the response payloads and the counts. Worked around by deleting both getters from the interface and reading the weapon's ammo state through a concrete-class cast instead, which reintroduces exactly the coupling the interface existed to remove.
- `#2-interface-graph-reuse` `IN-REVIEW` developer — Reused interface-owned function graphs for value-returning overrides, matched explicit signatures by per-direction name/type bijection, documented routing, and added `PinWright.blueprint.interface.OutputOverrideUsesFunctionGraph`, `PinWright.blueprint.override.SignatureMatchesByNameAndDirection`, and `PinWright.bpir.compiler.integration.InterfaceNamedOutputsOverride`.
- `#3-interface-rollback-coverage` `IN-REVIEW` developer — Included implemented-interface graphs in BPIR snapshot rollback and added a full-handler final-compile-failure regression that requires exact interface graph restoration.
