---
id: B-bpir-subsystem-getter-roundtrip
title: "BPIR decompile emits subsystem getter as free function; compile only accepts subsystem<T>() keyword"
status: DONE
severity: High
category: bug
tags: [bpir, decompiler, compiler, round-trip, subsystem]
---

# BPIR decompile emits subsystem getter as free function; compile only accepts subsystem<T>() keyword

`blueprint.decompile_function` emits `UK2Node_GetSubsystem` (and its
`*FromPC`, `*EditorSubsystem`, `*EngineSubsystem` siblings) as a generic
free-function call:

```
%n0: object<ReplaySaveWorldSubsystem> = call Get_ReplaySaveWorldSubsystem() @(...)
```

The compiler does not have a `Get_<TypeName>` UFunction to resolve — those
identifiers are synthetic node names UE assigns to typed subsystem getter
nodes. Feeding the decompiled text back into `blueprint.compile_bpir`
fails with:

```
COMPILE_FAILED: Unresolved function: 'Get_ReplaySaveWorldSubsystem'.
```

The only form the compiler accepts is the documented BPIR keyword
(`bpir-language-reference.md` §2.13, around line 612):

```
%ss = subsystem<ReplaySaveWorldSubsystem>()
```

So decompile and compile disagree on the surface for subsystem getters,
and any BPIR text that uses a subsystem getter cannot round-trip
through decompile → compile.

## Repro

1. `blueprint.decompile_function` on
   `/App/App/UI/HUD/W_HUD_RaceTrackEnd::RefreshAnalyzerButtonVisibility`
   (or any widget that calls into `ReplaySaveWorldSubsystem`,
   `RaceAnalyzerSubsystem`, `TrackUISubsystem`, `DroneGameInstance`, …).
2. Observe `call Get_ReplaySaveWorldSubsystem()` (etc.) in the BPIR.
3. Feed the same string back into `blueprint.compile_bpir`.
4. `COMPILE_FAILED: Unresolved function: 'Get_<SubsystemType>'.`

Same shape as `F-decompile-enum-names` and the async-action / component-
bound-event identity tickets: decompile produces a representation the
compiler can't ingest, breaking the documented decompile → edit → compile
iteration loop.

## Impact

- Decompile → compile retries are the documented BPIR iteration workflow
  (`bpir-language-reference.md`, MCP wiki). This break makes that loop
  unusable for any widget/Blueprint that touches subsystems.
- Subsystem getters are pervasive in PDS CommonUI / Lyra patterns
  (`Get_RaceAnalyzerSubsystem`, `Get_ReplaySaveWorldSubsystem`,
  `Get_TrackUISubsystem`, `Get_DroneGameInstance`, …) — many widgets
  hit this on first attempt.
- The synthetic `Get_<TypeName>` identifier collides with the namespace
  of real UFunctions, so a generic "accept this on compile" workaround
  would be ambiguous if a project ever defines a real
  `UFUNCTION Get_<Name>`.

## Fix

**Recommended:** fix the decompile side. Detect `UK2Node_GetSubsystem`
(and `UK2Node_GetSubsystemFromPC`, `UK2Node_GetEngineSubsystem`,
`UK2Node_GetEditorSubsystem`) before the generic call emitter and emit
the documented keyword form:

```
%n0 = subsystem<ReplaySaveWorldSubsystem>()
```

Read the configured subsystem class off the node (`CustomClass` or the
result pin's `PinSubCategoryObject`, depending on the K2Node subclass)
and render it as the type argument. This matches the keyword the
compiler already implements, keeps the surface single-spelling
(consistent with the design intent on §2.13), and parallels the fixes
in `B-bpir-asyncaction-decompile-factory-identity` and
`B-bpir-component-bound-event-decompile-identity`.

Add a regression test that decompiles a function containing a
subsystem getter, asserts the `subsystem<T>()` form, and round-trips
through compile.

**Alternative considered:** teach the compiler to also accept
`call Get_<TypeName>()` as a synonym. Rejected because (a) it adds a
second spelling for the same construct (drift risk), (b) the
`Get_<TypeName>` identifier is synthetic — there's no real UFunction
behind it — so the compile resolver would need a special case anyway,
(c) it would conflict with any project that defines a real
`UFUNCTION Get_<Name>` and want to call it directly.

**Workaround:** hand-edit decompiled BPIR to replace every
`call Get_<SubsystemType>()` with `subsystem<SubsystemType>()` before
recompile. Tedious and easy to miss.

## History
- `#1-initial-repro` `OPEN` reporter — `blueprint.decompile_function` of
  `W_HUD_RaceTrackEnd::RefreshAnalyzerButtonVisibility` emits
  `call Get_ReplaySaveWorldSubsystem()`. Recompile via
  `blueprint.compile_bpir` fails with `COMPILE_FAILED: Unresolved
  function: 'Get_ReplaySaveWorldSubsystem'`. The compiler only accepts
  `subsystem<ReplaySaveWorldSubsystem>()` per
  `bpir-language-reference.md` §2.13. Breaks decompile → edit → compile
  round-trip for any BP that touches subsystems (pervasive across PDS
  CommonUI / Lyra widgets).
- `#2-emit-subsystem-keyword` `IN-REVIEW` developer — `BpirTextEmitter::EmitPureNode` now recognizes `UK2Node_GetSubsystem` (and all four subclasses via a single base cast) and emits the documented `subsystem<T>()` keyword form, reading the configured subsystem class off the result pin's `PinSubCategoryObject`. Added round-trip regression `EditorAutomationRpcGateway.bpir.round_trip.SubsystemGetter` (compile → decompile → assert `subsystem<` + no `call Get_` → recompile). Resolves the `Get_<TypeName>` decompile/compile mismatch.
- `#3-verified-pass` `DONE` tester — Re-ran `EditorAutomationRpcGateway.bpir.round_trip.SubsystemGetter` against the current build via UnrealEditor-Cmd; `Result={Success}` with no AddError events. The four `Failed to find object 'Class None.GameInstanceSubsystem'` warnings during compile are benign (the compiler resolves the class through its other lookup paths and the compile succeeds; the test only short-circuits on `!CompileResult.bSuccess`). The `subsystem<T>()` decompile and the duplicate-plain-entry compile-side check are both behaving as designed for this test.
