---
id: B-bpir-decompile-skel-qualified-cross-class-calls
title: "Decompiler spuriously qualifies cross-Blueprint calls (SKEL/GEN UFunction-instance mismatch) and emits an unresolvable SKEL_<OtherClass>::Func token, breaking round-trip"
status: OPEN
severity: Medium
category: bug
lastSeen: 2026-09-05T18:09:22Z
encounters: 2
costly: 1
tags: [bpir, decompiler, compiler, qualified-call, cross-class, skeleton-class, round-trip]
---

# Decompiler spuriously qualifies cross-Blueprint calls and emits an unresolvable `SKEL_<OtherClass>::Func` token

For a `K2Node_CallFunction` whose target is a *different* Blueprint's function (an explicit `Target:` pin,
non-self-context), the decompiler emits a class-qualified call using the **skeleton** class name, e.g.
`call SKEL_W_PDSSettingsUtils::DroneEditSaveAvailable(Target: $W_PDSSettingsUtils)`. Feeding that exact shape back into
`compile_bpir` fails: `Unresolved class 'SKEL_W_PDSSettingsUtils' in qualified call ...; EmitInstruction failed for opcode 0`.
This breaks the documented "decompile a graph and adapt the output" round-trip.

**Root cause (verified against source at HEAD).** The qualification is a **false positive** caused by a
skeleton-vs-generated `UFunction`-instance mismatch — the same pathology the DONE self-event sibling
(`B-bpir-decompile-emits-skel-class-prefix-on-self-event-calls`) fixed for self-context, but here on the **cross-class
(non-self) arm** that fix does not cover:
- `Decompiler/BpirTextEmitter.cpp::ShouldQualifyFunctionName` returns `Found != BoundFunc` (lines ~465-479), where
  `BoundFunc = Node->GetTargetFunction()` is the **SkeletonGeneratedClass** copy the resolver handed back, while
  `Found = TargetClass->FindFunctionByName(...)` is the **GeneratedClass** copy. These are two DISTINCT `UFunction`
  objects for the SAME authored function, so the raw pointer `!=` spuriously reports a name collision and qualifies.
- `GetFunctionDisplayName` (line ~514) then reads `Func->GetOuterUClass()->GetName()` = `SKEL_<Class>_C`, and
  `StripBPGeneratedClassSuffix` (line ~489) chops only `_C` (never `SKEL_`), so the emitted token is the bare
  `SKEL_<Class>`.
- On re-compile, `Compiler/BpirCompiler.cpp:5635-5644` calls `ResolveUClass(Inst.TypeArg)` and hard-errors on null;
  `Utils/ClassUtils.cpp::ResolveUClass` (lines 100-271) has no `SKEL_` handling and its only bare-BP-shortname path
  (step 7, line ~236) requires a trailing `_C`, so `SKEL_W_PDSSettingsUtils` resolves nowhere → the hard error.

**Not just a SKEL_ cosmetic issue.** The qualifier here is genuinely *not needed*: the ticket's own workaround (bare name
+ `Target:` pin — `call DroneEditSaveAvailable(Target: $W_PDSSettingsUtils)`) compiles, because DONE
`B-bpir-target-shadowed-by-self-class` (Fix 1) reordered the compiler cascade to run target-class lookup first, so the
`Target:` pin already disambiguates. The over-qualification is the sole defect. (A broader, latent edge also exists: for a
*genuine* same-name/different-function collision whose bound func's outer is a generated class `W_Foo_C`, the emitter
would still emit the bare `W_Foo`, which `ResolveUClass` also can't resolve — see below. That genuine-collision case is
rare/contrived and out of scope for this ticket; this ticket is the false-positive SKEL_ case the reporter hit.)

**Distinct from siblings (both DONE, both present in HEAD):** `B-bpir-decompile-emits-skel-class-prefix-on-self-event-calls`
fixed the SELF arm only (the `IsSelfContext()` short-circuit at line 436; its regression tests assert self-calls).
`B-bpir-target-shadowed-by-self-class` introduced the qualified `Class::Method` emit + `ResolveUClass` compile path but
only ever round-trip-verified a NATIVE token (`KismetSystemLibrary::PrintString`), never a BP-to-BP skeleton token.
Neither covers this cross-class arm.

