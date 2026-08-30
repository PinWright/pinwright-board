---
id: F-bpir-emit-pin-types
title: "BPIR emitter drops pin types on intermediate `%n` registers, forcing readers to infer types"
status: DONE
severity: Medium
category: feature
tags: [bpir, decompiler, emitter, readability, llm-ergonomics, asset-dump]
---

# BPIR emitter drops pin types on intermediate `%n` registers, forcing readers to infer types

Decompiled BPIR (the same text written into `bpir.txt` sidecars by
`asset.dump`) names every intermediate node output `%n0`, `%n1`, … but
emits **no type information** on those registers or on call return values.
The output is syntactically parseable but every reader (human or LLM)
has to look up each function on the side to know what `%nK` is.

## Concrete examples from real dumps

`.editor-automation/asset-dumps/App/App/UI/LobbyAndMenu/Editor/W_ProgressBarEditor/bpir.txt`:

```
entry override OnMouseEnter(struct<Geometry> MyGeometry, struct<PointerEvent> MouseEvent) @(0, 0) {
    %n0 = call IsOwningPlayerUsingTouch() @(294, 120)
    %n1 = branch(%n0) [false -> @else] @(294, 0)
@else:
    call PlayAnimationForward(InAnimation: $OnHover, ...) @(634, 17)
}

entry widget_event Slider_SettingValue.OnValueChanged(float Value) @(0, 937) {
    %n0 = sequence(2) [0 -> @s0, 1 -> @s1] @(401, 937)
@s0:
    %n1 = call GetDynamicMaterial(Target: $ProgressBar) @(707, 922)
    %n2 = call Subtract_DoubleDouble(A: $Value, B: $MinValue) @(1181, 1073)
    %n3 = call Divide_DoubleDouble(A: %n2, B: $MaxValue) @(1364, 1073)
```

`%n1` is `UMaterialInstanceDynamic*`, `%n2`/`%n3` are `double`, `%n0` (the
boolean) and `%n0` (the sequence handle) are reused names for completely
different types — nothing in the text tells you that.

`.editor-automation/asset-dumps/App/App/LevelBlueprints/B_DronePlayerController/bpir.txt`:

```
entry event BeginPlay() @(0, 16) {
    %n0 = call GetDroneGameInstance() @(260, 16)
    call TurnDLSS(Target: %n0.DroneGameInstance, isTurnOn: true) @(574, 0)
    %n1 = call GetGameUserSettings() @(892, 16)
    %n2 = call GetFullscreenMode(Target: %n1) @(1197, 238)
    %n3 = call Equal(A: %n2, B: EWindowMode::Fullscreen) @(1197, 135)
    set WasFullscreen = %n3 @(1197, 32)
    %n4 = call Get_AppActivationSubsystem() @(1993, 164)
    bind_dispatcher OnAppActivationChanged(Target: %n4, event: @OnAppActivationChanged_Event) @(1993, 16)
}
```

Reader must know that `%n0` is `UDroneGameInstance*`, `%n1` is
`UGameUserSettings*`, `%n2` is `EWindowMode` (PC_Byte enum), `%n3` is
`bool`, `%n4` is `UAppActivationSubsystem*` — none of which is in the
text.

`.editor-automation/asset-dumps/App/App/UI/LobbyAndMenu/Elements/Buttons/W_AnimatedButton/bpir.txt`:

```
entry override BP_OnPressed() @(0, 2752) {
    %n0 = branch($IsDisabled) [false -> @else] @(208, 2752)
@else:
    %n1 = call GetAnimationCurrentTime(InAnimation: $OnPressed) @(608, 2928)
    %n2 = call K2Node_PlayAnimationTimeRange(Widget: self, InAnimation: $OnPressed,
        StartAtTime: %n1, EndAtTime: 0.0, ..., PlayMode: EUMGSequencePlayMode::Forward, ...)
}
```

`%n1` is a `float` (passed to `StartAtTime: float`), `%n2` is the
animation handle — invisible from the text alone.

`.editor-automation/asset-dumps/App/HELIOS/Drones/Icarus/Parent/Icarus_ParentBP/bpir.txt`:

(Body is trivial — picked as low-signal contrast: dumps of large BPs all
exhibit the same pattern regardless of complexity.)

