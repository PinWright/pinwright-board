---
id: E-bpir-forward-ref-functions
title: "BPIR rejects forward-reference to user-defined functions in the same compile body"
status: DONE
severity: Low
category: ergonomic
tags: [bpir, forward-reference, function-declaration-order, two-pass]
---

# BPIR rejects forward-reference to user-defined functions in the same compile body

If a BPIR body defines a function `F1` that calls a function `F2`, and `F2` is defined later in the same `compile_bpir` body, the compile fails with:

```
COMPILE_FAILED: Line N: Unresolved function: 'F2'. Searched:
  - Blueprint class: W_MyBP_C
  - UKismetMathLibrary
  - UKismetSystemLibrary
  ...
  - Broad search: 797 static functions from non-library classes
Hint: If this is a member function of another class, pass the object as 'Target: %ref' or 'Target: $var'. ...
```

BPIR is single-pass and resolves `call F2(Target: self)` by looking up `F2` on the BP class — but `F2` hasn't been emitted yet in this compile, so lookup fails.

## Repro (observed this session)

Tried to author `W_MyReplaySelect` event graph in one BPIR call:

```
entry override BP_OnActivated() {
    bind_dispatcher OnStateChanged(target: self, event: @HandleStateChanged)
    call Refresh(Target: self)
}

entry custom_event HandleStateChanged(EReplayListState NewState) {
    call RebuildList(Target: self)   // <-- RebuildList declared later, below
}

entry function RebuildList() {
    ...
}
```

Result: `Unresolved function: 'RebuildList'` error on the `call RebuildList(Target: self)` line.

**Workaround (session-tested):** Reorder the compile body so callee functions appear before callers:

```
entry function RebuildList() { ... }           // declared first
entry override BP_OnActivated() { ... }        // then the callers
entry custom_event HandleStateChanged(...) { call RebuildList(Target: self) }
```

Or split into two separate `compile_bpir` calls: one to create `RebuildList`, one to create the callers.

**Proposed fix:** The compiler is already phased (Phase 1 creates all entries, Phase 1.5 regenerates skeleton, Phase 2 emits bodies) — forward-ref lookups should succeed via the Phase 2 SkeletonGeneratedClass fallback. The real bug was that fallback's gate (`CreatedCustomEvents.Num() > 0`) excluded compiles that declare only functions/macros. Fix: extend the gate so the SkeletonGeneratedClass fallback also runs when the unresolved name matches a function graph or macro graph created this compile.

## History
- `#1-forward-ref-unresolved-function` `OPEN` reporter — Hit during `W_MyReplaySelect` event-graph authoring. `HandleStateChanged` calling `RebuildList` failed when both were in the same BPIR body with `RebuildList` declared later. Shipped with a reordered body (RebuildList first) in the final compile.
- `#2-fallback-gate-extended` `IN-REVIEW` developer — Reshaped from "two-pass compile" (architectural overkill) to targeted fallback-gate extension. In BpirCompiler.cpp function-resolution fallback, relaxed the `CreatedCustomEvents.Num() > 0` gate to also admit names matching CreatedFunctionGraphs/CreatedMacroGraphs/CreatedCustomEvents — Phase 1.5's skeleton regen has already registered them on SkeletonGeneratedClass by Phase 2 emit time. Pinned by FBpirForwardReferenceFunctionTest in TestBpirForwardReference.cpp.
- `#3-verified-forward-ref-resolves` `DONE` tester — Verified on `/Game/App/UI/Test/W_McpVerifyTemp`. `compile_bpir` on a body where a `custom_event StartFlow` calls `LaterFunction(Target: self)` with `entry function LaterFunction()` declared AFTER the custom event: response `compiled: true, status: "UpToDate", errors: [], nodeCount: 3`. Previously the same order would fail with `Unresolved function: 'LaterFunction'`; forward reference now resolves via the Phase 2 SkeletonGeneratedClass fallback.