**Workaround:** Strip the qualifier and use the bare name + Target pin — `call DroneEditSaveAvailable(Target: $W_PDSSettingsUtils)` — which compiles fine (and re-decompiles back to the `SKEL_` qualified form again until fixed).

**Fix (implemented):** Extend the self-event sibling's suppression to the cross-class arm — in
`Decompiler/BpirTextEmitter.cpp::ShouldQualifyFunctionName`, compare authored-function **identity** rather than raw
`UFunction` pointers: normalize both the bound function and the cascade-`Found` function to their `GeneratedClass`
instance (via `OwnerClass->ClassGeneratedBy` → `UBlueprint::GeneratedClass`) before the `!=` test. A SKEL BoundFunc and
a GEN Found of the same authored function then compare equal → the call emits its bare, round-trippable name, and the
misleading `SKEL_` token never appears. This needs **no** `ResolveUClass` change. (Rejected the reporter's original
option B — "teach `ResolveUClass` to accept `SKEL_X`" — as the wrong layer: `SKEL_` is an editor-transient/corruption
artifact that should never appear in authored BPIR; and rejected the "keep the qualifier but make its token resolvable"
direction as unnecessary for the reported case, since the qualifier itself is the false positive.)

## History
- `#1-initial-report` `OPEN` reporter — Filed after `blueprint.decompile` on `/App/App/UI/LobbyAndMenu/W_DroneSelect_EditDrone` (EventGraph) emitted `%n0: bool = call SKEL_W_PDSSettingsUtils::DroneEditSaveAvailable(Target: $W_PDSSettingsUtils)` and `call SKEL_W_DemoNavigationHelper::ShowNotAvailable(Target: $W_DemoNavigationHelper)`. Re-compiling those verbatim in a `widget_event` body → `Line 28: Unresolved class 'SKEL_W_PDSSettingsUtils' in qualified call 'SKEL_W_PDSSettingsUtils::DroneEditSaveAvailable'; Line 28: EmitInstruction failed for opcode 0; Line 29: Could not resolve value '%n0.Available' for pin ''` (the `%n0` error is consequential on the failed emit). Unqualified variants (`call DroneEditSaveAvailable(Target: $W_PDSSettingsUtils)`) compiled successfully in the same session. Verified by source read: `Decompiler/BpirTextEmitter.cpp::GetFunctionDisplayName` (line ~514, `Func->GetOuterUClass()->GetName()`) + `StripBPGeneratedClassSuffix` (line ~489, strips `_C` only, no `SKEL_`); `ShouldQualifyFunctionName` `IsSelfContext()` short-circuit (line 436) is self-only so the cross-class qualifier is emitted; compiler `BpirCompiler.cpp:5638-5644` (`ResolveUClass` → `Unresolved class` on miss) and `Utils/ClassUtils.cpp::ResolveUClass` (line 100, no `SKEL_` handling; bare `SKEL_*` matches no script package / no loaded class). New sibling to the DONE self-event item, scoped to cross-class (non-self) targets — that fix's self-context short-circuit and regression tests do not cover this path. Severity Medium: loud (not silent) compile failure with a clean documented workaround (soft blocker per rubric).
- `#2-reword-and-fix` `IN-REVIEW` developer — Reworded to reflect the verified root cause and correct the original framing. The cross-class qualifier is a **false positive**, NOT a "legitimately-needed disambiguator that cannot be dropped" as the original body claimed: `ShouldQualifyFunctionName` returns `Found != BoundFunc` only because `BoundFunc` is the SkeletonGeneratedClass instance while `Found = TargetClass->FindFunctionByName` is the GeneratedClass instance of the SAME authored function (same SKEL/GEN pointer-mismatch pathology as DONE `B-bpir-decompile-emits-skel-class-prefix-on-self-event-calls`, on the non-self arm it left uncovered) — proven by the ticket's own working workaround (bare name + `Target:` pin compiles via the DONE `B-bpir-target-shadowed-by-self-class` target-first cascade). Fix: `Source/PinWright/Private/Decompiler/BpirTextEmitter.cpp::ShouldQualifyFunctionName` now normalizes both the bound and cascade-found `UFunction` to their `GeneratedClass` instance before the identity compare (`IsSameAuthoredFunction`/`NormalizeToGeneratedFunction` locals), so the SKEL-vs-GEN duplication no longer qualifies; only a genuine same-name/different-function collision qualifies. No `ResolveUClass` change (rejected original option B as the wrong layer). Regression test `PinWright.bpir.decompiler.CrossClassCallNoSkelPrefix` (`Source/PinWright/Private/Tests/Bpir/TestBpirDecompileCrossClassCallNoSkelPrefix.cpp`) builds two transient orphan Blueprints in-code (Target owns custom event `CrossFoo`; Self hosts the call), binds the call node to Target's SKELETON `CrossFoo`, links a Target-GeneratedClass-typed DynamicCast result into the self/target pin, then asserts `GetFunctionDisplayName` emits no `SKEL_`, no `::`, and the bare `CrossFoo`; fixture-validity gates assert non-self-context + SKELETON-instance binding so a green result can't be vacuous. Left OUT of scope (noted in body): the rare genuine-collision case where a resolvable qualified token would still be desirable.

