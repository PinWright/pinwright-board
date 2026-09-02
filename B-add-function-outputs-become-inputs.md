---
id: B-add-function-outputs-become-inputs
title: "blueprint.add_function creates every declared `outputs` pin as an INPUT — no return value, no FunctionResult node; every Blueprint-Interface getter it authors is silently unusable, and the response echoes `outputs` as if it worked"
status: OPEN
severity: High
category: bug
tags: [blueprint, add_function, outputs, return-value, blueprint-interface, silent-false-success, bpir, cross-stream-contract]
---

# `blueprint.add_function` turns declared outputs into inputs

## Symptom

`blueprint.add_function` with an `outputs` array creates the pin on the **input**
side of the function entry. The function ends up with no return value and no
`UK2Node_FunctionResult`, so it can never return anything — while the RPC response
reports `success: true` and echoes back `outputs: [{name, type}]`, which reads as
confirmation that the output was created.

Reproduced on both a Blueprint **Interface** and a plain **Actor** Blueprint, so it
is not interface-specific.

## Repro

```
call("blueprint.create",       {name:"BP_WeaponStub_PlayerTest", savePath:"/Game/FPS/Player", parentClass:"Actor"})
call("blueprint.add_function", {path:"/Game/FPS/Player/BP_WeaponStub_PlayerTest",
                                functionName:"GetADSFOV",
                                outputs:[{name:"FOV", type:"float"}],
                                isPublic:true})
-> {"success":true, "functionName":"GetADSFOV", "outputs":[{"name":"FOV","type":"float"}], "saved":true}
```

Two independent readers disagree with that response:

```
call("system.inspect.inspect_class", {className:"/Game/FPS/Player/BP_WeaponStub_PlayerTest.BP_WeaponStub_PlayerTest_C"})
-> {"name":"GetADSFOV","returnType":"void",
    "params":[{"name":"FOV","cppType":"float","direction":"in"}],
    "flags":["BlueprintCallable","BlueprintEvent","Public"]}

call("blueprint.decompile_function", {assetPath:"...", functionName:"GetADSFOV"})
-> "entry function GetADSFOV(float FOV) @(0, 0) {\n}"
```

The BPIR decompiler is the decisive one: a real output would decompile as
`entry function GetADSFOV() -> float`. It decompiles as a **parameter**.

Same on an interface:

```
call("blueprint.create",       {name:"BPI_RecoilReceiver", savePath:"/Game/FPS/Player", blueprintType:"interface"})
call("blueprint.add_function", {path:"/Game/FPS/Player/BPI_RecoilReceiver",
                                functionName:"PW_TempOutProbe", outputs:[{name:"Val", type:"float"}]})
-> inspect_class: {"name":"PW_TempOutProbe","returnType":"void",
                   "params":[{"name":"Val","cppType":"float","direction":"in"}]}
```

## Why it matters here (and how it was found)

It silently destroyed a **cross-stream contract**. `Docs/fps/INTERFACES.md` in this
project specifies `BPI_HUDSource` as four getters —
`GetHealth() -> float 0..1`, `GetAmmo() -> (Mag, Reserve)`, `GetSpread() -> float`,
`GetWeaponName() -> Text`. Another agent authored that interface through
`blueprint.add_function` and it landed on disk as four **input-only
`BlueprintEvent`s**:

```
call("blueprint.decompile_function", {assetPath:"/Game/FPS/Player/BPI_HUDSource", functionName:"GetHealth"})
-> "entry function GetHealth(float Health01) @(0, 0) {\n}"
```

A HUD widget calling `GetHealth` on the possessed pawn would *pass a float in* and
get nothing back. Nothing in the authoring session reported a problem: the call
succeeded, the echo said `outputs`, and the asset saved. The break surfaces only at
integration, in a different stream, hours later.

## Expected

- Pins declared in `outputs` must be created as output pins on the
  `UK2Node_FunctionEntry` (i.e. `CPF_OutParm`, with a `UK2Node_FunctionResult`
  present) so the function actually returns a value — the shape
  `blueprint.decompile_function` would emit as `-> float`.
- If for some reason an output cannot be created, the call must **fail**, not echo
  the request array back as if it had succeeded. The response's `outputs` field is
  currently sourced from the request, not from the created pins — the same
  false-echo pattern already filed for `blueprint.add_event` in
  `B-add-event-echoes-failed-pins` (that ticket is about *failed* pins being echoed;
  this one is about pins created on the **wrong side**, which no existing ticket
  covers).
- A read-back that disagrees with the writer is the second half of the problem:
  `system.inspect.inspect_class` and `blueprint.inspect` both list these under
  `params`/`inputs` with `direction: "in"`, which is *correct* reporting of a wrong
  asset — but `blueprint.inspect`'s label `"inputs"` makes the damage easy to
  mistake for a display quirk. Worth a note on the method page either way.

## Wiki

`Saved/PinWright/wiki/blueprint.add_function.md` documents
`outputs` — *"Array of {name, type} pin definitions for outputs"* — with no caveat,
and `blueprint.md` advertises the verb as "Create a new UFunction graph on a Blueprint
with caller-specified input/output pins". Nothing warns that outputs do not work.

## Workaround

Author the function with `blueprint.compile_bpir` instead, declaring the signature in
the BPIR entry line, which does support returns:

```
entry function GetADSFOV() -> float {
    return $ADSFOV
}
```

BPIR's `entry function ... -> (type Name, ...)` form also covers multi-output
(`GetAmmo() -> (int Mag, int Reserve)`), which is what the contract needs.

severity rationale: impact=wrong-output + silent-false-success on a cross-stream
contract (the defect is invisible in the authoring session and surfaces in a
different agent's stream) x reach=every Blueprint getter or interface accessor
authored through the typed surface -> High

## History
- `#1-filed` `OPEN` reporter — Found while building `BP_FPSCharacter` for the FPS PLAYER stream on UE 5.8 / EAContentExamples58. `blueprint.create {blueprintType:"interface"}` reported `/Game/FPS/Player/BPI_HUDSource` already existed, and `blueprint.inspect` listed its four contract getters with their values under `"inputs"` — which looked at first like another agent having authored the interface backwards. Probing ruled that out: I created a fresh `BPI_RecoilReceiver` and added `PW_TempOutProbe` with `outputs:[{name:"Val",type:"float"}]`, and `system.inspect.inspect_class` returned `direction:"in"`; the same probe on a plain Actor BP (`BP_WeaponStub_PlayerTest`, `GetADSFOV`, `outputs:[{name:"FOV",type:"float"}]`) also returned `direction:"in"`, so it is the verb, not the asset and not interfaces. `blueprint.decompile_function` confirms it independently and unambiguously: `entry function GetADSFOV(float FOV)` where a real output would read `entry function GetADSFOV() -> float`. All three calls returned `success:true` with `outputs` echoed back from the request. Consequence in this project: `Docs/fps/INTERFACES.md` specifies `BPI_HUDSource` as four *getters*, and what is on disk cannot return a value, so the UI stream's HUD would silently read nothing. Workaround adopted for my stream: declare every returning function through `blueprint.compile_bpir` (`entry function GetAmmo() -> (int Mag, int Reserve)`), never through `blueprint.add_function`'s `outputs`. Related but distinct: `B-add-event-echoes-failed-pins` (pins that failed to create, echoed as created) and `E-create-rpc-function-no-param-slot` (which assumes `blueprint.add_function`'s `outputs` works and proposes copying it into `networking.create_rpc_function` — that proposal would propagate this defect).
