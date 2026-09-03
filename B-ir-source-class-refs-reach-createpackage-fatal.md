---
id: B-ir-source-class-refs-reach-createpackage-fatal
title: "Class refs inside AGIR/CRIR/BPIR source text reach `CreatePackage`'s Fatal — the dispatch gate can never see them"
status: IN-REVIEW
severity: High
category: bug
tags: [agir, crir, bpir, compiler, createpackage, editor-crash, path-safety, tokenizer]
blockedBy: [B-createpackage-unvalidated-paths-plugin-wide]
---

# IR source text carries class references past every boundary guard

`B-createpackage-unvalidated-paths-plugin-wide` closes the `CreatePackage` Fatal
by typing path-shaped params `path` / `classref` and refusing `//` at the
dispatch boundary. The IR compilers defeat that by construction: the class
reference is not a parameter, it is **a substring of the `text` parameter**, and
`text` is IR source that must stay typed `string`. The boundary can never see it.

`CreatePackage` logs at **Fatal** on a package name containing `//`
(`UObjectGlobals.cpp:1094-1096`) — not compiled out in any configuration; it ends
the process and every unsaved package in it. Door:
`StaticLoadObjectInternal` → `ResolveName2(..., Create=true)` (`:1427`) →
`CreatePackage` (`:1310`).

## The tokenizers do not filter `/`

Verified by reading each, rather than assumed:

- The IR comment character is **`#`, not `//`**. `FIrTextUtils::StripTrailingComment`
  (`IrCore/IrTextUtils.cpp:154-178`), `FIrTokenizer` (`IrCore/IrTokenizer.cpp:145`)
  and BPIR's own copy (`Compiler/BpirParser.cpp:92-115`) all cut only at an
  unquoted `#`. `/` is special nowhere in the lexer.
- `Unquote` (`AGIR/AGIRCompiler_BlendSpace.cpp:42-49`) strips one surrounding pair
  of `"` and returns the rest **verbatim**. It filters nothing.
- AGIR arg values are pure substring reads (`AGIRParser.cpp:534`, `:594`, `:637`)
  with no character-class check.

So `//` survives the tokenizer intact and arrives at the load.

## Repro shape

```
blend_space `BS` class=/Game//X.X_C {
```

reaches `LoadObject<UClass>` at `AGIRCompiler_BlendSpace.cpp:283` with `//`
intact → `ResolveName2(Create=true)` → `CreatePackage` → **Fatal**.

Quoting is optional. `class=`, `asset=` and `control_enum=` need no quoting at
all; the `call` / `implements` opcodes need the backtick form, because
`DecodeDelimitedToken` accepts every character while bare tokens are held to
`[A-Za-z_][A-Za-z0-9_]*`.

## Sites — seven raw loads

These are raw `LoadObject`, **not** `ResolveUClass`, so the shared-resolver
guards added by the parent wave do not cover them:

| File | Line |
|---|---|
| `AGIRCompiler.cpp` | `:1040`, `:1615` |
| `AGIRCompiler_BlendSpace.cpp` | `:124`, `:283`, `:360` |
| `CRIRCompiler.cpp` | `:1059` |
| `CodeNodeEmitter.cpp` | `:1502` (via `BpirCompiler.cpp:5741`) |

`CodeNodeEmitter.cpp:1502` is the one BPIR site not routing through a guarded
resolver, against 25 that do — so BPIR is nearly clean and this is the outlier.

## Why the parent wave's mechanism does not apply

The parent ticket's whole argument is that guarding N call sites is wrong because
it does not prevent the N+1th, and that declaring the input as a path and
validating once at the boundary is the maintainable answer. Neither half is
available here:

- The boundary sees `text`, an opaque IR document. Typing it `path` is nonsense
  and typing it `classref` would refuse every valid program.
- The class ref is produced by the parser, mid-compile, from caller text.

The validation therefore has to live where the compiler resolves a reference —
which is the one place that knows a token is a class ref. The candidate fix is a
single guarded resolve helper inside the IR compilers, used by all seven sites,
refusing `//` with the compiler's normal diagnostic machinery (line/column) rather
than a Fatal. That keeps it to one chokepoint per compiler rather than seven
scattered guards.

## Related, same wave, different cause

`data_table.set_row_struct` and `data_table.create` resolve `structPath` through
`ResolveRowStruct` (`DataTableAuthoringHandler.cpp:56-59`), which calls
`LoadObject<UScriptStruct>` directly rather than the guarded
`ResolveUScriptStruct`. That one **is** closed by the parent wave, because
`structPath` is a real top-level param and was retyped `path` — but the resolver
asymmetry is worth fixing so the defence is not single-layered.

## History

