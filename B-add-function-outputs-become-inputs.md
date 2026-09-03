---
id: B-add-function-outputs-become-inputs
title: "blueprint.add_function creates every declared `outputs` pin as an INPUT — no return value, no FunctionResult node; every Blueprint-Interface getter it authors is silently unusable, and the response echoes `outputs` as if it worked"
status: IN-REVIEW
severity: High
category: bug
tags: [blueprint, add_function, outputs, return-value, blueprint-interface, silent-false-success, bpir, cross-stream-contract]
encounters: 2
lastSeen: 2026-09-02T20:05:00Z
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

## Fix

The defect was broader than the ticket's proposed direction: both logical directions used the wrong UE terminator direction, a fresh graph had no result terminator, and the handler wrote the request arrays into its registry and response instead of reading the created graph. `Source/PinWright/Private/Handlers/Blueprint/BlueprintFunctionHandler.cpp` now creates or reuses `UK2Node_FunctionResult`, authors logical inputs as physical `EGPD_Output` entry pins and logical outputs as physical `EGPD_Input` result pins, reconstructs the nodes, requires a one-to-one name/type match against their surviving non-exec physical pins, rolls back collisions and duplicate declarations, and derives function and macro signatures from those physical pins. Both handlers compile through `CompileBlueprintWithDiagnostics`, set `success` from the measured `compiled` result, expose `status`/`errors`/`warnings`, and attempt saving only after a successful compile. The creation transactions close before structural compilation and saving. `Source/PinWright/Private/Tests/Blueprint/TestBlueprintFunctionOutputPins.cpp` adds `PinWright.blueprint.add_function.SignaturePinsAndResponseMatchGraph` and `PinWright.blueprint.add_macro.ResponsePinsMatchGraph`, including `then`/`execute` collision and duplicate-name rollback cases; the function test also requires `compiled:true` and verifies the generated `UFunction` parameter names, reflected types, and `CPF_Parm`/`CPF_OutParm`/`CPF_ReturnParm` direction flags. `Docs/wiki-src/blueprint.md` documents the readback and compile-result contracts. Macro tunnel placement was deliberately not changed because it was already correct, and existing malformed assets were not migrated because that needs an explicit asset scope and live verification.

## History
- `#1-filed` `OPEN` reporter — Found while building `BP_FPSCharacter` for the FPS PLAYER stream on UE 5.8 / EAContentExamples58. `blueprint.create {blueprintType:"interface"}` reported `/Game/FPS/Player/BPI_HUDSource` already existed, and `blueprint.inspect` listed its four contract getters with their values under `"inputs"` — which looked at first like another agent having authored the interface backwards. Probing ruled that out: I created a fresh `BPI_RecoilReceiver` and added `PW_TempOutProbe` with `outputs:[{name:"Val",type:"float"}]`, and `system.inspect.inspect_class` returned `direction:"in"`; the same probe on a plain Actor BP (`BP_WeaponStub_PlayerTest`, `GetADSFOV`, `outputs:[{name:"FOV",type:"float"}]`) also returned `direction:"in"`, so it is the verb, not the asset and not interfaces. `blueprint.decompile_function` confirms it independently and unambiguously: `entry function GetADSFOV(float FOV)` where a real output would read `entry function GetADSFOV() -> float`. All three calls returned `success:true` with `outputs` echoed back from the request. Consequence in this project: `Docs/fps/INTERFACES.md` specifies `BPI_HUDSource` as four *getters*, and what is on disk cannot return a value, so the UI stream's HUD would silently read nothing. Workaround adopted for my stream: declare every returning function through `blueprint.compile_bpir` (`entry function GetAmmo() -> (int Mag, int Reserve)`), never through `blueprint.add_function`'s `outputs`. Related but distinct: `B-add-event-echoes-failed-pins` (pins that failed to create, echoed as created) and `E-create-rpc-function-no-param-slot` (which assumes `blueprint.add_function`'s `outputs` works and proposes copying it into `networking.create_rpc_function` — that proposal would propagate this defect).
- `#2-arity-and-name-independent-plus-inconsistent-reader` `OPEN` reporter — Additional evidence from a second stream (UI) on the same host, EAContentExamples58 / UE 5.8. Two things this adds. (a) **Neither arity nor pin name changes the outcome.** Fresh throwaway `/Game/FPS/UI/Test/BP_PWProbe_UI` (Actor), three `blueprint.add_function` calls, each answering `success:true` with `outputs` echoed verbatim; `blueprint.decompile_function` on each: `outputs:[{name:"Foo",type:"float"}]` -> `entry function P1_OneNamed(float Foo) @(0, 0) {}`; `outputs:[{name:"ReturnValue",type:"float"}]` -> `entry function P2_OneReturnValue(float ReturnValue) @(0, 0) {}`; `outputs:[{name:"A",type:"float"},{name:"B",type:"int"}]` -> `entry function P3_TwoOutputs(float A, int B) @(0, 0) {}`. So a multi-output declaration is broken exactly like a single-output one, and naming the pin `ReturnValue` (UE's own default return name) does not route it to the output side either — worth stating explicitly because both are the obvious things a caller tries next. (b) **`blueprint.inspect` is an unreliable check for this bug.** On `/Game/FPS/Player/BPI_HUDSource` it simultaneously reported `GetAmmo` with `"outputs":[{"name":"Mag"},{"name":"Reserve"}]` and `graphs` `nodeCount:2`, and `GetHealth`/`GetSpread`/`GetWeaponName` with `"inputs":[{"name":"ReturnValue"}]` and `nodeCount:1` — same asset, same verb that authored them, opposite verdicts. A caller who verifies with `blueprint.inspect` and happens to look at a function it reports correctly will conclude the verb works. `blueprint.decompile_function` was consistent across all six functions and is the check to recommend. Cost in this stream: `BPI_HUDSource` had to be rebuilt and the implementing pawn's interface removed and re-added, and `blueprint.compile_bpir` rejected the implementation with `Input count mismatch: expected 0, actual 1` — a diagnosis that points at the caller's own BPIR rather than at the malformed interface, which is how the bug hides. Related: `B-interface-function-with-outputs-unimplementable` (the downstream half) and `B-bpir-single-named-output-forced-returnvalue` (a separate BPIR-side naming defect found on the same task).
- `#3-function-signature-readback` `IN-REVIEW` developer — Corrected function terminator creation and physical pin directions, made function/macro signature responses graph-derived, documented the contract, and added `PinWright.blueprint.add_function.SignaturePinsAndResponseMatchGraph` plus `PinWright.blueprint.add_macro.ResponsePinsMatchGraph`.
- `#4-compile-signature-proof` `IN-REVIEW` developer — Exposed measured Blueprint compile diagnostics from function and macro creation, and strengthened their tests to require `compiled:true` plus the generated `UFunction` parameter directions and types.