## Why this hurts

The asset-dumps cache exists so agents don't have to call live MCP
analysis tools to understand a Blueprint's logic. Without type info on
`%n` registers, the cache is half-useful: readers must still call
`blueprint.inspect` / `asset.dump` of the callee function to learn the
return type, or trace through to a `set $TypedVar = %nK` line where the
sink reveals the source's type. That defeats the cache's purpose for
any non-trivial graph.

## Root cause

The BPIR language reference (`docs/bpir-language-reference.md`) defines
type syntax only for **entry-point signatures** (`-> float`, parameter
types) and **type-parameterised instructions** (`cast<T>`, `make<T>`,
`break<T>`, `switch_enum<T>`, `struct<T>`, `object<T>`, etc.). The body
grammar for `%name = call …`, `%name = branch(…)`, `%name = latent …`,
etc. has **no production for a type annotation** on the register name or
on the call expression.

The emitter (`Source/PinWright/Private/Decompiler/BpirTextEmitter.cpp`,
e.g. `EmitCallNode` at ~line 1061) writes:

```cpp
return FString::Printf(TEXT("%%%s = call %s(%s)"), *ResultName, *FuncName, *AllArgs);
```

— no type token, because the language has nowhere to put one.

The information is available at emit time: every emitter callback
receives the node and its pins, and `FBpirTextEmitter::PinTypeToBpirType(FEdGraphPinType)`
already converts the primary-output pin's `PinType` to a BPIR type
string (it's the same helper that emits entry parameter and return
types). So this is a language-extension request, not a "lost data"
recovery problem.

## Proposal

Extend the BPIR grammar with a non-breaking, **optional, ignored-on-compile**
type-suffix syntax for register assignments. Two candidate forms (pick
whichever is cleaner for the parser):

**Option A — colon-after-register (Rust/TypeScript-style):**
```
%n0: bool = call IsValid(Object: %target)
%n1: object<MaterialInstanceDynamic> = call GetDynamicMaterial(Target: $ProgressBar)
%n2: double = call Subtract_DoubleDouble(A: $Value, B: $MinValue)
```

**Option B — arrow-after-call (Haskell/C++ trailing-return-style):**
```
%n0 = call IsValid(Object: %target) -> bool
%n1 = call GetDynamicMaterial(Target: $ProgressBar) -> object<MaterialInstanceDynamic>
%n2 = call Subtract_DoubleDouble(A: $Value, B: $MinValue) -> double
```

In both options the type is **descriptive only** — the compiler parses
and discards it (or validates against the resolved pin type as a
soft-check). Round-trip with a decompile-emitted annotation is the
golden path; user-written code with stale or omitted annotations still
compiles unchanged. This preserves backwards compatibility with every
existing `compile_bpir` callsite.

Apply the annotation to every node-backed `%name = …` form in §6 of the
language reference (call, latent, branch, switch_*, foreach, while,
sequence, cast, macro, timeline, select, get). For multi-output forms
(`%loop` on `foreach`, `%cast` on cast, etc.) the annotation describes
the *primary* output pin; the `.PinName` accessors continue to carry
their own implicit types from the node.

Also consider annotating call **arguments** that reference `%refs`
when the formatted source is ambiguous — but that's a much larger
diff; the register-binding form alone covers ~80% of the readability
gap.

## Scope

- Parser: add the optional suffix to the BPIR body grammar in
  `Source/PinWright/Private/Compiler/` (likely the
  statement-parsing path that handles `%name = …`). Parse-and-discard
  is sufficient for v1; pin-type cross-validation is a follow-up.
- Emitter: after computing `ResultName` in `EmitCallNode` / `EmitPureNode`
  / `EmitCast` / `EmitLoop` / `EmitBranch` / etc., look up the node's
  primary output pin and append the formatted `PinTypeToBpirType(Pin->PinType)`
  in the chosen syntax.
- Decompiler tests: extend `Tests/Private/Bpir/TestDecompiler.cpp` to
  assert the type appears on representative call/branch/cast/loop
  output lines.
- Round-trip tests: re-compile with the annotation present, verify no
  spurious errors and that the emitted Blueprint matches the
  unannotated baseline.
