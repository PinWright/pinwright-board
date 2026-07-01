---
id: F-bpir-add-generic-node-statement-form
title: "Round-trip any K2Node subclass via call K2Node_<Type>(args) node_props { ... }"
status: DONE
severity: High
category: feature
tags: [bpir, parser, compiler, generic-node]
---

# Round-trip any K2Node subclass via the existing `call` opcode

The BPIR parser today exposes ~26 K2Node subclasses, each through a
dedicated **sugar keyword** (`branch`, `foreach`, `while`, `switch`,
`cast`, `select`, `make`, `break`, `make_array`, `enum`, `timeline`,
`call_dispatcher`, `bind_dispatcher`, `subsystem`, `field_notify_*`,
etc.). Two opcodes — `call` and `pure` — wire `UK2Node_CallFunction`,
and the existing `call` opcode already accepts a class-name fallback
(`call K2Node_<Type>(...)`) that routes to
`CodeNodeEmitter::CreateGenericK2Node` when the function name slot
starts with `K2Node_`.

The original ticket proposed a new `generic` opcode keyword. We
dropped that surface. The verb mismatch (`call K2Node_<Type>` does
not really "call" anything) is cosmetic; a new opcode would mean a
new tokenizer entry, two new parser-dispatch entries, and a
back-compat shim for emitted text. The real gap was missing UPROPERTY
round-trip on the `K2Node_<Type>` fallback path. We extend the
existing `call` opcode to accept a trailing `node_props { ... }`
block instead.

## Surface

```bpir
call K2Node_PlayAnimationTimeRange(
    Target: %n0,
    Asset: $MontageAsset,
    StartTime: 0.5,
    EndTime: 1.2,
) node_props {
    bSomeFlag: true,
    SomeStructField: { X: 1, Y: 2 },
} [
    Completed -> @done,
    Interrupted -> @on_interrupt,
]
```

- The function-name slot must start with `K2Node_` or `UK2Node_` for
  the `node_props` block to be accepted; on any other call it is a
  parse error.
- Args are named pin assignments, identical to existing `call` form.
- `node_props { ... }` is an optional sub-block carrying UPROPERTY
  values to replay onto the constructed node via FProperty
  reflection. Must run **before** `AllocateDefaultPins()` for
  shape-determining properties — see
  `B-bpir-compile-property-vs-allocate-pins-ordering`.
- Exec targets follow the existing `[Name -> @label, ...]` form.
- Multi-line `node_props` is deferred — single-line covers the
  emitter output (`BuildSparsePropertyDiffJson` sorts deterministically).

## Implementation touchpoints

1. **Compiler helper extraction** (`BpirCompiler.cpp`): the
   `K2Node_<Type>` fallback inside `EmitInstruction` has been
   extracted into a private member `EmitGenericK2NodeInstruction`.
   AsyncAction routing (`K2Node_AsyncAction_<Factory>` route + bare
   `K2Node_AsyncAction` reject) lives at the start of the helper so
   any K2Node-prefixed instruction goes through one entry point. The
   helper calls `BpirShapeMetadata::ReplayGenericNodeProps` between
   construction and exec wiring, and `BpirShapeMetadata::RunPostWireHooks`
   after pin wiring (both supplied by
   `B-bpir-compile-property-vs-allocate-pins-ordering`).
2. **Decompiler emit** moved to umbrella ticket
   `B-bpir-decompiler-emitter-coverage-gap`.

## Acceptance

- Any `UK2Node` subclass in the engine + loaded plugins, with no
  bespoke compiler/decompiler entry, decompiles via the generic
  emitter and round-trips through `generic K2Node_<Type>(...)`
  back to an equivalent node.
- All existing sugar keywords still work (the change is purely
  additive — sugar keywords are now optional shorthand).
- The generic resolver **does not** override existing per-class
  handlers. Specifically, the resolver dispatches in this order
  before falling through to `FindObject<UClass>`:
  1. If `<Type>` starts with `K2Node_AsyncAction_` → route to the
     existing async-action handler (`BpirCompiler.cpp:3998-4005`),
     never construct a bare `UK2Node_AsyncAction` via FindObject.
     There is no `UClass` named
     `K2Node_AsyncAction_<FactoryName>`; the suffix is a synthetic
     load-bearing string the async-action handler decodes.
  2. If `<Type>` is `K2Node_AsyncAction` or `UK2Node_AsyncAction` →
     reject with the existing error (`BpirCompiler.cpp:3989-3990`)
     directing to the `K2Node_AsyncAction_<Factory>` form. Do not
     fall through to FindObject. This preserves the deterministic
     factory-identity contract from `B-bpir-asyncaction-decompile-factory-identity`.
  3. Add similar deny/route entries for any future class whose
     synthetic-name convention diverges from a real `UClass` name.
- The shape-determining property ordering rules in
  `B-bpir-compile-property-vs-allocate-pins-ordering` apply. In
  particular, `UK2Node_AsyncAction` requires
  `InitializeProxyFromFunction` before `AllocateDefaultPins`; the
  generic resolver must never construct one without that step
  (which is why bare `K2Node_AsyncAction` is rejected above).

## History
- `#1-initial-spec` `OPEN` reporter — Implementation sub-ticket of `B-bpir-decompiler-emitter-coverage-gap`. Concrete parser/tokenizer/compiler surface change derived from a 5-agent audit of BpirParser/BpirTokenizer/BpirCompiler/BpirDecompiler/CodeNodeEmitter. Sugar keywords stay; this adds the universal escape hatch.
- `#2-asyncaction-routing-acceptance` `OPEN` reporter — Added explicit acceptance criteria to dispatch through the existing async-action handler before generic FindObject, preserving the factory-identity contract from `B-bpir-asyncaction-decompile-factory-identity` (IN-REVIEW). Bare `K2Node_AsyncAction` stays rejected; `K2Node_AsyncAction_<Factory>` routes to the dedicated handler. Cross-references the property-ordering rule in `B-bpir-compile-property-vs-allocate-pins-ordering`.
- `#3-implemented-as-call-extension` `IN-REVIEW` developer — Dropped the proposed `generic` keyword. Extended existing `call` opcode to accept `node_props { Key: Value, ... }` between args and exec clause when function-name slot starts with `K2Node_`. SmartSplit gained `{}` depth tracking. `EmitGenericK2NodeInstruction` extracted from the existing K2Node fallback so property-ordering ticket can hook in. AsyncAction routing preserved. Multi-line `node_props` deferred.
- `#4-verify-parser-accepts-node-props` `DONE` tester — Verified: `blueprint.compile_bpir` on a temp Actor BP accepted `call K2Node_Knot() node_props { }` and `%arr = call K2Node_MakeArray() node_props { NumInputs: 5 }` with `errors: []` and a node entry in `createdNodes`. Pre-fix this surface produced `Unrecognized instruction: ) node_props {` followed by per-line errors (see PDS.log lines 2247/2327). Round-trip decompile not exercised because dangling test nodes were dropped by decompile; parser/compiler acceptance of the new form confirmed.
