---
id: B-bpir-single-named-output-forced-returnvalue
title: "BPIR drops the name of a single named output: `-> (float Health01)` compiles to `ReturnValue`, so no Blueprint Interface function with one named return can be implemented"
status: IN-REVIEW
severity: High
category: bug
tags: [bpir, compile_bpir, function, entry-function, entry-override, interface, return-value, output-name]
encounters: 1
lastSeen: 2026-09-02T19:52:00Z
---

# BPIR renames a single named output to `ReturnValue`, blocking Blueprint Interface implementation

## Repro

Interface `/Game/FPS/Player/BPI_HUDSource` (Blueprint Interface) declares four functions, created
with `blueprint.add_function`:

```
blueprint.add_function {path:"/Game/FPS/Player/BPI_HUDSource", functionName:"GetHealth",
                        outputs:[{name:"Health01", type:"float"}], isPublic:true}
  -> success, outputs:[{name:"Health01",type:"float"}]
```

`blueprint.add_interface` on the implementing Pawn creates the four graphs
(`interfaceGraphs:["GetAmmo","GetWeaponName","GetSpread","GetHealth"]`). Implementing their bodies
with BPIR fails:

```
blueprint.compile_bpir {assetPath:"/Game/FPS/UI/Test/BP_HUDTestPawn", code:
  "entry function GetHealth() -> (float Health01) { return (Health01: $Health01) }
   entry function GetAmmo() -> (int Mag, int Reserve) { return (Mag: $MagAmmo, Reserve: $ReserveAmmo) }
   entry function GetSpread() -> (float SpreadDegrees) { return (SpreadDegrees: $Spread) }
   entry function GetWeaponName() -> (text WeaponName) { return (WeaponName: $WeaponNameVar) }"}

-> [COMPILE_FAILED]
   Function 'GetHealth' matches an overridable parent function, but the authored signature does not
     match the parent override signature: Output name mismatch at index 0:
     expected 'Health01', actual 'ReturnValue';
   Function 'GetSpread' ... expected 'SpreadDegrees', actual 'ReturnValue';
   Function 'GetWeaponName' ... expected 'WeaponName', actual 'ReturnValue'
```

**`GetAmmo` — the two-output function — is not in the error list.** Only the three single-output
functions fail, and every one of them fails the same way: the authored name is discarded and
`ReturnValue` appears in its place.

`entry override` fails identically:

```
blueprint.compile_bpir {assetPath:"/Game/FPS/UI/Test/BP_HUDTestPawn", code:
  "entry override GetHealth() -> (float Health01) { return (Health01: $Health01) }"}
-> [COMPILE_FAILED] Override 'GetHealth' does not match the parent signature:
   Output name mismatch at index 0: expected 'Health01', actual 'ReturnValue'
```

## Diagnosis

The multi-output grammar `-> (Type Name, Type Name, ...)` is documented in `bpir.entry-points`
(§1: *"Both `function` and `macro` entries support multi-output via `-> (type Name, type Name, ...)`
syntax. Functions with a single return value can use the shorthand `-> type`"*). With **exactly one**
entry in the parentheses the parser evidently collapses to the shorthand path and hardcodes the
UE default pin name `ReturnValue`, discarding the authored identifier. With two or more entries the
names survive — `GetAmmo`'s `Mag` / `Reserve` passed.

The validator clearly *has* the correct expected name in hand (it prints `expected 'Health01'`), so
the loss is upstream in the signature parser, not in the comparison.

## Impact

Any Blueprint Interface function with one output whose pin is not literally named `ReturnValue`
cannot be implemented through `compile_bpir` at all — neither as `entry function` nor as
`entry override`. Named single returns are the normal shape for a BP interface getter
(`GetHealth() -> Health01`), so this blocks a routine authoring task with no in-language escape.

## What should happen

`-> (Type Name)` with one output must produce an output pin named `Name`, exactly as the two-output
form does. If a single named output is genuinely unsupported, the parser must reject the syntax with
that message rather than silently substituting `ReturnValue` and surfacing it as a confusing
signature-mismatch error against the caller's own declaration.

**Workaround used:** recreate the interface's single-output functions with the output pin literally
named `ReturnValue` (`blueprint.add_function outputs:[{name:"ReturnValue",type:"float"}]`), which
matches UE's own default for a single return and lets BPIR compile. This is only available when you
own the interface; an agent implementing someone else's interface has no workaround short of
`blueprint.graph.*` node surgery inside the existing interface graph.

severity rationale: impact=hard blocker with no in-language workaround (a routine, valid authoring
task is impossible) x reach=BP interfaces with a named getter are common, and `compile_bpir` is the
primary graph-authoring surface -> High.

## Fix

Verdict: PARTLY TRUE. Direct parsing and compilation were already correct: a parenthesized one-item output stays in `OutputParams` with its authored name, while only bare `-> type` uses `ReturnType` and the conventional `ReturnValue` pin. The actual loss was in `BpirTextEmitter.cpp`, which emitted every sole result pin as bare `-> type`; it now uses that shorthand only for `UEdGraphSchema_K2::PN_ReturnValue` and reuses the named-list form for every other sole output.

Files changed: `Source/PinWright/Private/Decompiler/BpirTextEmitter.cpp`, `Source/PinWright/Private/Tests/Bpir/TestBpirSingleNamedOutput.cpp`, the stale explanation in `Source/PinWright/Private/Tests/Bpir/TestBpirMultiBranchReturn.cpp`, and `docs/wiki-src/bpir.entry-points.md` plus `docs/wiki-src/bpir.instructions.md`.

Tests added: `PinWright.bpir.parser.SingleNamedOutputDistinction` guards the already-correct grammar split, and `PinWright.bpir.round_trip.SingleNamedOutputPreserved` exercises production compile, decompile, and recompile while retaining the compact conventional-return form.

Deliberately unchanged: `BpirParser.cpp`, `BpirCompiler.cpp`, multi-output emission, and bare `return value` semantics. Those paths already preserve parenthesized names or intentionally target `ReturnValue`; changing them would alter a valid language contract instead of fixing the decompiler loss.

## History
- `#1-filed` `OPEN` reporter — Hit on EAContentExamples58 (UE 5.8) implementing `BPI_HUDSource` on `/Game/FPS/UI/Test/BP_HUDTestPawn`. Four interface functions authored via `blueprint.add_function`; `blueprint.add_interface` created all four graphs; `blueprint.compile_bpir` then rejected exactly the three single-output ones with `Output name mismatch at index 0: expected '<AuthoredName>', actual 'ReturnValue'`, while the two-output `GetAmmo` (`Mag`, `Reserve`) validated fine — isolating the defect to the one-entry case of the `-> (Type Name)` grammar. `entry override` reproduces it identically, so there is no alternative entry kind. The validator prints the correct expected name, so the authored identifier is being dropped by the signature parser before validation, not mis-compared. Wiki page consulted: `bpir.entry-points` §1 (documents `-> (type Name, ...)` for functions without excluding the single-output case). Worked around by renaming the interface's single outputs to `ReturnValue`; noted that this is only possible because I own the interface.
- `#2-preserve-named-output` `IN-REVIEW` developer — Preserved non-`ReturnValue` single-output names in BPIR decompilation, clarified the shorthand/named grammar, and added `PinWright.bpir.parser.SingleNamedOutputDistinction` plus `PinWright.bpir.round_trip.SingleNamedOutputPreserved`.
