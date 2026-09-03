---
id: B-bpir-interface-call-never-dispatches
title: "RETRACTED by reporter in #2 - the interface call does dispatch. Residual real defect: call K2Node_Message(...) silently builds a plain CallFunction instead and reports success"
status: OPEN
severity: Medium
category: bug
tags: [bpir, compile-bpir, interface, k2node-message, silent-noop, dispatch, generic-k2node, node-props]
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

## Severity

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
