---
id: B-bpir-member-call-through-chained-property
title: "BPIR cannot call a member function through a chained property ref (`Target: %m.PauseRef`) — 'Unresolved function', and a same-named local function silently binds Target to self"
status: DONE
severity: Medium
category: bug
tags: [bpir, compile_bpir, member-call, chained-property, target-pin, function-resolution]
encounters: 1
costly: 1
lastSeen: 2026-09-03T00:50:00Z
---

# A chained property is not usable as a member-call `Target`

## Repro A — `Unresolved function` through a chained ref

```
entry function DebugMenuNav() {
    %c = call GetController(Target: self)
    %m = cast<BP_HUDManager_C>(%c)
    %okp = call IsValid(Object: %m.PauseRef)          <- this resolves fine
    %b2 = branch(%okp) [true -> @nav, false -> @endNav]
@nav:
    call DebugFocusNext(Target: %m.PauseRef)          <- this does not
@endNav:
    return
}
-> [COMPILE_FAILED] Unresolved function: 'DebugFocusNext'. Searched:
     - Blueprint class: BP_HUDTestPawn_C
     - UKismetMathLibrary ... and 335 more libraries
```

`%m.PauseRef` is a `WBP_PauseMenu_C` reference and `DebugFocusNext` is public on that class. The
same chain works as a **value** (`IsValid(Object: %m.PauseRef)` compiles), so the chain resolves;
what fails is using it to pick the class for function lookup. Binding the same value to a typed
member variable first and calling `Target: $MenuRef` works.

## Repro B — same-named local function captures the Target pin

Before hitting A, the pawn's own function was also called `DebugFocusNext`:

```
call DebugFocusNext(Target: %m.PauseRef)
-> [COMPILE_FAILED] TryCreateConnection failed wiring data 'PauseRef' -> 'self'
```

The resolver matched the **pawn's own** `DebugFocusNext` (a self-call whose only object pin is
`self`) and then tried to wire the widget reference into that `self` pin. Renaming the caller to
`DebugMenuNav` changed the error to Repro A, which is how the two were separated.

## What should happen

- A chained property ref should supply its class for member-function resolution wherever it is
  accepted as a value, so `Target: %ref.Prop` works like `Target: $TypedVar`.
- When a name matches both a local function and a member function of the supplied `Target`'s class,
  the `Target`'s class should win — or the error should say the name was ambiguous, rather than
  reporting a pin-wiring failure that points at the argument.

**Workaround:** assign the chained value to a typed member variable and call through that, or move
the call into the class that already owns a typed reference (what I did: the helper now lives on
`BP_HUDManager`, which has `PauseRef` as a typed variable).

severity rationale: impact=soft blocker with a workaround, but both diagnostics misdirect — one
lists 335 libraries without mentioning the chain, the other blames the argument x reach=calling into
a widget or subobject held by another class is routine -> Medium.

