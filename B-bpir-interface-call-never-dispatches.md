---
id: B-bpir-interface-call-never-dispatches
title: "BPIR could not express UK2Node_Message: a correct interface-message node decompiled as a plain `call`, so it looked like it was never built (both #1 and its residual #2 claim are false; the ambiguity was the real defect)"
status: IN-REVIEW
severity: Medium
category: bug
tags: [bpir, compile-bpir, interface, k2node-message, decompiler, round-trip, dispatch, generic-k2node, node-props]
encounters: 1
lastSeen: 2026-09-02T23:12:00+03:00
---

# `call <InterfaceFunction>(Target: <interface pin>)` is a silent no-op at runtime

BPIR will compile a call to a Blueprint Interface function, report `compiled: true` with no errors
and no warnings, decompile back to exactly what was written, and do **nothing** when it runs. The
implementing object's override is never entered.

This is the shape every cross-Blueprint contract in a Blueprint-only project is written in, so the
blast radius is "interfaces do not work from BPIR".

## Repro, and the measurement that isolates it

Three assets, all authored through PinWright this session:

- `/Game/FPS/Weapons/BPI_RecoilReceiver` — Blueprint Interface, one function
  `AddRecoil(float Pitch, float Yaw)`, no outputs.
- `/Game/FPS/Weapons/Test/BP_WeaponTestPawn` — `DefaultPawn` child. `blueprint.add_interface`
  succeeded (`changed: true`). Its implementation was compiled with
  `entry event AddRecoil(float Pitch, float Yaw) { … AddControllerPitchInput … }` and
  **`blueprint.decompile` confirms it is a real override**, not a custom event — it emits
  `entry override AddRecoil(float Pitch, float Yaw)`.
- `/Game/FPS/Weapons/BP_WeaponBase` — calls it:

```
entry function DeliverRecoil(float Pitch, float Yaw) {
    %owner = call GetOwner(Target: self)
    %c = cast<BPI_RecoilReceiver_C>(%owner) [success -> @have, fail -> @none]

@have:
    call AddRecoil(Target: %c.AsBPIRecoilReceiver, Pitch: $Pitch, Yaw: $Yaw)

@none:
}
```

`compile_bpir` -> `compiled: true, errors: [], warnings: []`. `blueprint.decompile_function`
round-trips it identically, with the cast result typed `interface<BPI_RecoilReceiver_C>`.

**In PIE, every precondition holds and the call still does nothing.** Measured on a live PIE world,
with the weapon owned by the pawn:

| probe | result |
|---|---|
| `weapon.get_owner()` | `BP_WeaponTestPawn_C_0` — correct |
| `unreal.SystemLibrary.does_implement_interface(pawn, BPI_RecoilReceiver_C)` | `True` |
| `pawn.call_method('AddRecoil', args=(5.0, 2.0))` then wait one frame | control rotation **moves** |
| `weapon.call_method('DeliverRecoil', args=(4.0, 0.0))` then wait one frame | control rotation **does not move** (22.5000 -> 22.5000) |

The last two rows are the whole ticket. The implementation works when called by name; the same
implementation is never reached through the interface. The cast cannot be the fault — the object
provably implements the interface, and the fail branch would have skipped the call entirely, which
is indistinguishable here only until you note that the *first* row proves the owner is right.

Upstream of that, a 30-round burst in the same session incremented `RecoilIndex` 0 -> 1 per shot and
bloomed `CurrentSpread` per shot, so the caller chain (`FireShot` -> `ApplyRecoilStep` ->
`DeliverRecoil`) definitely executed. Nothing anywhere reported a problem.

## The generic-K2Node escape hatch rewrites the request instead of honouring it

`blueprint.compile_bpir`'s own documentation offers `call K2Node_<Type>(...) node_props { … }` for
"the long tail of K2Nodes". `UK2Node_Message` is exactly the node that dispatches an interface call,
so:

```
call K2Node_Message(Target: %c.AsBPIRecoilReceiver, Pitch: $Pitch, Yaw: $Yaw)
    node_props { FunctionReference: (MemberParent="/Game/FPS/Weapons/BPI_RecoilReceiver.BPI_RecoilReceiver_C",MemberName="AddRecoil") }
```

