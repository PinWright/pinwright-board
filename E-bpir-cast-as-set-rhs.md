---
id: E-bpir-cast-as-set-rhs
title: "BPIR cast<T>(...) cannot be used as an RHS expression in a set statement"
status: DONE
severity: Medium
category: ergonomic
tags: [bpir, compiler, cast, set, expression]
---

# `set Var = cast<T>(...)` not supported

BPIR allows `cast<T>(expr)` only as a top-level statement that yields
a named `%result` with exec branches (`[success -> @ok, fail -> @bad]`).
It cannot appear inline as the RHS of a `set` statement:

```
set GI.ForcedDrone = cast<UDroneDefinition>($UserFacingExp.ForcedDrone)
```

fails with:

```
COMPILE_FAILED: Could not resolve value 'cast<UDroneDefinition>($UserFacingExp.ForcedDrone)' for pin ''
```

The forced workaround is a separate cast statement with explicit
success/fail labels, both pointing to the next code block:

```
%nDrone = cast<UDroneDefinition>($UserFacingExp.ForcedDrone) [success -> @okDrone, fail -> @okDrone]

@okDrone:
    set GI.ForcedDrone = %nDrone.AsDroneDefinition
```

This is bloated — three extra lines plus a label — for a common idiom:
"downcast an object reference before assigning it to a strongly-typed
field." Common when assigning a `UPrimaryDataAsset*` field (with
AllowedClasses metadata) into a strongly-typed `T*` field, which is
how the project handles cross-module data references (the source type
in PDSGame, the destination type in plugin code).

Different idiom from `E-bpir-set-on-createwidget-needs-cast` (DONE),
which was about `set %ref.Prop = value` where `%ref` is the output of
a typed `K2Node_CreateWidget`. That fix recognized the constructor's
class parameter as the authoritative type. This ticket is about
literal `cast<T>(...)` in expression position.

## Repro

```
entry function Apply(object<UPrimaryDataAsset> Source) {
    set $TypedField = cast<UDroneDefinition>($Source)
}
```

`COMPILE_FAILED: Could not resolve value 'cast<UDroneDefinition>($Source)' for pin ''`

## Workaround

Split into a two-statement form with success/fail labels:

```
entry function Apply(object<UPrimaryDataAsset> Source) {
    %typed = cast<UDroneDefinition>($Source) [success -> @apply, fail -> @apply]

@apply:
    set $TypedField = %typed.AsDroneDefinition
}
```

Works, but every "cast and assign" idiom doubles in length.

## Proposed fix

The parser's value-resolution path on `set` RHS should recognize
`cast<T>(expr)` as an expression and emit a pure (impure if exec needed)
`K2Node_DynamicCast` with `bIsPureCast=true`, route the result pin to
the set node's input. UE's BP schema supports pure casts natively; the
generated graph would be visually equivalent to a manually-drawn pure
cast feeding the variable set.

On a fail (cast returns null), the set still proceeds with null — same
semantic as the explicit two-statement workaround that converges both
labels to the same block. No exec branching is needed for the common
case.

If the cast must remain impure for some reason, the parser could emit
the cast on its own line as a synthetic helper, with both
success/fail wires routed to the same following statement — essentially
the workaround generated automatically.

## History
- `#1-initial-repro` `OPEN` reporter — Session needed to assign `UFED.ForcedDrone` (typed `TObjectPtr<UPrimaryDataAsset>` with `meta=(AllowedClasses="/Script/App.DroneDefinition")`) into `GI.ForcedDrone` (typed `TObjectPtr<UDroneDefinition>`). Inline form `set GI.ForcedDrone = cast<UDroneDefinition>($UserFacingExp.ForcedDrone)` failed; required restructuring to a separate cast statement with success/fail labels both converging on the next block. Three extra lines + a label for a one-line assignment idiom.
- `#2-pure-cast-value-resolver-branch` `IN-REVIEW` developer — Added cast<T>(...) branch in BpirValueResolver::ResolveValue that emits a pure K2Node_DynamicCast (SetPurity(true)) and returns its result pin; extended BpirCompiler::PreEmitVariableRefs to recurse into cast<...> arguments so inner $params pre-emit. New regression test TestBpirSetCastRhs.cpp asserts exactly one pure dynamic-cast node wired between entry param and the variable set.
- `#3-fail-bp-compile-undetermined-object` `OPEN` tester — Returned: BPIR parser accepts `set $TypedRef = cast<Actor>($Source)` (no more "Could not resolve value 'cast<...>'") but UE's Blueprint compile then fails with "The type of Object is undetermined. Connect something to Cast To Actor to imply a specific type." — the cast node's source/input pin isn't being wired to the inner `$Source` entry parameter (or to a member-var `$SourceObj`). Test: `blueprint.compile_bpir` with the exact form from TestBpirSetCastRhs.cpp (`entry function Apply(object<Object> Source) { set $TypedRef = cast<Actor>($Source) }`) on an Actor-parented BP with a TypedRef:object<Actor> member; also reproduced with a member-var source. The in-process automation test passes only because it asserts graph topology and never re-runs `FKismetEditorUtilities::CompileBlueprint` on the resulting graph; the wildcard input-pin type isn't propagating, so the UE-side BP compile rejects it.
- `#4-schema-cast-source-link` `IN-REVIEW` developer — Changed FBpirValueResolver::ResolveCastExpression to connect inline pure-cast source pins through the K2 schema so UK2Node_DynamicCast resolves its wildcard Object pin before full Blueprint compile; extended TestBpirSetCastRhs.cpp to assert the source pin type is resolved and CompileBlueprintWithDiagnostics succeeds.
- `#5-verify-fix` `DONE` tester — Verified on Actor-parented temp BP /Game/App/UI/Test/W_McpVerifyTemp_E_bpir_cast_as_set_rhs with TypedRef:object<Actor> member. `blueprint.compile_bpir` with `entry function Apply(object<Object> Source) { set $TypedRef = cast<Actor>($Source) }` returned compiled=true, status=UpToDate, errors=[], nodeCount=2. Follow-up `blueprint.compile` also returned compiled=true, status=UpToDate, errors=[] — the "type of Object is undetermined" symptom from #3 no longer reproduces. Temp BP deleted.