## History
- `#1-filed` `OPEN` reporter — Hit on EAContentExamples58 (UE 5.8) building a pawn-side test hook that drives the pause menu (`BP_HUDTestPawn` -> `BP_HUDManager.PauseRef` -> `WBP_PauseMenu.DebugFocusNext`), needed because `B-simulate-input-key-events-never-reach-pie-pawn` blocks the critic from opening the menu. Repro B came first (identical function names), Repro A after renaming. `IsValid(Object: %m.PauseRef)` on the line above compiles in the same body, which is what shows the chain itself is fine and only the function-lookup path rejects it. Resolved by relocating the helper onto the class holding the typed reference.
- `#2-chain-target-class` `IN-REVIEW` developer — Reproducible in current source. `ResolveViaTargetArg` in `Source/PinWright/Private/Compiler/BpirCompiler.cpp` split `Target: %m.PauseRef` into `m` + `PauseRef`, looked for an OUTPUT PIN named `PauseRef` on `%m`'s node, and on a miss silently kept the producer's primary pin (the cast's `As...` pin), so the lookup ran on the holder's class. Not found there, the cascade fell to the calling Blueprint (Repro B: same-named local function bound, then `PauseRef -> self` wiring failed) or to libraries (Repro A: `Unresolved function`). Fix: when the `%ref` suffix is not an output pin it is a property chain, so the class now comes from `ResolveTargetClass(Arg.Value)` (the value resolver's chain walk, the same pin the wire pass connects to Target). Both repros are fixed by the same change: the Target's class now wins over a same-named local function. Not changed: when the Target class is resolved but genuinely lacks the function, the cascade still falls through to self/libraries as before (same as `Target: $Var`); an alias register (`%r = $Var`) as `Target:` still has no class at this step. Tests in new `Tests/Bpir/TestBpirChainedPropertyRefs.cpp`: `PinWright.bpir.compiler.chained_property.MemberCallTargetThroughChain` (Repro A shape: callee BP function reached via `cast<Holder>($HolderRef)` then `Target: %m.CalleeRef`) and `PinWright.bpir.compiler.chained_property.ChainTargetBeatsSameNamedLocalFunction` (Repro B shape: caller also defines `ChainCalleeFn()`); both assert success, the call bound to the callee BP's function, the `Count` pin and literal, and self wired from the `CalleeRef` VariableGet. Docs: `docs/wiki-src/bpir.instructions.md` (chain as member-call Target), `docs/bpir-test-matrix.md`, CHANGELOG. Compile-checked with UBT -SingleFile; not yet run.
- `#3-review-silent-class-and-alias` `IN-REVIEW` developer — Review follow-up. (1) BLOCKER: the chain branch in `ResolveViaTargetArg` called `ResolveTargetClass`, which logs `LogBpirCompiler: Error ... pin type is not an object class` when `Target:` is a value parameter fed by a non-object chain (`FInterpTo(Target: %v.X)`, `%hit.OutHit.ImpactPoint`, backtick `%loop.`Array Element``) — a logged Error that fails any automation test on that path. It now uses a silent `GetAuthoritativePinClass(ValueResolver->ResolveValue(Arg.Value, Block))` and returns nullptr on a non-object pin. (2) An alias register (`%r = $Var`, `%r = get $obj.Prop`, no emitted node) as `Target:` now resolves its class through the same silent pair instead of falling back to the caller/libraries. Both in `Source/PinWright/Private/Compiler/BpirCompiler.cpp` (`ResolveViaTargetArg`, `%` branch, shared `ResolveWiredValueClass` lambda). New tests in `Tests/Bpir/TestBpirChainedPropertyRefs.cpp`: `PinWright.bpir.compiler.chained_property.ValueParamNamedTargetStaysSilent` (fails through the automation log-error hook if the Error comes back) and `PinWright.bpir.compiler.chained_property.MemberCallTargetThroughGetAlias` (`%m = get $HolderRef.CalleeRef` then `call ChainCalleeFn(Target: %m, Count: 2)`). Docs: `bpir.instructions.md` (alias register as Target), `bpir-test-matrix.md`. Syntax-checked with clang -fsyntax-only (fastcheck.sh, UBT module flags); no UHT-relevant declarations changed; not yet run.
- `#4-review-silent-dollar-target` `IN-REVIEW` developer — Re-review follow-up: the `$` branch of `ResolveViaTargetArg` (`Source/PinWright/Private/Compiler/BpirCompiler.cpp`) still went through `ResolveTargetClass`, which logs `LogBpirCompiler: Error ... not an object class` for `call FInterpTo(Target: $SomeFloat)` / `Target: $obj.FloatProp`. The silent `ResolveWiredValueClass` lambda (`GetAuthoritativePinClass(ValueResolver->ResolveValue(...))`) is now hoisted to the per-arg scope and used by both the `%` and `$` branches; `ResolveTargetClass`'s `self` and ConvertAsset special cases never apply to `$` refs, so object targets resolve the same class as before. Test `PinWright.bpir.compiler.chained_property.DollarValueParamNamedTargetStaysSilent` (`Tests/Bpir/TestBpirChainedPropertyRefs.cpp`): `FInterpTo(Target: $Goal)` and `FInterpTo(Target: $Cam.FieldOfView)` compile, both Target pins wired, and a logged Error fails the test. Also: CHANGELOG entry now lists alias-register Targets and the new `%r = $obj.Prop` rejection message; `AliasDotSuffixRejected` header comment points at `%x = get $MyVar.Field`. Syntax-checked with fastcheck.sh; not yet run.
- `#5-verified-linux` `DONE` tester — Run3 on PinWright 7230b41d, UE 5.8 Linux, all non-skipped in run3/full. Repro A: `PinWright.bpir.compiler.chained_property.MemberCallTargetThroughChain` (`cast<Holder>` then `Target: %m.CalleeRef`) binds the callee BP's function, with the `Count` literal and self wired from the `CalleeRef` VariableGet. Repro B: `PinWright.bpir.compiler.chained_property.ChainTargetBeatsSameNamedLocalFunction`: the Target's class wins over the caller's same-named function. Review guards: `PinWright.bpir.compiler.chained_property.ValueParamNamedTargetStaysSilent` and `PinWright.bpir.compiler.chained_property.DollarValueParamNamedTargetStaysSilent` show that `FInterpTo(Target: %v.X / $Goal / $Cam.FieldOfView)` compiles with no logged Error. `PinWright.bpir.compiler.chained_property.MemberCallTargetThroughGetAlias` covers an alias register as Target. Doc: `bpir.instructions.md` documents a property chain as a member-call Target and says its class wins. Limit: when the Target class genuinely lacks the function, the lookup still falls through to self/libraries, as before (stated in #2).