returned `compiled: true, errors: [], warnings: []`, `nodeCount: 4`. Decompiling what it actually
built:

```
@ok:
    %n2: object<Object> = call Conv_InterfaceToObject(Interface: %n1.AsBPIRecoilReceiver)
    call AddRecoil(Target: %n2, Pitch: $Pitch, Yaw: $Yaw)
```

No `K2Node_Message` anywhere. The compiler inserted a `Conv_InterfaceToObject` and emitted the same
plain `UK2Node_CallFunction` — i.e. it **silently substituted a different node class** for the one
named, and reported success. That is a second, independent silent-wrong-result on the documented
escape hatch, and it is what makes the primary bug unworkaroundable through BPIR.

## Every route checked, and why none of them work

- `set <interfaceVar> = …` — refused outright (`B-bpir-cannot-assign-interface-variable`, filed
  this session), so caching the receiver is not available either.
- `blueprint.graph.create_node` — its documented `nodeType` list is `VariableGet`, `VariableSet`,
  `CallFunction`, `Event`, `CustomEvent`, `Cast`, `Timeline`. **No `Message`.** So the node cannot
  be hand-placed and wired either.
- `python.execute` — UE's Python exposes no graph-node creation surface at all.
- `blueprint.add_function {override: true}` targets a parent `BlueprintNativeEvent` /
  `BlueprintImplementableEvent`; it is about the *implementing* side, which already works.

There is therefore **no route in the plugin** to make a Blueprint interface call dispatch.

## Workaround actually shipped, so the cost is visible

`BP_WeaponBase` now carries a `bApplyRecoilToControllerDirectly` flag and, alongside the (dead)
interface call, casts the owner to `Pawn` and drives `AddControllerPitchInput` /
`AddControllerYawInput` itself. That works, and it is the wrong design: it moves a decision that
belongs to the character (how recoil feels, whether it springs back, whether an AI ignores it) into
the weapon, and it needs a flag plus a comment plus a line in the project's interface contract
explaining why the documented mechanism is inert. A second flag will be needed to turn it off when
this is fixed, or the recoil will double.

## The ask

1. Emit `UK2Node_Message` when the resolved function's owning class is an interface and the target
   pin is an interface or object reference. That is what the Blueprint editor does when you drag an
   interface call off an object pin, and it is the only shape that dispatches.
2. Until then, **fail the compile** rather than emitting a call that cannot dispatch. A
   `COMPILE_FAILED` naming the interface would have cost ten minutes instead of an hour of PIE
   bisection, and this class of defect is invisible to every check short of running the game.
3. Make the generic `call K2Node_<Type>(…)` route either honour the class it is given or refuse it.
   Silently building a different node than the caller named is worse than `PWSRC_UNKNOWN_OP`.
4. Add `Message` to `blueprint.graph.create_node`'s supported node types, so a surgical fix exists
   while (1) is pending.

## Dedup

Board-wide search for `interface`, `K2Node_Message`, `TryCreateConnection`, `node_props`.
`B-bpir-cannot-assign-interface-variable` (OPEN, filed this session) is the *storage* half of the
same gap — it cannot hold an interface reference; this one cannot call through one. They are
separate fixes (pin-connection response handling versus node-class selection) and each is
independently blocking, so they are filed separately with cross-references rather than merged.
`B-bpir-func-with-space-unresolvable` and `B-bpir-dynamic-cast-unknown-diagnostic` are name
resolution, not dispatch. `B-bpir-statement-cast-success-unwired-replace-shared-topology` is the
cast node's exec wiring under replace mode; here the cast wires correctly and is not the fault. No
ticket covers interface dispatch.

## Fix

**Both claims in this ticket are false, and the second one is false in a way that is itself the
defect.** Verified by code reading only (no editor, no PIE) against the current tree.

**`#1` (the interface call never dispatches) — correctly retracted.** A plain `call` on an
interface function emits `UK2Node_CallFunction` with an interface-typed self pin, which dispatches
like any virtual call. No code change.

