---
id: B-bpir-cannot-assign-interface-variable
title: "BPIR cannot assign to an interface-typed Blueprint variable by any route — object RHS and cast<Iface> RHS both fail TryCreateConnection, while the Blueprint editor makes the same connection by hand"
status: OPEN
severity: Medium
category: bug
tags: [bpir, compile-bpir, interface, variable, set, trycreateconnection, k2-schema, conversion-node]
encounters: 1
lastSeen: 2026-09-02T23:05:00+03:00
---

# `set <interfaceVar> = <anything>` fails in BPIR

A Blueprint variable whose pin type is `PinCategory="interface"` cannot be written from BPIR. Both
spellings the language offers fail on the wire, at compile time, with the same diagnostic.

## Repro

Blueprint `/Game/FPS/Weapons/BP_WeaponBase` (parent `AActor`), one member variable:

```
RecoilReceiver => (PinCategory="interface",PinSubCategory="",
  PinSubCategoryObject="/Script/Engine.BlueprintGeneratedClass'/Game/FPS/Weapons/BPI_RecoilReceiver.BPI_RecoilReceiver_C'",
  ContainerType=None, …)
```

created with `unreal.BlueprintEditorLibrary.add_member_variable`, and confirmed by
`get_member_variable_type(...).export_text()` to be exactly the shape the Blueprint editor produces
for an interface variable. `BPI_RecoilReceiver` is an ordinary Blueprint Interface with one function
`AddRecoil(float Pitch, float Yaw)`.

