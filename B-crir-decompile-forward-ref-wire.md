---
id: B-crir-decompile-forward-ref-wire
title: "CRIR decompiler emits forward-referencing exec wires the compiler rejects — round-trip fails on real rigs"
status: IN-REVIEW
severity: High
category: bug
tags: [control-rig, rigvm, crir, roundtrip, forward-reference, decompiler, exec-wire]
---

# CRIR decompiler emits forward-referencing exec wires the compiler rejects — round-trip fails on real rigs

`controlrig.decompile_crir` orders the `rig_graph` `unit` instructions by
**UObject name** (`CRIRDecompiler.cpp:490-495`,
`Nodes.Sort([](A,B){ return A.GetName().Compare(B.GetName()) < 0; })`), not by
execution / declaration order. On a real rig whose exec chain's node names do
not sort in topological order, the decompiler emits a `wire_in_*` that
references a `%nN` declared *later* in the same block. The compiler is
single-pass — the parser requires the referenced id to already be in `KnownIds`
(`CRIRParser.cpp:1067-1074`) — so recompiling the decompiler's own output fails
with `CRIR_UNDEFINED_REF`.

This breaks the **documented** CRIR contract:
- `docs/wiki-src` / `controlrig` wiki: "a decompile → compile → decompile
  round-trip is byte-equal."
- `docs/crir-language-reference.md:489`: "Decompile → compile → decompile is
  byte-equal. **This is the authoritative correctness test for CRIR.**"