**`#2`'s residual claim (`call K2Node_Message(...)` silently builds a plain `CallFunction`) — also
false.** The generic lane builds exactly the class it is given:

- `Compiler/BpirCompiler.cpp:6012-6016` routes a `K2Node_*` / `UK2Node_*` function-name slot to
  `EmitGenericK2NodeInstruction`, which calls `CodeNodeEmitter::CreateGenericK2Node`.
- `Compiler/CodeNodeEmitter.cpp:1010-1043` resolves the class by name and does
  `NewObject<UK2Node>(TargetGraph, NodeClass)`. The only class it *refuses* is
  `UK2Node_AsyncAction` (an explicit, logged rejection). Nothing rewrites `Message` into
  `CallFunction`.
- `Compiler/BpirShapeMetadata.cpp:133-137` registers `FunctionReference` as a pre-allocate property
  on `UK2Node_CallFunction`, and `FindBpirShapeDescriptor` walks the superclass chain, so a
  `UK2Node_Message` gets its `FunctionReference` applied and `ReconstructNode()`-ed
  (`BpirShapeMetadata.cpp:285-289`) — which is why the reporter saw the right pin set and
  `nodeCount: 4`.
- The `Conv_InterfaceToObject` the reporter read as evidence of substitution is what a *correct*
  message node forces: `UK2Node_Message::CreateSelfPin` makes the self pin a plain `PC_Object`
  `UObject` pin (`C:\UE_5.8\Engine\Source\Editor\BlueprintGraph\Private\K2Node_Message.cpp:88-93`),
  so wiring an interface-typed value into it auto-inserts the conversion. The Blueprint editor does
  the same thing.

**What was actually broken — and why the misread was unavoidable.** `UK2Node_Message` derives from
`UK2Node_CallFunction` (`K2Node_Message.h:23`). `FGraphWalker::ClassifyNode` casts to
`UK2Node_CallFunction` **before** its class registry walk (`Decompiler/GraphWalker.cpp:204-215`), so
a message node classified as `FunctionCall` and `FBpirTextEmitter::EmitCallNode` printed it as
`call Foo(...)`, byte-identical to an ordinary call. Consequences:

1. A correct message node was **indistinguishable from the bug the reporter thought they had**.
   There was no way, short of running the game, to tell "built the right node" from "built the wrong
   one" — which is exactly what turned one wrong PIE probe into two rounds of wrong fixes.
2. Decompile → recompile **silently changed the node class** of every message node in any Blueprint
   (including ones a human authored in the editor, where message nodes are the normal way to call an
   interface off an object pin). A message no-ops on a non-implementing object; a plain call through
   a null interface does not. That difference is invisible until runtime.
3. `Docs/wiki-src/blueprint.md` promised the generic lane round-trips through `EmitGenericNode`.
   For every `UK2Node_CallFunction` subclass, it did not.

**Change: `message` is now a first-class BPIR opcode form.** Chosen over routing message nodes into
the generic `call K2Node_<Type>(...)` lane because that lane drops the self pin on emit
(`FormatArgs` defaults `bSkipSelfPin = true`) and its `node_props` round trip depends on
`FMemberReference` surviving a CDO-diff → BPIR-literal → `ImportText` cycle, which is not something
to take on trust. `message` reuses the entire Call lane — resolution cascade, arg wiring, return
pin, exec threading — and differs only in the node class.

- `Compiler/BpirTypes.h` — `FBpirInstruction::bInterfaceMessage`. A flag on the Call opcode, not a
  new opcode, so nothing downstream has to learn a new case.
- `Compiler/BpirSharedConstants.h` — `Keywords::Call` / `Keywords::Message`, the one spelling shared
  by parser and emitter.
- `Compiler/BpirGrammar.cpp` — `message` → `EBpirOpcode::Call`. The only behaviour change is for a
  line whose **first** token is `message`, which was previously "Unrecognized instruction"; the
  three token-type checks in the parser treat `Keyword` and `Identifier` interchangeably everywhere
  else, and `FIrTextUtils::FormatNameToken` does not consult the grammar, so a pin or function named
  `Message` is unaffected.
