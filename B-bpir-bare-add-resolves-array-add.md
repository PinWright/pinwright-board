---
id: B-bpir-bare-add-resolves-array-add
title: "BPIR bare `Add(A:,B:)` shadow-resolves to array Add; error misdirects instead of naming Add_IntInt"
status: OPEN
severity: Low
category: bug
tags: [bpir, function-resolution, displayname, diagnostics]
---

# BPIR bare `Add(A:, B:)` shadow-resolves to array Add; error misdirects

In BPIR, `call Add(A: $intVar, B: $intVar)` resolves to the **array** `Add`
node (`UKismetArrayLibrary::Array_Add`, pins `self`, `TargetArray`, `NewItem`)
rather than the arithmetic `Add_IntInt`. The resolver is name-only:
`FCodeFunctionResolver::ResolveFunction`
(`Source/PinWright/Private/Compiler/CodeFunctionResolver.cpp:78`) runs four
passes and never inspects argument pin types; its DisplayName/ScriptName meta
pass (lines 146-183) matches `Array_Add`, which carries
`meta=(DisplayName = "Add")` (`KismetArrayLibrary.h:36`). `UKismetMathLibrary`
has **no** function whose FName or DisplayName is bare `Add` — the arithmetic
overloads are `Add_IntInt`/`Add_Int64Int64`/`Add_DoubleDouble` with
`DisplayName = "int + int"` / `"float + float"` and `Keywords = "+ add plus"`
(`KismetMathLibrary.h:321,424,547`) — so the array library wins regardless of
lookup order. Because `Array_Add` lacks the caller's `A`/`B` pins, the call
fails to bind and the error (`BuildMissingPinHint`, `BpirCompiler.cpp:6767`)
previously misdirected the author to **rename the pin**
(`Could not find target pin 'A' on node 'Add'. Did you mean 'TargetArray'?`) —
a dead end (renaming `A`→`TargetArray` then fails on `B` and is semantically
wrong). The real problem is the **function name**.

**The mis-binding itself is arguably-correct Blueprint behavior and is NOT the
defect to fix here.** In the Blueprint editor the visible node named "Add" *is*
the array-append node; the arithmetic node's visible name is "int + int" /
CompactNodeTitle "+". The only thing linking the bare token `Add` to
`Add_IntInt` is the `Keywords` meta ("add"), which the resolver deliberately
does not read — and if it did, `add` would match every `Add_*` overload at once
(`Add_IntInt`, `Add_Int64Int64`, `Add_DoubleDouble`, `Add_VectorVector`, ...),
which cannot be disambiguated without a full arg-type overload-resolution engine
threaded through the resolver core (`ResolveFunction` /
`ResolveFunctionAcrossLibraries`, hit on **every** BPIR call resolution). That
is disproportionate for a soft issue with a one-token workaround. So the actual,
scoped defect is the **misdirecting diagnostic**.

**Workaround:** spell it `Add_IntInt` (or `Add_DoubleDouble` for floats) — it
resolves via exact FName (pass 1); the qualified `UKismetMathLibrary::Add_IntInt`
form also works.

**Fix (diagnostic only — do NOT add arg-type overload resolution to the resolver
core):** when a bare name fuzzy-resolves (space-strip / DisplayName-meta pass) to
a `UK2Node_CallFunction` whose bound FName differs from the requested name and
which lacks the caller's named arg pins, the missing-pin error must (a) suppress
the misdirecting per-pin `Did you mean '<pin>'?` guess, and (b) name the
display-name shadow and surface the exact cross-library operator overloads of the
requested name (`<Name>_<Types>`, e.g. `UKismetMathLibrary::Add_IntInt`,
`Add_DoubleDouble`) plus the qualified `Class::Method` escape hatch. This leaves
resolution behavior unchanged (bare `Add` still binds to `Array_Add` and still
fails, as it arguably should) — it only replaces the dead-end pin hint with an
actionable function-name hint.

## History
- `#1-initial-repro` `OPEN` reporter — `blueprint.compile_bpir` with `%sum = call Add(A: $Score, B: $Points)` on `/Game/_FabGallery/BP_ScoreBus` → `Line 2: Could not find target pin 'A' on node 'Add'. Did you mean 'TargetArray'? Available pins: self, TargetArray, NewItem`. Replacing `Add` with `Add_IntInt` compiled (5 nodes). Verified in source: `ResolveFunction` (`CodeFunctionResolver.cpp:78-186`) matches by name across 4 passes and takes no arg pin types; pass 4 (DisplayName/ScriptName meta, lines 146-183) matches `Array_Add`'s `meta=(DisplayName="Add")` (`KismetArrayLibrary.h:36`); orchestration `BpirCompiler.cpp:5663-5705`. Not a dup: `B-bpir-resolve-ufunction-by-displayname` (DONE) added the meta pass that causes this; `B-bpir-target-shadowed-by-self-class` (DONE) is explicit-`Target:` cascade ordering; `E-create-node-operator-symbol-discovery` (IN-REVIEW) is a create_node docs cheat-sheet.
- `#2-reword-narrow-to-diagnostic` `IN-REVIEW` developer — Reworded: the headline "thread arg types through the resolver core + prefer param-matching candidate" fix is over-scoped (bare `Add` matches `Add_IntInt` only via `Keywords`, which the resolver doesn't read; reading it makes `add` ambiguous across every `Add_*` overload → forces a full arg-type overload engine on every BPIR call — disproportionate for a soft issue with a one-token workaround, and the current mis-binding is defensible per Blueprint's own node naming). Severity Medium→Low (diagnostic-only; no wrong compiled output; documented one-token workaround). Narrowed to the ticket's own "at minimum" clause and implemented it. Changes: (1) new `FCodeFunctionResolver::FindOverloadSiblings(BareName)` (`CodeFunctionResolver.h/.cpp`) — read-only, returns cross-library `Class::<Name>_<Types>` overloads (e.g. `UKismetMathLibrary::Add_IntInt`); (2) `BuildMissingPinHint` (`BpirCompiler.cpp`) takes a function-shadow hint and, when present, suppresses the misdirecting `Did you mean '<pin>'?` and appends the shadow+overloads note; (3) `WireDataPins` (`BpirCompiler.cpp:6767`) detects the shadow via `UK2Node_CallFunction::GetTargetFunction()` name mismatch and builds that hint. Resolution behavior unchanged (only the error path). Regression test `PinWright.bpir.compiler.errors.BareAddShadowSuggestsOverload` (`Tests/Bpir/TestCompilerErrors.cpp`) compiles `call Add(A:3, B:4)` on an in-code transient BP and asserts the error surfaces `Array_Add` + `Add_IntInt` and no longer says `Did you mean 'TargetArray'`.
- `#3-attempt-failed` `OPEN` developer — Auto-fix reached tests:CLEAN on the correct host, but the commit ran against the wrong (sibling) host tree — clean there — so it returned committed:false and the verified diff never reached origin (root cause: the fix workflow hardcoded fuzz1 projectPath default; now removed + published-gating added). The diff was backed up and reset. Returned to OPEN for a clean retry.