- Documentation: add the syntax to `docs/bpir-language-reference.md`
  §2 (instructions) and §6 (pure/impure table), with one example each
  for the major instruction kinds.

## Out of scope

- Annotating bare value references (`%n0` used as an argument) — only
  the **binding site** `%n0 = …` gets the suffix; argument sites stay
  unchanged.
- Annotating `$param` / `$variable` accesses — those carry types in the
  entry signature or the BP's variable list and are easy to look up.
- Compiler-side type checking against pin types beyond a soft warning
  on mismatch (could be a follow-up).

**Workaround:** None at the dump-consumer level. Readers must call
`blueprint.inspect` or open the corresponding callee asset dump to
recover the type. The asset-dump cache loses much of its value on
non-trivial graphs.

## History
- `#1-initial-feature-request` `OPEN` reporter — Filed after reviewing four real `bpir.txt` dumps (`W_ProgressBarEditor`, `B_DronePlayerController`, `W_AnimatedButton`, `Icarus_ParentBP`). All four show `%nK = call …` lines with no type info on either side. Confirmed the BPIR language reference defines type syntax only for entry signatures and `<T>`-parameterised instructions — no body-line register-type production exists, so the omission is structural in the language, not just an emitter dropout. Confirmed pin type info is available at emit time (`FBpirTextEmitter::PinTypeToBpirType` is already used for entry signature emission). Filing as feature request (language extension + emitter) rather than pure emitter bug because the syntax doesn't exist yet.
- `#2-implement-register-type-annotations` `IN-REVIEW` developer — Added optional `%name: Type =` register-binding annotation to BPIR. Emitter (`BpirTextEmitter.cpp`) hoists `ResolvePrimaryOutputTypeAnnotation` from existing `CollectParams` filter and prepends `: <PinTypeToBpirType>` at 23 binding sites; exec/delegate/hidden pins yield no annotation. Parser (`BpirParser.cpp`) consumes optional `Colon`-prefixed typespec before `Equals` in the `PercentRef`-led path, records on new `FBpirInstruction::{DeclaredResultType, bHasDeclaredResultType}` fields, parse-and-record only (no validation in v1). Per-keyword dispatch lambdas untouched. Added Tests 1–4 (CallEmitsPinTypeAnnotation, BranchAndCastEmitTypeAnnotation, ParserAcceptsOptionalTypeAnnotation, RoundTripPreservesAnnotatedGraph). Docs updated in `bpir-language-reference.md` §2/§6 and `bpir-examples.md`.
- `#3-review-fixes` `IN-REVIEW` developer — Applied review battery findings: extracted shared `IsPrimaryOutputDataPin` predicate so `ResolvePrimaryOutputTypeAnnotation` and `CollectParams` share it (DRY); switched parser warning from `LogTemp` to `LogBpirCompiler`; trimmed WHAT-style comments in emitter helper and parser block per CLAUDE.md; switched Test 1 substring check to line-level via `ContainsLineWith`; removed/replaced tautological assertion in Test 2; added explicit `TestTrue`/`TestNotNull` on the foreach sub-test in Test 3 to fail loudly on compile failure; resolved placeholder heading in `bpir-language-reference.md`; switched `OutInst.DeclaredResultType = ParsedType` to `MoveTemp` to avoid TUniquePtr deep-copy.
- `#4-verify-fix` `DONE` tester — Verified: `blueprint.decompile` on `/App/App/LevelBlueprints/B_DronePlayerController` (one of the ticket's four cited assets). BeginPlay now emits `%n0: object<DroneGameInstance> = call GetDroneGameInstance()`, `%n1: object<GameUserSettings>`, `%n2: enum<EWindowMode>`, `%n3: bool`, `%n4: object<AppActivationSubsystem>` — exactly the types the reporter said had to be inferred. Branch/latent `%n` bindings whose primary output is exec/delegate correctly carry no annotation (e.g. `%n0 = branch(...)`, `%n1 = latent Delay(...)`), matching the spec's exec/delegate/hidden-pin exclusion.
- `#5-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 2 body citations repointed in place; every rewritten path was confirmed to exist at plugin HEAD `ef8a1f1b`. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
