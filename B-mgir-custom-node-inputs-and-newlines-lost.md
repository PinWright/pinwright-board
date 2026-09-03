---
id: B-mgir-custom-node-inputs-and-newlines-lost
title: "material.compile_mgir silently breaks every Custom HLSL node it writes: named inputs are never created (so the generated shader has no `Input` parameter) and `\n` escapes in a string literal lose their backslash — the material reports success, saves, and renders nothing"
status: IN-REVIEW
severity: High
category: bug
tags: [material, compile_mgir, decompile_mgir, custom-hlsl, round-trip, silent-false-success, ui-material, shader-compile]
encounters: 1
lastSeen: 2026-09-03T02:00:00Z
---

# `material.compile_mgir` cannot write a working Custom HLSL node

## Symptom

Two independent defects in the same code path. Together they mean **every**
`MaterialExpressionCustom` authored through `compile_mgir` produces a material that
reports `blocksCompiled: 1`, writes a `.uasset`, and then fails shader compilation and
draws nothing. Nothing in the `compile_mgir` response says so.

### 1. Named Custom inputs are never created

`material.decompile_mgir` emits Custom inputs as call arguments plus an `Inputs` array:

```
%n2CB = call `/Script/Engine.MaterialExpressionCustom`(
    UV: %n04C, I: %n6DC, Code: "...",
    Inputs: [(InputName="UV",Input=(Expression="...")),(InputName="I",Input=(...))],
    OutputType: "CMOT_Float1")
```

Feeding that straight back to `compile_mgir` is rejected:

```
[MGIR_INPUT_NOT_FOUND] Input 'UV' was not found. Available inputs: Input. Did you mean 'Input'?
```

So a material containing a Custom node with any input not named `Input` **cannot survive
its own decompile**. That is the same class of round-trip bug the wiki records as fixed
for clear-coat and customized-UV root pins (`material.mgir.md` line 138).

Falling back to the one name the compiler admits — `Input` — is accepted, saves, and is
still broken: the connection is made but the `Inputs` array entry is not populated, so the
translator emits a Custom function with **no parameter**, and the shader fails:

```
/Engine/Generated/Material.ush:3746:12: error: use of undeclared identifier 'Input'
float2 d = Input.xy - 0.5; float r = length(d) * 2.0; ...
           ^
```

### 2. `\n` in an MGIR string literal loses its backslash

`Code: "float2 p = ...;\nfloat4 B[27] = ...;"` reaches the shader as

```
float2 p = float2(Input.x * 2.92857, Input.ynfloat4 B[27] = {float4(...
```

— the `\` is dropped and the `n` is kept, collapsing the whole program onto one line and
splicing the last token of each line into the first token of the next. The decompiler
emits `\n` (see the block quoted above), so this is the second half of the same
round-trip failure: decompile writes an escape the compiler cannot read.

The one-line collapse also silently destroys any preprocessor directive. A Custom node
whose code begins `#define SDB(cx,cy,hx,hy) {...}` fails with

```
/Engine/Generated/Material.ush(3745): Expected a parameter name to follow # character
in definition of macro 'SDB'.
```

which reads as "your macro is malformed" when the macro was fine and the newlines were
removed underneath it.

## Repro

```
call("material.compile_mgir", {mode:"Append", save:true, text:
  "entry material `/Game/T/M_Probe.M_Probe` {\n"
  "    property MaterialDomain: MD_UI\n"
  "    property BlendMode: BLEND_Translucent\n"
  "    %uv = call `/Script/Engine.MaterialExpressionTextureCoordinate`() @(-900, 0)\n"
  "    %c = call `/Script/Engine.MaterialExpressionCustom`(Input: %uv,\n"
  "         Code: \"float2 d = Input.xy - 0.5;\nreturn length(d);\",\n"
  "         OutputType: \"CMOT_Float1\") @(-500, 0)\n"
  "    output Opacity: %c\n"
  "}"})
-> {"blocksCompiled":1, "expressionsCreated":3, "assetPaths":[...]}     # reports success

call("material.authoring.compile_material", {materialPath:"/Game/T/M_Probe"})
-> compileSucceeded: false
   "use of undeclared identifier 'Input'"
   "float2 d = Input.xy - 0.5;nreturn length(d);"                        # note the bare `n`
```