The decompiler even has a (dev-only) self-check at `CRIRDecompiler.cpp:1023-1027`
asserting the opposite of reality: *"Forward references aren't possible here
(the decompiler emits in declaration order)"* — but it emits in **name-sorted**
order, and the check is `#if WITH_DEV_AUTOMATION_TESTS` so it never runs in the
shipped editor. The existing round-trip regression test
(`CRIR.RoundTrip.ForwardsSolve`, see `F-control-rig-ir-language` #2) only
exercises a synthetic `BeginExecution → SetBoneTransform` two-node chain whose
names happen to sort topologically, so the gap went unnoticed. The design ticket
`F-control-rig-ir-language` (#`Fix`) even anticipated needing "two-pass symbol
resolution if RigVM exec wires need forward references" — that two-pass path was
never shipped.

## Repro (live, this session — replay-confirmed)

Stock project asset `/Game/ExampleContent/ControlRig/Rigs/CR_FK` decompiles a
`RigUnit_ParentConstraint` exec chain in this order (verbatim from
`controlrig.decompile_crir`):

```
%n0 = unit /Script/ControlRig.RigUnit_BeginExecution()
%n1 = unit /Script/ControlRig.RigUnit_GetTransform()
%n2 = unit /Script/ControlRig.RigUnit_ParentConstraint(wire_in_ExecutePin=%n7.ExecutePin)   <-- references %n7
%n3 = unit /Script/ControlRig.RigUnit_ParentConstraint(wire_in_ExecutePin=%n2.ExecutePin)
...
%n7 = unit /Script/ControlRig.RigUnit_SetTransform(wire_in_ExecutePin=%n0.ExecutePin, wire_in_Value=%n1.Transform)   <-- declared 5 lines AFTER %n2
```

Feeding that exact decompiled text back to `controlrig.compile_crir`
(`mode=replace`, the documented round-trip) fails. Minimal isolated replay
(`controlrig.compile_crir`, `context=/Game/ExampleContent/ControlRig/Rigs/CR_FK`,
`mode=replace`, `save=false`):

```
text =
rig_graph "RigVMModel" {
    %n0 = unit /Script/ControlRig.RigUnit_BeginExecution()
    %n1 = unit /Script/ControlRig.RigUnit_GetTransform()
    %n2 = unit /Script/ControlRig.RigUnit_ParentConstraint(wire_in_ExecutePin=%n7.ExecutePin)
    %n3 = unit /Script/ControlRig.RigUnit_ParentConstraint(wire_in_ExecutePin=%n2.ExecutePin)
    %n7 = unit /Script/ControlRig.RigUnit_SetTransform(wire_in_ExecutePin=%n0.ExecutePin, wire_in_Value=%n1.Transform)
}

→ [CRIR_UNDEFINED_REF] line 4: Wire arg 'wire_in_ExecutePin=%n7.ExecutePin' references undefined node '%n7'
```

The text the compiler rejects is text the decompiler produced. The round-trip
guarantee is therefore violated for any rig whose exec-node names don't sort
into topological order — which includes a stock Content-Examples FK rig.

## Impact

The "author / review a Control Rig as text and feed it back" workflow that CRIR
exists for is broken for real rigs: you cannot recompile a rig's own decompiled
text. Severity High — it's the central documented invariant of the IR
("authoritative correctness test"), it reproduces on a stock asset with zero
mutation, and it's silent (no warning on decompile; the failure only shows up on
the recompile).

**Workaround:** Manually reorder the `unit` lines so every `wire_in_*` source is
declared before its sink (topological sort of the exec/data graph) before
recompiling. Tedious and defeats the round-trip promise.

**Fix:** Make the round-trip self-consistent. Either (a) emit `unit` instructions
in topological (definition-before-use) order in `CRIRDecompiler` instead of
sorting purely by UObject name — keep name as a tiebreaker among
mutually-independent nodes for determinism; or (b) make the compiler two-pass:
pre-scan all `%nN` declarations in a block into `KnownIds` before validating
wires (the path `F-control-rig-ir-language` flagged as "two-pass symbol
resolution if RigVM exec wires need forward references"). (b) is the more robust
fix because hand-authored CRIR can also legitimately forward-reference. Either
way, replace the false dev-only self-check comment at `CRIRDecompiler.cpp:1024`
and add a round-trip regression test seeded from a real multi-`ParentConstraint`
exec chain (CR_FK's graph) so the name-sort-vs-topo gap is pinned.

## History
- `#1-initial-repro` `OPEN` reporter — Decompiler sorts graph nodes by UObject name (`CRIRDecompiler.cpp:490-495`), emitting a `wire_in_ExecutePin=%n7` on `%n2` while `%n7` is declared 5 lines later; the single-pass parser (`CRIRParser.cpp:1067-1074`, `KnownIds.Contains`) rejects it with `CRIR_UNDEFINED_REF`. Replay-confirmed live against stock `/Game/ExampleContent/ControlRig/Rigs/CR_FK`: minimal `controlrig.compile_crir` (mode=replace) of the decompiler's own forward-ref pattern → `[CRIR_UNDEFINED_REF] line 4: Wire arg 'wire_in_ExecutePin=%n7.ExecutePin' references undefined node '%n7'`. Breaks the documented "decompile → compile → decompile is byte-equal" guarantee (crir-language-reference.md:489, controlrig wiki). Dev-only self-check at CRIRDecompiler.cpp:1024 wrongly assumes "emits in declaration order". Existing round-trip test only covers a 2-node synthetic chain that sorts topologically; F-control-rig-ir-language anticipated but never shipped the two-pass resolver.
- `#2-two-pass-validation-fix` `IN-REVIEW` developer — Implemented fix (b): made `ValidateInstructionScope` two-pass in `CRIRParser.cpp`. Pass 1 collects every `%localId` declaration in the scope into `KnownIds`; Pass 2 validates `wire_in_*` refs against that complete set, so forward references (wire source declared on a later line of the same `rig_graph`) now resolve while a genuinely-missing/typo'd id is still rejected with `CRIR_UNDEFINED_REF`. This aligns the upfront parse gate with the compiler engine, which already tolerates forward refs (Pass A creates all nodes, Pass B wires — `CRIRCompiler.cpp` `CompileInstructionsIntoGraph`), matching the same two-pass pattern the compiler already uses for block ordering. Also corrected the false dev-only self-check at `CRIRDecompiler.cpp` (was comment "emits in declaration order" + `bSkipReferenceValidation=true`; now documents the name-sort reality and re-parses with `bSkipReferenceValidation=false` so the self-check can actually detect a dangling ref), and the stale doc comment on `FCRIRParser::Parse` in `CRIRParser.h`. Files: `Source/EditorAutomationRpcGateway/Private/CRIR/CRIRParser.cpp`, `Source/.../CRIR/CRIRParser.h`, `Source/.../CRIR/CRIRDecompiler.cpp`. Tests added in `Source/.../Private/Tests/Assets/TestCRIRRoundTrip.cpp`: `CRIR.Parse.ForwardRefWire` (drives production `FCRIRParser::Parse` with the ticket's minimal forward-ref repro and asserts it parses; negative control asserts a nowhere-declared `%n9` still errors `CRIR_UNDEFINED_REF`) and `CRIR.RoundTrip.ForwardRefWire` (synthetic CR with nodes named so name-sort is the reverse of exec order — source "ZBegin" sorts after sink "ASet" — forcing the decompiler to emit a forward-ref wire, then asserts decompile→compile→decompile is byte-equal). Both fail if the two-pass pre-scan is reverted to single-pass. Not compiled/run here (later phase).
- `#3-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 1 citation sits in history rows and is left verbatim per the append-only rule, mapping by the same rule; the mapped path was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