- `#1-filed-from-plugin-wide-wave` **OPEN** (Reporter) — Found during the
  `B-createpackage-unvalidated-paths-plugin-wide` implementation wave by the
  agent retyping the blueprint/IR cluster, which read the tokenizers to check
  whether `//` survives them. It does. Filed separately because the parent
  wave's dispatch-boundary mechanism structurally cannot reach it. Blocked on the
  parent landing, to avoid conflicting edits in the same compilers.
- `#2-guarded-at-the-point-of-use` **IN-REVIEW** (Developer) — Verified TRUE from source;
  all seven listed sites were still raw, plus five the ticket did not list (MGIR, including two
  where the existing `IsValidLongPackageName` check ran AFTER the load it was meant to protect).
  Fixed with the ticket's own recommendation, but the guarded resolver is SHARED with
  `B-nested-path-values-reach-createpackage-fatal` rather than per-compiler: both tickets are one
  defect seen from two input shapes. All nine IR trees now hold at zero unguarded loads under a
  source-scan ratchet. Not compiled or run by the implementing agent; a separate compile pass
  follows.

## Fix

Confirmed TRUE by reading source on 2026-09-03; all seven sites were still raw `LoadObject` on
parser output. Fixed with the ticket's own recommendation - one guarded resolve helper - but the
helper is SHARED with the nested-value ticket rather than being per-compiler, because both tickets
are the same defect seen from two input shapes and a guard keyed on where the load HAPPENS is blind
to input shape by construction.

New shared guard: `Source/PinWright/Private/Utils/GuardedLoad.h` -
`PinWrightGuardedLoad::LoadObjectChecked<T>(Path, OutRefusal?, LoadFlags?)`. Every converted site
appends the refusal to the diagnostic it already emits, so the compiler's line/column is preserved
rather than replaced.

Sites converted (14, up from the ticket's 7):
- `AGIR/AGIRCompiler.cpp` - `call` node class, `implements` interface class, and (defence in depth)
  the `Options.Context` anim-blueprint target.
- `AGIR/AGIRCompiler_BlendSpace.cpp` - `sample_graph` child class, `class=`, `asset=`.
- `CRIR/CRIRCompiler.cpp` - `control_enum` (which now also reports the refusal as a
  `CRIR_CONTROL_BAD_SUBBLOCK_KEY` warning instead of dropping it), and `Options.TargetAssetPath`.
- `Compiler/CodeNodeEmitter.cpp:1502` - the one BPIR site not routing through a guarded resolver.
- **`MGIR/MGIRCompiler.cpp` and `MGIR/MGIRExpressionEmitter.cpp` (5 sites), NOT in the ticket's
  table.** Same class: `Spec.FunctionPath` / `LayerFunctionPaths` / `BlendFunctionPaths` are parser
  output, and `GetOrCreateMaterial` / `GetOrCreateMaterialFunction` load on the block name BEFORE
  their own `IsValidLongPackageName` check - so the validation that would have refused `//` ran
  after the load that died on it.

`CodeNodeEmitter.cpp:234` is deliberately NOT converted: its argument is a `TEXT(...)` literal
(`/Engine/EditorBlueprintResources/StandardMacros`), which no caller text reaches.

The related `DataTableAuthoringHandler::ResolveRowStruct` asymmetry noted in this ticket is
untouched - `structPath` is a top-level param already typed `path`, so it is a resolver-consistency
cleanup, not a reachable defect. Left for a separate ticket rather than widened into this one.

Docs: `Docs/rpc-design.md` section 24; `Handlers/ParamTypeCheck.h` and `Utils/PathUtils.h` header
contracts updated to name the second layer.

### Verification

No editor run is needed for the code review; the tests are automation tests.

1. Run `PinWright.infra.contract.LoadGuard.IrCompilersLoadThroughTheGuard`
   (`Private/Tests/Infra/TestIrAndNestedLoadGuard.cpp`). It holds ALL nine IR trees - AGIR, BTIR,
   CRIR, Compiler, IrCore, MGIR, MSIR, NIR, SCIR - at **zero** unguarded loads, with a literal
   argument exempt (decided by reading the arguments, not by naming a site). BTIR/MSIR/NIR/SCIR were
   already clean and are enrolled so a future IR compiler inherits the rule by location.
2. Run `PinWright.Core.Path.GuardedLoad.*` (`Private/Tests/Core/TestGuardedLoadPathSafety.cpp`) for
   the guard's own contract, including that `class=/Game//X.X_C` - the repro shape in this ticket -
   is refused and that the six legal reference shapes still resolve.
3. Sanity: `grep -rn 'LoadObject<\|StaticLoadObject\|LoadClass<' Source/PinWright/Private/{AGIR,BTIR,CRIR,MGIR,MSIR,NIR,SCIR,Compiler,IrCore}`
   should return only the `StandardMacros` literal and one prose mention in a comment.
4. Do NOT drive an IR compile with `//` on a build without the fix: `CreatePackage`'s Fatal ends the
   suite host rather than reporting a red.

Not committed (board and plugin repos both left dirty, per the task).