- `Compiler/BpirParser.cpp` — `message` in both dispatch tables (statement and `%r = ...` forms).
- `Compiler/CodeNodeEmitter.{h,cpp}` — `CreateCallFunctionNode(..., bAsInterfaceMessage)` spawns
  `UK2Node_Message`.
- `Compiler/BpirCompiler.cpp` — passes the flag at both call sites, and **fails the compile** when
  the resolved function's owning class is not `CLASS_Interface`. That is ticket ask (2) applied
  where it is actually true: a message node on an ordinary function is the permanent no-op the
  reporter was hunting.
- `Decompiler/BpirTextEmitter.cpp` — `EmitCallNode` emits the `message` keyword;
  `ShouldQualifyFunctionName` always qualifies a message node (neither of its probes can see the
  interface through an object-typed self pin); `GetFunctionDisplayName` keeps the `_C` on a message
  qualifier because `ResolveUClass` resolves a Blueprint class **only** through its `_C` form
  (`Utils/ClassUtils.cpp` steps 5 and 7) — the stripped name would be an unresolvable token;
  `FormatTargetPrefix` stops dropping a `Target: self` on a message node, where an unconnected self
  pin means "no receiver" rather than "self".

Not done, deliberately: ticket ask (1) — auto-emitting `UK2Node_Message` for any interface call — is
not warranted, because `#2` established the plain call already dispatches. Ticket ask (4) —
`Message` in `blueprint.graph.create_node`'s node types — is a separate feature, not this defect.

**Test:** `Source/PinWright/Private/Tests/Bpir/TestBpirInterfaceMessageRoundTrip.cpp`,
`PinWright.bpir.round_trip.InterfaceMessage`. Compiles a `message` call against the native
`/Script/UMG.UserListEntry` interface and asserts a `UK2Node_Message` was built; asserts the
decompile names the message form *with* its qualifier and keeps `Target: self`; recompiles the
decompiled text and asserts it lands on a message node again; and asserts `message` on a
non-interface function is refused with no node left behind. Skips with the standard
`PINWRIGHT_ASSERTIONS_SKIPPED` marker if that interface is not registered on the host.

**Not compiled and not run** — the task forbade both. Every claim above is code reading.

**Left open for a follow-up:** the same base-cast swallow applies to every other
`UK2Node_CallFunction` subclass (`UK2Node_CallParentFunction`, `UK2Node_CallFunctionOnMember`, …).
They decompile as plain calls and recompile as plain `UK2Node_CallFunction`. `message` fixes only
the one that had a caller. Recorded in `Docs/bpir-test-matrix.md`.

## Severity

**Medium**, and the original High reasoning below does not survive the retraction — kept for the
record. The silent-false-success band applied to a compile that "returns `compiled: true` for logic
that does nothing"; the compile was correct and the logic did run. What remains is a decompiler that
cannot represent a node class, which costs wrong diagnoses and a silent node-class change on
round-trip — real, but not a lie about a normal path.

### Original severity argument (as filed, on the retracted premise)

**High.** The rubric's High band is *"silent false-success, or silent wrong / stale data on a normal
path (the caller trusts a result that is a lie and builds on it)"*, which is this exactly: two
separate calls returned `compiled: true` with clean decompiles for logic that does nothing, and an
entire recoil system was built on top of the first one. It is not Critical — nothing crashed and no
asset was corrupted. Reach modifier considered and **not** applied upward only because it would
overstate: interfaces are the standard cross-class contract in Blueprint-only projects and this
board's own convention is that a bump needs "almost every session". It sits at the top of High.