Swapping `Input:` for the decompiler's own `UV:` fails earlier, at validation:
`[MGIR_INPUT_NOT_FOUND] Input 'UV' was not found. Available inputs: Input.`

## Why this one costs real time

`compile_mgir` is the documented, preferred way to author a material graph, and it is the
only one that round-trips. Every field it returns says the write worked. The failure is
invisible until either the material is put on screen — a PIE slot under a shared world
lock, on this project — or `material.authoring.compile_material` is called by hand, and
nothing in `compile_mgir`'s response or in `material.compile_mgir.md` suggests that second
step exists. Two UI materials (a radar backing disc and a traced weapon silhouette) were
authored, saved, verified present on disk, captured in PIE and reviewed before the
missing pixels were traced back to here; one of them had been visually "confirmed" from a
capture in which the shape actually on screen belonged to a different material underneath.

## Workaround

Author Custom nodes with `material.authoring.add_custom_expression`, which takes an
explicit `inputs: ["UV"]` array and populates it correctly, then wire with
`material.authoring.connect_nodes`. Keep the HLSL on **one line** with spaces instead of
newlines, so no `\n` escape and no preprocessor directive is needed. Both materials
compile clean this way (`compileSucceeded: true`, zero errors).

## Suggested fix

1. `compile_mgir` should create the `Inputs` array entry for every named Custom input it
   is asked to connect, and accept the `Inputs: [...]` property the decompiler emits.