**Attempt 1 — plain object RHS**, which is what the editor accepts by hand (drag an `Actor` pin onto
an interface variable's Set node and the K2 schema inserts the conversion):

```
call("blueprint.compile_bpir", { assetPath: "/Game/FPS/Weapons/BP_WeaponBase", code:
  "entry event BeginPlay() {\n    %owner = call GetOwner(Target: self)\n    set RecoilReceiver = %owner\n}\n" })

-> [COMPILE_FAILED] Line 18: TryCreateConnection failed wiring data 'ReturnValue' -> 'RecoilReceiver'
```

**Attempt 2 — the documented RHS cast form** (`bpir.instructions` §2.3: *"`cast<T>(...)` is accepted
on the right of `set` … Use it when the receiver can handle null"*, which is exactly this case):

```
    set RecoilReceiver = cast<BPI_RecoilReceiver_C>(%owner)

-> [COMPILE_FAILED] Line 18: TryCreateConnection failed wiring data 'AsBPI Recoil Receiver' -> 'RecoilReceiver'
```

The second message is the informative one: the cast node **was** created and its result pin **was**
found, so the failure is purely the final wire into the variable's Set node. The cast to a Blueprint
interface class is legal — `UK2Node_DynamicCast::GetCastResultPin` gives an interface-category pin
for a class with `CLASS_Interface` — and the same two nodes wired by hand in the Blueprint editor
compile and run.

## What this costs

An interface-typed variable is the normal way to cache "the thing I send messages to" — it is how
you avoid re-casting every frame, and it is the shape `INTERFACES.md`-style contracts between
Blueprint classes are written in. With BPIR unable to write one, the only route left is to re-cast
at every call site:

```
entry function DeliverRecoil(float Pitch, float Yaw) {
    %owner = call GetOwner(Target: self)
    %c = cast<BPI_RecoilReceiver_C>(%owner) [success -> @have, fail -> @none]

@have:
    call AddRecoil(Target: %c.AsBPIRecoilReceiver, Pitch: $Pitch, Yaw: $Yaw)

@none:
}
```

That is the workaround in use and it is correct, but it pays a dynamic cast per call instead of one
per possession, and it forced deleting the variable that had already been created — so the class's
shape is now driven by what the IR can wire rather than by the design.

## Suspected mechanism (not source-read)

`TryCreateConnection` is the right call — the wiki's own cast note says a raw `MakeLinkTo` breaks
`UK2Node_DynamicCast` pin notification — so the emitter is using the schema. What the editor does
additionally on an object→interface drop is accept the connection as `CONNECT_RESPONSE_MAKE_WITH_CONVERSION_NODE`
and then spawn the conversion node; a caller that treats any non-`MAKE` response as failure would
produce exactly this symptom, and would explain why the interface→interface case (attempt 2) fails
too if the cast result pin's `PinSubCategoryObject` is the `_C` class while the variable's is
compared by identity rather than by `ImplementsInterface`. Both are guesses; no source was read.

## The ask

1. Accept `CONNECT_RESPONSE_MAKE_WITH_CONVERSION_NODE` in the BPIR emitter's data-wiring path and
   spawn the conversion node the schema names, so `set <interfaceVar> = <objectRef>` compiles the
   way the editor does.
2. Failing that, make the diagnostic say what is wrong. `TryCreateConnection failed wiring data
   'AsBPI Recoil Receiver' -> 'RecoilReceiver'` names two pins and no reason; the schema's own
   `FPinConnectionResponse::Message` is right there and would have said whether a conversion was
   offered, whether the types are incompatible, or whether the interface is not implemented.
3. Either way, `bpir.types`' `interface<InterfaceName>` row should say whether an interface pin can
   be a `set` **target** — it currently lists the type as supported with no caveat, which is what
   sent this session down two dead ends.

## Dedup

Board-wide search for `interface`, `TryCreateConnection`, and `set_default`/variable-write tickets.
`B-array-get-item-connection-fail` is the same *diagnostic* on a different node and was about
wildcard propagation lagging behind wiring (fixed by a `NotifyPinConnectionListChanged`), not about
a conversion-node response — related shape, different cause. `B-bpir-field-notification-delegate-pin-wire-fails`
is a delegate pin, not an interface variable. `B-bpir-statement-cast-success-unwired-replace-shared-topology`
concerns the cast node's *exec* wiring under replace mode. `E-bpir-cast-as-set-rhs` is the request
that produced the `set x = cast<T>(...)` sugar used in attempt 2 above, so this ticket is a gap in
what that sugar can reach rather than a duplicate of it. No ticket covers writing an interface-typed
variable.

## Severity

**Medium.** Impact class is the rubric's *"soft blocker: doable, but only via a documented
workaround"* — re-casting at each call site works and is what shipped. It is not High because
nothing returned a false success: the compile failed loudly and rolled back cleanly both times. It
is not Low because the failure has no diagnostic pointing at a fix, and because an author who does
not already know that BP inserts conversion nodes has no way to reach the conclusion "the language
cannot express this" from the message. Reach modifier declined: interface variables are common in
Blueprint-only projects but not present in every session.

## History
- `#1-filed` `OPEN` reporter — A Blueprint variable of pin type `interface` (`PinSubCategoryObject` = `BlueprintGeneratedClass'/Game/FPS/Weapons/BPI_RecoilReceiver.BPI_RecoilReceiver_C'`, created via `unreal.BlueprintEditorLibrary.add_member_variable` and confirmed with `get_member_variable_type(...).export_text()` to match the editor's own shape) cannot be assigned from BPIR by either available route. `set RecoilReceiver = %owner`, where `%owner = call GetOwner(Target: self)` — the assignment the Blueprint editor makes by hand, inserting a conversion node — fails `[COMPILE_FAILED] Line 18: TryCreateConnection failed wiring data 'ReturnValue' -> 'RecoilReceiver'`. The documented RHS cast sugar from `bpir.instructions` §2.3, `set RecoilReceiver = cast<BPI_RecoilReceiver_C>(%owner)`, fails the same way: `TryCreateConnection failed wiring data 'AsBPI Recoil Receiver' -> 'RecoilReceiver'`. The second message localises the fault precisely — the cast node was created and its result pin resolved, so only the final wire into the variable's Set node failed, and a cast to a Blueprint interface class is legal (`UK2Node_DynamicCast::GetCastResultPin` yields an interface-category pin for a `CLASS_Interface` class). Suspected mechanism, **guessed, no source read**: the emitter likely treats anything other than `CONNECT_RESPONSE_MAKE` as failure and so never spawns the `MAKE_WITH_CONVERSION_NODE` the schema offers. Cost: the variable had to be deleted and every call site now re-casts — `%c = cast<BPI_RecoilReceiver_C>(%owner) [success -> @have, fail -> @none]` then `call AddRecoil(Target: %c.AsBPIRecoilReceiver, …)` — which is correct but pays a dynamic cast per shot instead of one per possession, and let the IR's limits decide the class's shape. Asked for: accept the conversion-node response and spawn the node; failing that surface `FPinConnectionResponse::Message` in the diagnostic instead of two bare pin names; and either way say on `bpir.types` whether an `interface<…>` pin can be a `set` target, since the type is listed as supported with no caveat. Dedup: `B-array-get-item-connection-fail` is the same diagnostic from wildcard-propagation lag, `B-bpir-field-notification-delegate-pin-wire-fails` is a delegate pin, `B-bpir-statement-cast-success-unwired-replace-shared-topology` is the cast node's exec wiring, and `E-bpir-cast-as-set-rhs` is the request that created the sugar attempt 2 uses — this is a gap in that sugar's reach, not a duplicate. No ticket covers writing an interface-typed variable.
