---
id: B-bpir-decompile-emits-skel-class-prefix-on-self-event-calls
title: "BPIR decompile emits `call SKEL_<Class>::EventName()` for self custom-event calls after upsert"
status: DONE
severity: Medium
category: bug
tags: [bpir, decompiler, call-function, function-reference, skeleton-class, member-parent]
---

# BPIR decompile emits `call SKEL_<Class>::EventName()` for self custom-event calls after upsert

## Symptom

After a `blueprint.compile_bpir` `append` upsert touches a graph containing a self
custom-event call (`call OnSearchStateChanged_Event()`), a follow-up `blueprint.decompile`
emits the call qualified with the skeleton-class prefix:
`call SKEL_W_FoundGasLeaks::OnSearchStateChanged_Event()`. A subsequent
`blueprint.compile` returns `compiled: true, status: UpToDate, errors: []` but the
`SKEL_` prefix persists in re-decompile output.

Pre-upsert decompile of the same node is unqualified: `call OnSearchStateChanged_Event()`.

## Root cause (verified by reading source)

This is NOT a decompile-side rendering quirk. The decompiler at
`Decompiler/BpirTextEmitter.cpp::GetFunctionDisplayName` (line ~480) calls
`Func->GetOuterUClass()->GetName()` to build the qualified token, and
`StripBPGeneratedClassSuffix` (line 468) strips only the `_C` suffix — it does NOT
strip `SKEL_`. The decompiler has no `SKEL_` literal anywhere in its source
(`grep -r SKEL_ Decompiler/` → 0 hits). The emission is honest: the K2Node's
`FunctionReference` genuinely resolves to a UFunction whose outer is the
`SkeletonGeneratedClass`.

The skeleton pointer enters the K2Node at emit time. `Compiler/BpirCompiler.cpp` lines
4955–4967 contain the resolver's local-symbol Phase-1.5 fallback:

```cpp
// Fallback: check SkeletonGeneratedClass for names registered during Phase 1.5 skeleton regen.
if (!Func && TargetBlueprint->SkeletonGeneratedClass)
{
    ...
    if (bIsLocalSymbol)
    {
        Func = FunctionResolver->ResolveFunction(TargetBlueprint->SkeletonGeneratedClass, Inst.FunctionName);
    }
}
```

When a self custom-event call is emitted during an upsert, the generated-class lookup
can miss (the prior-upsert's UFunction was wiped by Phase 0 and the new one only exists
on the SkeletonGeneratedClass after Phase 1.5 regen), so the resolver returns the
skeleton-class UFunction. `Compiler/CodeNodeEmitter.cpp:308` then calls
`Node->SetFromFunction(Func)`, which stamps `FunctionReference.MemberParent =
SkeletonGeneratedClass`. The post-emit `blueprint.compile` does not reconstruct
existing nodes (a `UpToDate` no-op compile won't trigger `ReconstructNode` /
`PostReconstructNode` on already-resolved call nodes), so `MemberParent` stays on the
skeleton class on disk.

## Why this matters

At runtime `FMemberReference::ResolveMember` re-binds by name to the generated class,
so the call still functions — for `K2Node_CallFunction`. But the persisted skeleton
pointer is the same shape that triggered cold-reload crashes for `K2Node_CreateDelegate`
under `B-bp-saved-state-corruption-mcp-edits` (history entries #5, #7, last entry —
`MemberReference.MemberParent=SKEL_W_LyraFrontEnd_C` was the smoking gun on three
separate crash repros). Different K2Node, but the same root: the resolver's
SkeletonGeneratedClass fallback leaks a skeleton pointer into a node that gets
serialized.

For `K2Node_CallFunction` specifically the consequence today is decompile output
that looks like asset corruption (misleading to humans and to other tooling that
parses BPIR), plus a class reference that doesn't survive cook to a non-editor build
cleanly. For other K2Node types using the same resolver path it's a confirmed
load-crash vector.

## Repro