- `#3-returned` `OPEN` reporter - **Returned to OPEN: the fix does not cover the `K2Node_Message` path, and that path is every Blueprint-interface call.** Found on the FPS build after the wave 4-9 rebuild, 2026-09-05, while verifying `B-bpir-interface-call-never-dispatches`.

  Symptom is this ticket's, on a node class its fix does not reach: you author `message BPI_X_C::Func(...)` and the decompiler returns `message SKEL_BPI_X_C::Func(...)`. Three reproductions, two of them on assets no agent in this project authored:

```
authored probe        message BPI_PWProbe_Msg_C::ProbePing(...)  ->  SKEL_BPI_PWProbe_Msg_C::
BP_Button_Interface                                              ->  SKEL_BPI_Player_Interactions_C::
BP_KioskButton                                                   ->  SKEL_BPInterface_Button_C::
```

  Mechanism, read from source rather than inferred: `BpirTextEmitter.cpp::ShouldQualifyFunctionName` returns `true` for `UK2Node_Message` **at the top of the function, ahead of** the `NormalizeToGeneratedFunction` helper that `#2` added for the `K2Node_CallFunction` case. `GetFunctionDisplayName` then prints `Func->GetOuterUClass()->GetName()` verbatim for message nodes with no owner-class normalization, and `FMemberReference` hands back the skeleton copy in the editor. So `#2`'s fix is correct and simply sits behind an earlier return on this branch.

  **Milder blast radius than `#1`'s case, and the difference matters for triage.** This text *does* recompile in-session: `ResolveUClass` step 3 (`ClassUtils.cpp:92-118`) matches the loaded SKEL class by exact short name, because `_C` is preserved here rather than stripped. It is not durable across a session where the interface is not loaded, and it does **not** reach the asset — the saved `.uasset` carries `BPI_PWProbe_Msg` 3x and `SKEL_BPI_PWProbe_Msg` 0x, the only `SKEL_` string being the Blueprint's own generated name. So: decompiled-text fidelity defect, not corruption, and not the unresolvable-token failure `#1` described.

  Why no test caught it: `TestBpirInterfaceMessageRoundTrip.cpp:49` pins its fixture to the native `/Script/UMG.UserListEntry`, which has no skeleton twin. Every Blueprint interface takes the untested path, which is all of them in this project.

  **Full evidence, including the `get_nodes` / `get_execution_flow` output and the byte counts, is durable in the `#4-verified-in-fps-build-with-one-new-defect` entry on `B-bpir-interface-call-never-dispatches`** (commit `7eb5465`); it is not duplicated here. Fix direction: move the `UK2Node_Message` branch behind `NormalizeToGeneratedFunction` so it takes `#2`'s existing normalization, and add a Blueprint-interface fixture to the message round-trip test.
