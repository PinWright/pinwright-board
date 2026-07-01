---
id: B-agir-alphaboolblend-empty-struct-not-reimportable
title: "AGIR compile-side WriteAnimNodeFieldByName never unquotes, so quoted struct fields (e.g. AlphaBoolBlend: \"()\") fail ImportText and drop on round-trip"
status: IN-REVIEW
severity: Low
category: bug
tags: [agir, animgraph, roundtrip, decompile, compile, lossy-field, alphaboolblend, unquote]
---

# AGIR `WriteAnimNodeFieldByName` forgets to unquote, dropping quoted struct fields

`anim.decompile_agir` emits the `AlphaBoolBlend` struct field as the
quoted empty-parenthesis literal `AlphaBoolBlend: "()"` (a default
`FInputAlphaBoolBlend` whose `ExportText` collapses to `()`, then wrapped
in quotes by the emitter). Feeding that **unmodified decompiler output**
back into `anim.compile_agir` does not error, but emits a non-fatal
warning and **silently drops the field**:

```
AGIR_FIELD_WRITE: ImportText failed for 'AlphaBoolBlend' on FAnimNode_ApplyAdditive
```

So `AlphaBoolBlend: "()"` is text the AGIR decompiler emits that its own
compiler cannot consume. The compile still "succeeds" (the rest of the
graph round-trips), but the field is dropped — the round-trip is **lossy**
for that property.

## Root cause (not the empty-struct symptom)

The empty-`"()"` case is just the case that *deterministically* fails; the
real defect is broader and lives on the **compile side**, not the emit side.

- **Emit:** `FIrTextUtils::FormatReflectedPropertyValue`
  (`IrCore/IrTextUtils.cpp:661-663`) exports any struct via
  `ExportTextItem_InContainer` then **unconditionally wraps it in
  `Quote()`**. So *every* struct field is emitted quoted, e.g.
  `AlphaBoolBlend: "()"` or `AlphaBoolBlend: "(BlendInTime=0.25)"`.
  (`AppendReflectedFields` already skips fields `Identical` to the CDO —
  `IrTextUtils.cpp:701-707` — but `FInputAlphaBoolBlend` carries `Transient`
  members that differ from the CDO, so it is emitted even when its editable
  members are at default, producing the `()` body.)
- **Parse:** `ParseArgList` (`AGIR/AGIRParser.cpp`) keeps the surrounding
  quotes verbatim — the value reaching the writer is the literal `"()"`
  (quote chars included).
- **Compile (the bug):** `WriteAnimNodeFieldByName`
  (`Handlers/Animation/AnimGraphConstructionUtils.cpp:400`) passed that raw
  quoted text straight to `FProperty::ImportText_Direct` with **no
  unquoting**. `FStructProperty::ImportText` expects to begin with `(` and
  rejects the leading `"`, so it returns null → `ImportText failed`, surfaced
  as the non-fatal `AGIR_FIELD_WRITE` warning (`AGIRCompiler.cpp`) and the
  field is dropped.

`WriteAnimNodeFieldByName` was the **lone outlier** among the reflective
writers: its two siblings already unquote before `ImportText` —
`WriteUObjectFieldByName` via `UnquoteIfQuoted`
(`AGIR/AGIRCompilerHelpers.cpp:47`) and `TryWriteEditorClassField` via
`UnquoteForImport` (`AGIR/AGIRCompiler_LinkedInputPose.cpp:50-51`). String /
Name / enum fields round-tripped only because their `ImportText` tolerates
surrounding quotes; structs do not — which is why this surfaced specifically
on struct fields. The narrow "don't emit empty structs" framing would only
hide the `()` case and leave every *non-empty* quoted struct field at the
same re-import risk.

## Fix

Unquote in `WriteAnimNodeFieldByName` before `ImportText`, mirroring its two
siblings (reuse `FIrTextUtils::TryUnwrapStringLiteral` /
`TryUnwrapNameToken`). A leading-`"` value unwraps (`"()"` → `()`, which a
struct imports as "leave editable members at default"); an unquoted value
passes through unchanged.

Invariant restored: **`anim.decompile_agir` output fed verbatim into
`anim.compile_agir` must round-trip every emitted field with no `ImportText
failed` warning and no silent field drop.** A round-trip regression test
covers a node carrying an `AlphaBoolBlend` (`ApplyAdditive`).

## Evidence

Surfaced during an `anim` round-trip authoring task on a real
`UAnimBlueprint` (`/Game/.../1-4_ABP_AdditiveAnimations`, via a safe
duplicate copy). The control round-trip (feeding the **unmodified**
decompiler output straight back) produced `warning only: ImportText failed
for AlphaBoolBlend` on `anim.compile_agir` — i.e. the decompiler's own
output is not losslessly re-importable. The task's friction note flagged it
verbatim: *"a minor lossy field: AlphaBoolBlend: \"()\" emitted by the
decompiler fails to re-import (ImportText warning) and drops on round-trip."*
The intended edit (an unrelated `Alpha: 0.5` on the `ApplyAdditive` node)
still landed and the rest of the graph round-tripped, so this is a
low-severity lossy-field gap, not a blocker — but it is a real hole in the
round-trip-lossless guarantee and is currently untracked.

## History
- `#1-initial-audit` `OPEN` reporter — Filed from a struggle audit of an
  `anim` AGIR round-trip task. `anim.decompile_agir` emits `AlphaBoolBlend:
  "()"`; feeding that verbatim back into `anim.compile_agir` warns `ImportText
  failed for AlphaBoolBlend` and drops the field (lossy round-trip). Same
  defect class as the `F-agir-cliff-completion` families but a still-open
  empty-struct `ExportText`/`ImportText` mismatch — distinct from the
  state-machine output-pose binding bug filed as
  `B-agir-state-machine-output-pose-unbound`. Evidence: the task's control
  round-trip on `1-4_ABP_AdditiveAnimations` (unmodified decompile fed back)
  produced the warning; the per-finding judge filed only the state-machine
  bug, leaving this lossy field untracked.
- `#2-reword-and-fix` `IN-REVIEW` developer — Rewording: the defect is real
  but the root cause is compile-side, not an emit-side empty-struct symptom.
  `WriteAnimNodeFieldByName` (`AnimGraphConstructionUtils.cpp:400`) was the
  lone reflective writer that fed the raw quoted value to
  `ImportText_Direct` without unquoting; its siblings
  `WriteUObjectFieldByName` and `TryWriteEditorClassField` already unquote.
  The AGIR emitter quotes *every* struct export (`IrTextUtils.cpp:663`), so a
  default `FInputAlphaBoolBlend` emits as `"()"` and any non-empty struct
  field emits quoted too — all of which `FStructProperty::ImportText` rejects
  for the leading `"`. Fix: added a local `UnquoteForReflectiveImport`
  (`TryUnwrapStringLiteral` / `TryUnwrapNameToken`, matching the siblings) and
  applied it in `WriteAnimNodeFieldByName` before `ImportText_Direct`; added
  the `IrCore/IrTextUtils.h` include. Regression test
  `Tests/Assets/TestAGIRAnimNodeStructField.cpp`
  (`EditorAutomationRpcGateway.AGIR.AnimNodeField.QuotedStructReimports`)
  builds a transient `UAnimGraphNode_ApplyAdditive` and asserts the quoted
  `"()"` literal and a quoted non-default struct body both re-import without
  error (and that BlendInTime/BlendOutTime land) — reverting the unquote fails
  the first assertion. Files: `Private/Handlers/Animation/AnimGraphConstructionUtils.cpp`,
  `Private/Tests/Assets/TestAGIRAnimNodeStructField.cpp`.
