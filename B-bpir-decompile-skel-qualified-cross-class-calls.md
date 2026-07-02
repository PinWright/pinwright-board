---
id: B-bpir-decompile-skel-qualified-cross-class-calls
title: "Decompiler emits SKEL_<OtherClass>::Func qualifier for cross-Blueprint calls; compile_bpir cannot resolve SKEL_ classes, so output is not re-compilable"
status: OPEN
severity: Medium
category: bug
tags: [bpir, decompiler, compiler, qualified-call, cross-class, skeleton-class, round-trip]
---

# Decompiler emits `SKEL_<OtherClass>::Func` for cross-Blueprint calls; `compile_bpir` cannot resolve `SKEL_*` classes

For a `K2Node_CallFunction` whose target is a *different* widget Blueprint's function, the decompiler emits a
qualified call using the SKELETON class name, e.g. `call SKEL_W_PDSSettingsUtils::DroneEditSaveAvailable(Target: $W_PDSSettingsUtils)`.
Feeding that exact shape back into `compile_bpir` fails: `Unresolved class 'SKEL_W_PDSSettingsUtils' in qualified call ...;
EmitInstruction failed for opcode 0`. This breaks the documented "decompile a graph and adapt the output" workflow.

Root cause (verified by source read): the qualifier is built in `Decompiler/BpirTextEmitter.cpp::GetFunctionDisplayName`
(line ~514) from `Func->GetOuterUClass()->GetName()`, and `StripBPGeneratedClassSuffix` (line ~489) strips only the `_C`
suffix — never the `SKEL_` prefix — so `SKEL_W_PDSSettingsUtils_C` renders as `SKEL_W_PDSSettingsUtils`. This is the
non-self path: `ShouldQualifyFunctionName`'s `IsSelfContext()` short-circuit (line 436, from the DONE self-event sibling)
does not fire for a cross-class target, so the qualifier is honestly emitted — but from the skeleton class object.
On re-compile, `Compiler/BpirCompiler.cpp:5638` calls `ResolveUClass(Inst.TypeArg)`; `Utils/ClassUtils.cpp::ResolveUClass`
(line 100) has no `SKEL_` handling and a bare `SKEL_W_PDSSettingsUtils` matches no script package and no loaded class
(the skeleton's real `GetName()` is `SKEL_..._C`), so it returns null → the hard error at `BpirCompiler.cpp:5641-5644`.

Distinct from `B-bpir-decompile-emits-skel-class-prefix-on-self-event-calls` (DONE): that fix short-circuits **self-context**
calls only (line 436) and its regression tests assert self-calls; the qualified emit here is a legitimately-needed
disambiguator introduced by `B-bpir-target-shadowed-by-self-class` (DONE), so it cannot simply be dropped.

**Workaround:** Strip the qualifier and use the bare name + Target pin — `call DroneEditSaveAvailable(Target: $W_PDSSettingsUtils)` — which compiles fine (and re-decompiles back to the `SKEL_` qualified form again).
**Fix:** Two acceptable directions. (A) Decompiler emits the *authored* class token instead of the skeleton object name: resolve the outer to the generated/authored BP class (or strip a leading `SKEL_` in `StripBPGeneratedClassSuffix`) — but the emitted token must be `ResolveUClass`-resolvable, and BP short names without `_C` currently aren't, so prefer emitting the BP's generated-class name / asset path. (B) Teach `Utils/ClassUtils.cpp::ResolveUClass` to map `SKEL_X`(`_C`) → X's skeleton/generated class. (A) also stops the misleading skeleton name from appearing in decompile output.

## History
- `#1-initial-report` `OPEN` reporter — Filed after `blueprint.decompile` on `/App/App/UI/LobbyAndMenu/W_DroneSelect_EditDrone` (EventGraph) emitted `%n0: bool = call SKEL_W_PDSSettingsUtils::DroneEditSaveAvailable(Target: $W_PDSSettingsUtils)` and `call SKEL_W_DemoNavigationHelper::ShowNotAvailable(Target: $W_DemoNavigationHelper)`. Re-compiling those verbatim in a `widget_event` body → `Line 28: Unresolved class 'SKEL_W_PDSSettingsUtils' in qualified call 'SKEL_W_PDSSettingsUtils::DroneEditSaveAvailable'; Line 28: EmitInstruction failed for opcode 0; Line 29: Could not resolve value '%n0.Available' for pin ''` (the `%n0` error is consequential on the failed emit). Unqualified variants (`call DroneEditSaveAvailable(Target: $W_PDSSettingsUtils)`) compiled successfully in the same session. Verified by source read: `Decompiler/BpirTextEmitter.cpp::GetFunctionDisplayName` (line ~514, `Func->GetOuterUClass()->GetName()`) + `StripBPGeneratedClassSuffix` (line ~489, strips `_C` only, no `SKEL_`); `ShouldQualifyFunctionName` `IsSelfContext()` short-circuit (line 436) is self-only so the cross-class qualifier is emitted; compiler `BpirCompiler.cpp:5638-5644` (`ResolveUClass` → `Unresolved class` on miss) and `Utils/ClassUtils.cpp::ResolveUClass` (line 100, no `SKEL_` handling; bare `SKEL_*` matches no script package / no loaded class). New sibling to the DONE self-event item, scoped to cross-class (non-self) targets — that fix's self-context short-circuit and regression tests do not cover this path. Severity Medium: loud (not silent) compile failure with a clean documented workaround (soft blocker per rubric).
