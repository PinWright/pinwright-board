---
id: B-bpir-const-ref-fname-literal-dropped
title: "BPIR silently drops a literal on a `const FName&` pin, so every Blackboard accessor fails the BP compile with \"by ref params expect a valid input\""
status: DONE
severity: Medium
category: bug
tags: [bpir, compile_bpir, pin-default, fname, by-ref, blackboard, ai]
encounters: 1
lastSeen: 2026-09-02T19:55:00Z
---

# A quoted FName literal on a `const FName&` pin is dropped, not applied

## Repro

Any `UBlackboardComponent` accessor. Their key parameter is `const FName& KeyName`, so this is
every AI Blueprint that touches a Blackboard:

```
%bb = call GetBlackboard(Target: %c.AsAIController)
%t  = call GetValueAsObject(Target: %bb, KeyName: "TargetActor")
```

```
[BLUEPRINT_COMPILE_FAILED] BPIR compile failed Blueprint compile: The current value (None) of the
' Key Name ' pin is invalid: 'Key Name' in action 'GetValueAsObject' must have an input wired into
it ("by ref" params expect a valid input to operate on).
```

BPIR placement itself succeeds — the diagnosis comes from UE's Blueprint compiler afterwards, and
`(None)` is the tell: the `"TargetActor"` literal never reached the pin's `DefaultValue`. Nothing in
the BPIR layer reported a problem with the argument; it was accepted and then discarded.

In the Blueprint editor the same node accepts a typed key name in its `Key Name` field with no wire,
so the pin genuinely does hold literal defaults; only the BPIR path fails to write one.

## Workaround

Feed the pin from a `MakeLiteralName` node so it is genuinely *wired*:

```
%k  = call MakeLiteralName(Value: "TargetActor")
%bb = call GetBlackboard(Target: %c.AsAIController)
%t  = call GetValueAsObject(Target: %bb, KeyName: %k)
```

That compiles. The cost is one extra node and one extra BPIR line per Blackboard key touched — an AI
controller and its Behavior Tree tasks reference a dozen keys, so it is a dozen `MakeLiteralName`
nodes that a hand-authored graph would not have. `MakeLiteralInt` / `MakeLiteralFloat` /
`MakeLiteralString` presumably cover the same shape for other by-ref primitive params.

## Expected

Either write the literal into the pin's `DefaultValue` (what the editor does when a user types into
the field), or, if a `const T&` pin genuinely cannot carry a default in this code path, synthesize
the `MakeLiteral*` node automatically — the compiler knows the pin type and has the literal in hand.
Silently dropping the argument and letting UE's compiler fail two layers later, with an error naming
a pin (`' Key Name '`, with the display-name spacing) that does not match the BPIR argument name
(`KeyName`), is the worst of the three.

A one-line note on `bpir.types` § "Optional Empty Pins" or `bpir.errors` would also have saved the
detour: nothing in the wiki mentions that reference parameters need a wired source.

severity rationale: impact=soft blocker (doable, but only via an undocumented workaround discovered
by reading a UE compiler error) x reach=every AI session (all Blackboard get/set verbs, and any
other `const FName&` / `const T&` parameter) -> Medium

## History
- `#1-filed` `OPEN` reporter — Hit while authoring `/Game/FPS/AI/EQC_Target`, a
  `UEnvQueryContext_BlueprintBase` subclass whose `ProvideSingleActor` override reads `TargetActor`
  off the querier's Blackboard. First attempt used the plain quoted literal
  `call GetValueAsObject(Target: %bb, KeyName: "TargetActor")`; `compile_bpir` reported
  `BLUEPRINT_COMPILE_FAILED` with the UE compiler text quoted above. Adding
  `%k = call MakeLiteralName(Value: "TargetActor")` and passing `KeyName: %k` compiled clean
  (`nodeCount:8, compiled:true`) with no other change to the body, which isolates the by-ref pin as
  the cause. Distinct from `B-bpir-fname-pin-bare-identifier-not-resolved` (DONE): that one is about
  the *bare vs quoted* surface form on an ordinary `PC_Name` pin and fails at the BPIR layer with
  `Could not resolve value`; here the quoted form is accepted by BPIR and dropped, and the failure
  is UE's by-ref check on a `const FName&` parameter. Same UE 5.8 host, `EAContentExamples58`.