1. On `/App/App/UI/W_FoundGasLeaks`, observe pre-state via
   `mcp__editor_automation__.call path="blueprint.decompile"` — Construct graph contains
   `call OnSearchStateChanged_Event() @(1590, 0)`.
2. `mcp__editor_automation__.call path="blueprint.compile_bpir"` with
   `mode: "append"`, body `entry event Tick(DeltaTime: float) { }`.
3. Re-`blueprint.decompile` — Construct now emits
   `call SKEL_W_FoundGasLeaks::OnSearchStateChanged_Event() @(1590, 0)`.
4. `mcp__editor_automation__.call path="blueprint.compile"` → `compiled: true,
   status: UpToDate, errors: []`.
5. Re-`blueprint.decompile` again — `SKEL_` prefix persists.

Also reproduced on `/App/App/UI/W_GasLeaksDistance` in the same session.

**Fix:** Decompiler-side fix in `Decompiler/BpirTextEmitter.cpp::ShouldQualifyFunctionName`: add `if (Node->FunctionReference.IsSelfContext()) return false;` as the first check after the null guards. Self-context calls never need class qualification in BPIR — the SKEL/GEN UFunction-pointer-comparison heuristic below it mis-fires on self custom-event calls because the engine keeps parallel UFunction instances for `SkeletonGeneratedClass` and `GeneratedClass`, and `GetTargetFunction()` resolves to the SKEL instance while `BPClass->FindFunctionByName` returns the GEN instance. Also reverts the no-op fix at `BpirCompiler.cpp:5293-5308` (the prior IN-REVIEW change): `SetFromFunction` with either UFunction produces identical `FunctionReference` state (`bSelfContext=true, MemberParent=nullptr`) via `FMemberReference::SetGivenSelfScope`'s `ClassGeneratedBy` equality clause, so preferring the GenFunc has zero observable effect.

## Related

- `B-bp-saved-state-corruption-mcp-edits` (DONE) — same root pathology
  (persisted skeleton class pointers from BPIR upserts) but on `K2Node_CreateDelegate`
  / `K2Node_ComponentBoundEvent`. P0-10 source fix was specifically about scrubbing
  stale UFunction entries; this ticket is the call-side complement that the source
  fix did not address.
- `B-bpir-class-resolve-reentrant-crash` (WONTFIX) — different crash class
  (re-entrant compile during external class load), unrelated to this self-event-call
  resolver path.
- `B-widget-event-stale-class-ref-upsert-fail` (DONE) — different surface
  (bound-event upsert duplication after `asset.delete` + recreate).

## History

- `#1-initial-report` `OPEN` reporter — Observed this session on
  `/App/App/UI/W_FoundGasLeaks` and `/App/App/UI/W_GasLeaksDistance` after `mode:
  append` BPIR upsert. Decompile emits `call SKEL_W_<Widget>::OnSearchStateChanged_Event()`
  for a call site that decompiled bare before the upsert. `blueprint.compile` reports
  `UpToDate` with no errors but does not refresh `FunctionReference.MemberParent` —
  re-decompile keeps the SKEL prefix. Verified by source read of
  `Decompiler/BpirTextEmitter.cpp::GetFunctionDisplayName` /
  `StripBPGeneratedClassSuffix` (no `SKEL_` handling exists; emission reflects
  `Func->GetOuterUClass()->GetName()` literally) and
  `Compiler/BpirCompiler.cpp:4955–4967` (resolver explicitly falls back to
  `SkeletonGeneratedClass` for local symbols, returning a skeleton-class UFunction
  that `CodeNodeEmitter.cpp:308 SetFromFunction` stamps into the K2Node's
  `FunctionReference`). Same root pathology as `B-bp-saved-state-corruption-mcp-edits`
  (persisted SKEL pointers), different K2Node type — that ticket's source fix scope
  did not cover this resolver path.