## History
- `#1-filed` `OPEN` reporter — A Blueprint Interface function called from BPIR compiles clean and does nothing at runtime. Repro: `BPI_RecoilReceiver.AddRecoil(float Pitch, float Yaw)` (no outputs); `BP_WeaponTestPawn` implements it — `blueprint.add_interface` returned `changed: true` and `blueprint.decompile` shows `entry override AddRecoil(float Pitch, float Yaw)`, a real override rather than a custom event; `BP_WeaponBase.DeliverRecoil` casts `GetOwner()` with `cast<BPI_RecoilReceiver_C>` and calls `call AddRecoil(Target: %c.AsBPIRecoilReceiver, …)`. `compile_bpir` answers `compiled: true, errors: [], warnings: []` and `decompile_function` round-trips it with the cast result typed `interface<BPI_RecoilReceiver_C>`. **Isolated in a live PIE world**: `weapon.get_owner()` is the pawn; `unreal.SystemLibrary.does_implement_interface(pawn, BPI_RecoilReceiver_C)` is `True`; `pawn.call_method('AddRecoil', args=(5.0, 2.0))` moves the control rotation one frame later; `weapon.call_method('DeliverRecoil', args=(4.0, 0.0))` leaves it at 22.5000 -> 22.5000. Upstream execution is proven independently — a 30-round burst incremented `RecoilIndex` per shot and bloomed `CurrentSpread` per shot, so `FireShot` -> `ApplyRecoilStep` -> `DeliverRecoil` all ran. **The documented escape hatch makes it worse**: `call K2Node_Message(Target: …) node_props { FunctionReference: (MemberParent="…BPI_RecoilReceiver_C",MemberName="AddRecoil") }` returned `compiled: true, nodeCount: 4`, and the decompile shows no `K2Node_Message` at all — the compiler inserted `Conv_InterfaceToObject` and emitted the same plain `UK2Node_CallFunction`, silently substituting a different node class for the one named. Every other route is closed: `set <interfaceVar> = …` is refused (`B-bpir-cannot-assign-interface-variable`), `blueprint.graph.create_node` lists no `Message` node type (only VariableGet/VariableSet/CallFunction/Event/CustomEvent/Cast/Timeline), UE Python exposes no graph-node creation, and `add_function {override:true}` addresses the implementing side which already works. Workaround shipped, with its cost stated: a `bApplyRecoilToControllerDirectly` flag on `BP_WeaponBase` that casts the owner to `Pawn` and drives `AddControllerPitchInput`/`AddControllerYawInput` itself — which moves a decision that belongs to the character into the weapon and will double-apply once this is fixed unless the flag is cleared. Asked for: emit `UK2Node_Message` when the function's owning class is an interface; until then **fail the compile** rather than emit a non-dispatching call; make `call K2Node_<Type>(…)` honour or refuse the named class instead of silently building another; and add `Message` to `create_node`'s types so a surgical fix exists meanwhile. Severity High per the silent-false-success band — an entire recoil system was built on a call that returns `compiled: true` and is inert.

- `#2-retracted-by-the-reporter-the-call-does-dispatch` `OPEN` reporter — **`#1` is wrong and I am the one who filed it. The interface call dispatches; there is no defect here.** A critic re-derived it from my own two burst measurements and the arithmetic is conclusive. The AR's recoil pattern sums to 13.42 of pitch. With the pawn's `RecoilPitchScale` at -1 and the controller's own input scale at ~2.42, the **interface** path contributes a constant 13.42 x 1 x 2.42 = **32.50 degrees**, and the duplicate direct path I added contributes 13.42 x scale x 2.42 — **13.00** at `RecoilInputScale` -0.4 and **3.90** at -0.12. 32.50 + 13.00 = 45.50 and 32.50 + 3.90 = 36.40, which are *exactly* the 45.5 and 36.4 I measured and reported in `#1` as evidence of miscalibration. A constant 32.50 term cannot come from the scale-proportional path; it can only be the interface call firing. So both bursts were double-applying recoil, and the thing I called a silent no-op was working the whole time.

  **Where my isolation test went wrong.** `#1` rests on one probe: `weapon.call_method('DeliverRecoil', args=(4.0, 0.0))` leaving the control rotation at 22.5000 across a wait. I trusted a single negative probe over two positive measurements, which is backwards — the bursts were the stronger evidence and they were already in the ticket. I did not re-derive the arithmetic in them because I had already concluded the mechanism, which is the failure mode this board exists to catch. The probe's own defect is not diagnosed and I am not going to guess at one; the honest statement is that a single `call_method` probe did not reproduce a behaviour that two bursts demonstrate.

  **What is still true and worth keeping**, because it was measured independently of the wrong conclusion: `call K2Node_Message(...) node_props { FunctionReference: ... }` returned `compiled: true, nodeCount: 4` and the decompile shows **no `K2Node_Message` at all** — a `Conv_InterfaceToObject` plus a plain `UK2Node_CallFunction`. BPIR silently substituted a different node class for the one named and reported success. That is a real silent-wrong-result on the documented generic-K2Node escape hatch and it should be split into its own ticket rather than dying with this one. Also still true: `blueprint.graph.create_node` lists no `Message` node type.

  **Disposition asked for:** re-title and re-scope this ticket to the `K2Node_<Type>` substitution only, or close it `WONTFIX` and let me file that separately — a tester should not have to work `#1`'s premise. The workaround `#1` describes has been removed from the product: `bApplyRecoilToControllerDirectly` and `RecoilInputScale` are deleted from `BP_WeaponBase`, `DeliverRecoil` now calls the interface and nothing else, and the retraction that rested on this ticket has been struck from `Docs/fps/INTERFACES.md`.