2. Decode standard escapes (`\n`, `\t`, `\"`, `\`) in MGIR string literals, since the
   decompiler already emits them.
3. Round-trip test: `decompile_mgir` -> `compile_mgir` -> `material.authoring.compile_material`
   on a material with a multi-input, multi-line Custom node must end at
   `compileSucceeded: true`.
4. Consider having `compile_mgir` report the shader compile result, or at minimum have
   `material.compile_mgir.md` say that `blocksCompiled` is not evidence the material
   compiles.

## Related

- `E-material-verbs-have-no-shader-compile-signal` — `compile_material` is the only verb
  that surfaces shader errors, and nothing points authors at it.

## Fix

Both halves confirmed against source before any edit.

**(a) Custom pins were never created.** `UMaterialExpressionCustom`'s pins live in a runtime
`TArray<FCustomInput>` whose class default is ONE entry with an empty name
(`MaterialExpressions.cpp:12747-12748`), and `Compile` skips every unnamed input
(`:12769-12772`) — so wiring the derived pin name `Input` populated `Inputs[0].Input` on a pin
the translator ignores, and the generated function had no parameters. Nothing in MGIR ever
wrote `InputName`. The call's named pin arguments are now the declaration: each one creates an
input with that name, in document order, before the wire pass runs. Routed through a new
class-dispatched seam rather than an inline `if (Custom)` branch:

- `Source/PinWright/Private/MGIR/MGIRDynamicInputs.h` / `.cpp` (new) —
  `GetInputArrayPropertyName(UClass*)` (the one place that knows which classes carry dynamic
  pins, and which reflected property backs them) and `ApplyDeclaredInputNames`, which resets the
  array and refuses an empty or duplicated name. `SetMaterialAttributes` deliberately stays on
  `FMGIRMaterialAttributeUtils`: its pin names must resolve to attribute GUIDs, so its
  declaration is the explicit `Attributes: [...]` list, not the pin args. Header documents how a
  future free-form-named class plugs in.
- `MGIRCompiler.cpp` `EmitInstruction` (Call case) — collects the declared names, applies them
  after `EmitExpression`, and refuses a restated `Inputs: [...]` argument with
  `MGIR_INVALID_INPUT_DECLARATION` naming the correct form (per the ephemeral-IR invariant in
  `Docs/ir-authoring.md`, no dual-accept window: re-decompile pinned text).
- `MGIRDecompiler.cpp` `AppendExpressionProperties` — suppresses the same reflected array, since
  `AppendConnectedInputs` already emits every name and connection. That array was the
  unreadable half of the round trip.

**(b) `\n` lost its backslash.** The decompiler quotes with `FIrTextUtils::EscapeString`
(`\\ \" \n \r \t \uNNNN`); the compiler's local `TrimQuotes` reversed `\"` and `\\` only, so the
rest survived as literal backslash pairs. Fixed at the literal layer, for every literal:
`MGIRHelpers::EncodeStringLiteral` / `DecodeStringLiteral` are now the single encoder/decoder
pair (delegating to the shared `FIrTextUtils` implementation of the documented IR escape set),
`TrimQuotes` is gone, and the decompiler's `Quote` goes through the same header so the two sit
side by side.

Also bumped the `mgir.txt` asset-dump aspect version 3 → 4 (`AssetDumpCache.cpp`): a material
holding a Custom node now dumps different bytes. **The committed `asset-dumps/` mirror in the
host project is stale for such materials until re-dumped.**

Files changed: `Source/PinWright/Private/MGIR/MGIRDynamicInputs.{h,cpp}` (new),
`MGIR/MGIRCompiler.cpp`, `MGIR/MGIRDecompiler.cpp`, `MGIR/MGIRHelpers.h`,
`Handlers/Asset/AssetDumpCache.cpp`, `Docs/wiki-src/material.mgir.md` (new "String literals and
escapes" + "Custom HLSL nodes" sections), tests in
`Source/PinWright/Private/Tests/Material/TestMGIRCustomExpression.cpp` (new).

**Reviewer verification.** Not compiled and not run here (separate compile pass). Then:

1. `PinWright.material.mgir.Custom.NamedInputsAndMultilineCodeCompile` — compiles a document
   with two named inputs and a multi-line program; asserts `Inputs` = `[UV, Scale]` in order,
   both wired, and `Code` holding real newlines/tab/quote/backslash.
2. `PinWright.material.mgir.Custom.DecompileRoundTripsInputsAndCode` — decompile emits `UV: %…`
   / `Scale: %…` and no `Inputs:`, recompiles, and the Custom call line decompiles identically
   (handles normalized).
3. `PinWright.material.mgir.StringLiteral.EscapeSetDecodes` — the escape set on a plain `Desc`,
   proving the fix is not Custom-specific, plus encoder/decoder symmetry.
4. Live: re-run the ticket's repro through `material.compile_mgir` with `UV:` (the decompiler's
   own name) and a `\n`-carrying program, then `material.authoring.compile_material` — expect
   `compileSucceeded: true`. That last step is the one nothing in this change can fake; the
   `compile_mgir` response still does not carry a shader verdict
   (`E-material-verbs-have-no-shader-compile-signal` owns that).

## History
- `#1-fix-implemented` `IN-REVIEW` developer — Both defects reproduced from source (not run
  live). Custom pins: added the `MGIRDynamicInputs` seam and made the call's named pin arguments
  the input declaration, applied before wiring; decompile stops restating the array and a
  restated `Inputs: [...]` is refused with `MGIR_INVALID_INPUT_DECLARATION`. Escapes: replaced
  the compiler's two-escape `TrimQuotes` with `MGIRHelpers::DecodeStringLiteral`, the exact
  inverse of the decompiler's encoder, applied to every MGIR string literal. Bumped the
  `mgir.txt` dump aspect to 4. Added three round-trip tests. Did not compile or run tests
  (later phase). No shader-compile signal added — that is
  `E-material-verbs-have-no-shader-compile-signal`.