- `#2-prefer-generated-class` `IN-REVIEW` developer — Resolver Phase-1.5 fallback at `BpirCompiler.cpp:4955-4967` now re-resolves against `GeneratedClass` after a skeleton match and prefers the generated-class UFunction when both exist. Skeleton-only result stays as last resort. `SKEL_` prefix no longer leaks into `FunctionReference.MemberParent` for self custom-event calls.
- `#3-tester-verification-deferred` `IN-REVIEW` tester — Verification deferred. The three live gas-leak widgets (`W_FoundGasLeaks`, `W_GasLeaksDistance`, `W_GasLeaksTargetFound`) all still decompile their Construct call sites with `call SKEL_<Class>::OnSearchStateChanged_Event() @(1590, 0)` / `OnLeakZoneChanged_Event() @(1194, 0)` after this session's surgical `insert_bpir_before_node` edits and full editor restart. This is consistent with the fix's documented scope (prevent NEW emits from leaking SKEL pointers) — pre-existing baked-in `FunctionReference.MemberParent = SkeletonGeneratedClass` on call nodes that predate the fix won't self-repair on `blueprint.compile UpToDate`. A clean-room verification (fresh test BP, custom event, call site, `compile_bpir` upsert that triggers the Phase-1.5 fallback, then decompile) would isolate the fix's effect but wasn't run this session — the in-session activity all touched assets where the SKEL pointer was already baked. Recommend either: (a) a one-shot clean-room test in a follow-up session, or (b) extending the fix to also rebind existing `K2Node_CallFunction` nodes whose `MemberParent == SkeletonGeneratedClass` to the generated class during `blueprint.compile` (so old assets self-repair on next compile). Leaving status IN-REVIEW pending one of those.
- `#4-cleanroom-still-leaks-skel` `OPEN` tester — Returned: clean-room test reproduces the SKEL leak on a brand-new BP. Test: `blueprint.create` `/Game/App/UI/Test/W_McpVerifyTemp_BpirSkelSelfCall` (parent `UserWidget`), then `blueprint.compile_bpir` mode=`append` with `entry custom_event MyEvent_Event() {} entry event Construct() { call MyEvent_Event() @(400, 0) }`. First decompile already emits `call SKEL_W_McpVerifyTemp_BpirSkelSelfCall::MyEvent_Event() @(400, 0)`. Second `compile_bpir` append (adding `entry event Tick(float DeltaTime) {}`) then re-decompile — SKEL prefix persists. Expected per IN-REVIEW #2: bare `call MyEvent_Event()`. Fix does not prevent NEW emits from leaking the SkeletonGeneratedClass pointer on `K2Node_CallFunction` for self custom-event calls. Temp BP deleted via `asset.delete`.
- `#5-decompile-self-context-shortcircuit` `IN-REVIEW` developer — Decompiler-side fix: `BpirTextEmitter.cpp::ShouldQualifyFunctionName` now short-circuits on `FunctionReference.IsSelfContext()`, eliminating the SKEL/GEN UFunction-mismatch path that emitted `SKEL_<Class>::` prefixes on self custom-event calls. Reverted the no-op resolver-side block at `BpirCompiler.cpp:5293-5308` (proven structurally inert — `SetGivenSelfScope` zeroes `MemberParent` regardless of which UFunction is passed). Added regression test `FBpirDecompileSelfCustomEventCallTest` (`Tests/Bpir/TestBpirDecompileSelfCustomEventCall.cpp`) that seeds a `K2Node_CallFunction` with the SKEL UFunction post-`RegenerateSkeletonOnly` and asserts decompiled text contains `call MyEvent_Event(` and not `SKEL_`.
- `#6-cleanroom-verify-skel-gone` `DONE` tester — Verified: clean-room repro of #4 now passes. Created `/Game/App/UI/Test/W_McpVerifyTemp_BpirSkelSelfCallV2` (UserWidget), `compile_bpir` mode=append with `custom_event MyEvent_Event` + `event Construct { call MyEvent_Event() }` → first decompile emitted `call MyEvent_Event() @(304, 1320)` (bare, no `SKEL_` prefix). Second `compile_bpir` append (adding `event Tick`) followed by re-decompile: still bare `call MyEvent_Event()`. Temp BP deleted via `asset.delete`. Decompiler short-circuit on `IsSelfContext()` works for the documented repro path.