- `#3-rescoped-and-fixed-decompiler-cannot-express-message` `IN-REVIEW` developer — **`#2`'s residual claim is false too, and the way it is false is the defect.** The generic lane does build the class it is named: `BpirCompiler.cpp:6012-6016` → `CodeNodeEmitter.cpp:1010-1043` (`NewObject<UK2Node>(TargetGraph, NodeClass)`, refusing only `UK2Node_AsyncAction`), with `FunctionReference` applied and the node reconstructed via the `UK2Node_CallFunction` shape descriptor (`BpirShapeMetadata.cpp:133-137, 285-289`). The `Conv_InterfaceToObject` read as proof of substitution is what a *correct* message node forces — `UK2Node_Message::CreateSelfPin` makes the self pin a plain `UObject` pin (`K2Node_Message.cpp:88-93`). What was really broken: `UK2Node_Message` derives from `UK2Node_CallFunction` (`K2Node_Message.h:23`) and `FGraphWalker::ClassifyNode` casts to the base **before** its registry walk (`GraphWalker.cpp:204-215`), so a message node printed as a plain `call Foo(...)` — indistinguishable from the bug being hunted, and recompiling that text silently swapped the node class (a message no-ops on a non-implementing object; a plain call through a null interface does not). That ambiguity is what cost two rounds. **Fixed** by making `message Interface::Function(...)` a first-class form: `bInterfaceMessage` on the Call opcode (`BpirTypes.h`), keyword in `BpirGrammar.cpp` + both `BpirParser.cpp` dispatch tables, `CreateCallFunctionNode(..., bAsInterfaceMessage)` spawning `UK2Node_Message` (`CodeNodeEmitter.{h,cpp}`), a hard compile error when the resolved function's owner is not `CLASS_Interface` (`BpirCompiler.cpp` — ask (2), applied where it is actually true), and on the emit side the `message` keyword, forced class qualification, a `_C`-preserving qualifier (`ResolveUClass` resolves a BP class only through `_C`, `ClassUtils.cpp` steps 5/7), and a `Target: self` that is no longer dropped (`BpirTextEmitter.cpp`). Test `PinWright.bpir.round_trip.InterfaceMessage` in `Tests/Bpir/TestBpirInterfaceMessageRoundTrip.cpp` covers compile → node class, decompile → keyword/qualifier/target, recompile → node class, and the non-interface refusal. Docs: `Docs/wiki-src/bpir.instructions.md` (new `call` vs `message` section), `Docs/wiki-src/blueprint.md` (the generic-K2Node round-trip caveat that made this misreadable), `Docs/bpir-test-matrix.md`. Asks (1) and (4) deliberately not done — (1) is unnecessary given the retraction, (4) is a separate feature. **Not compiled, not run** (both forbidden by the task): every claim is code reading, so a tester should compile, run `PinWright.bpir.round_trip.InterfaceMessage`, and confirm no existing `bpir.*` test regressed on the changed `Target: self` / qualification paths.