- `#2-makeliteral-for-by-ref` `IN-REVIEW` developer — Still reproducible in current source: `WireDataPins`' `ApplyDefaultValueToTargetPin` wrote every literal into the target pin's `DefaultValue` with no by-ref check, and `UEdGraphSchema_K2::IsPinDefaultValid` rejects any default on a pin with `bIsReference && !IsAutoCreateRefTerm` (`EdGraphSchema_K2.cpp` ~2990). UHT gives `const FName& KeyName` `CPF_OutParm|CPF_ReferenceParm|CPF_ConstParm` (gen flags `0x...08000182`), so `ConvertPropertyToPinType` sets `bIsReference`. Correction to the report: the editor does **not** accept a typed key in that field — `ShouldHidePinDefaultValue` hides it for exactly this predicate, and `K2Node_CallFunction` sets `bDefaultValueIsIgnored` — so wiring is the only route, which is what the fix does. Fix (`Source/PinWright/Private/Compiler/BpirCompiler.cpp`): new `BpirCompilerByRefLiteral` helpers (`PinRequiresWiredInput` = the engine predicate, input, non-container, not a FunctionResult; `FindMakeLiteralFunction` maps bool/int/int64/real/name/string/text/plain byte to `UKismetSystemLibrary::MakeLiteral*` by name), and `ApplyDefaultValueToTargetPin` — the single path every literal branch (quoted/numeric, `Enum::Value`, enum `%ref`, class path) goes through — now emits the MakeLiteral node via `FCodeNodeEmitter::CreateCallFunctionNode` (so it joins the rollback GUID set), writes the literal into its `Value` pin with the same `SetPinDefaultValue`, and wires `ReturnValue` -> the by-ref pin. Types with no MakeLiteral (struct, object, class, enum) fail at the BPIR layer with `Pin '<name>' is a by-reference parameter (<type>&): Unreal requires a wired input there and ignores a typed default, and no MakeLiteral node exists for this type…` — behaviour change: previously `BLUEPRINT_COMPILE_FAILED`, now `COMPILE_FAILED` naming the BPIR pin. AutoCreateRefTerm pins and by-value pins are untouched. Docs: `docs/wiki-src/bpir.errors.md` Common Pitfalls entry, pointer from `docs/wiki-src/bpir.types.md`; `CHANGELOG.md`. Tests (new `Source/PinWright/Private/Tests/Bpir/TestBpirByRefLiteral.cpp`): `PinWright.bpir.compile.ByRefLiteral.NameLiteralWiredThroughMakeLiteral` (`SetValueAsName(Target: $BB, KeyName: "TargetKey", NameValue: "Hello")` on a `UBlackboardComponent` member: asserts the KeyName precondition `bIsReference`, KeyName linked to `MakeLiteralName` whose `Value` = `TargetKey`, by-value `NameValue` unlinked with default `Hello`, full BP compile clean — fails on revert) and `PinWright.bpir.compile.ByRefLiteral.StructLiteralRefusedWithReason` (`FTruncVector(InVector: FVector(...))` refused naming `InVector`). Filter `PinWright.bpir.compile.ByRefLiteral`. Not run here — the orchestrator owns builds and tests.
- `#3-review-inout-refs` `IN-REVIEW` developer — Review follow-up. The by-ref predicate also matched non-const `UPARAM(ref)` in/out inputs (`bIsReference` without `bIsConst`), where a MakeLiteral temporary would compile and silently drop the callee's write. Now only `const` refs (`PinType.bIsConst`) get a MakeLiteral node; a literal on a non-const ref fails with `Pin '<name>' is a by-reference in/out parameter (<type>&): the function writes through it, so the literal '<v>' has nowhere to land. Pass a variable instead.` Float-subcategory real pins now use `MakeLiteralFloat` (double pins keep `MakeLiteralDouble`). New test `PinWright.bpir.compile.ByRefLiteral.InOutRefLiteralRefused` (`ReplaceInline(SourceString: "abc", …)`; fails if the `bIsConst` gate is dropped). `bpir.errors.md` notes the in/out refusal and that decompile shows the synthesized node as its own `%n = pure MakeLiteralName(...)` line (stable round trip, not byte-identical to the source); `CHANGELOG.md` entry extended. Not run here.
- `#4-verified-linux` `DONE` tester — Run3 on PinWright 7230b41d, UE 5.8 Linux, all non-skipped in run3/full. `PinWright.bpir.compile.ByRefLiteral.NameLiteralWiredThroughMakeLiteral`: `SetValueAsName(Target: $BB, KeyName: "TargetKey", ...)` on a UBlackboardComponent wires KeyName (asserted bIsReference) from a `MakeLiteralName` whose Value is `TargetKey`, leaves the by-value NameValue as a plain default, and the full BP compile is clean. That is the 'synthesize the MakeLiteral node' remedy the ticket offered. `PinWright.bpir.compile.ByRefLiteral.StructLiteralRefusedWithReason`: a by-ref struct literal now fails at the BPIR layer naming the BPIR pin, not two layers later in the BP compiler. `PinWright.bpir.compile.ByRefLiteral.InOutRefLiteralRefused`: a non-const in/out ref literal is refused instead of being silently dropped. Doc: `docs/wiki-src/bpir.errors.md` Common Pitfalls has the by-reference entry, including the in/out refusal and the decompile shape.
