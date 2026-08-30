---
id: B-metasound-variable-int-type-rejected
title: "add_metasound_variable rejects the documented variableType 'Int'; only the undocumented 'Int32' works"
status: IN-REVIEW
severity: Medium
category: bug
tags: [metasound, audio, authoring, add_metasound_variable, variableType, registry, docs-contract]
---

# `add_metasound_variable` rejects its own documented `variableType: "Int"`

`audio.authoring.add_metasound_variable`'s wiki page documents the
`variableType` parameter verbatim as **"Type name (Float, Int, Bool, String)"**.
Three of those four advertised type names work; the fourth, `Int`, is rejected
with a generic `[VARIABLE_FAILED]`. The only spelling that actually creates an
integer graph variable is `Int32`, which appears in **no** documentation. A
caller who copies the documented `Int` form gets a dead-end error with no hint
that `Int32` is the accepted key — a broken documented contract.

## What's wrong

The handler passes the **raw `variableType` string** straight through to the
MetaSound Frontend builder:

`MetaSoundVariableHandler.cpp:154-155`
```cpp
const FMetasoundFrontendVariable* Variable = Builder.AddGraphVariable(
    FName(*VariableName), FName(*VariableType), &Literal);
```

`FMetaSoundFrontendDocumentBuilder::AddGraphVariable` resolves the second
argument against the MetaSound data-type registry, whose key for a 32-bit
integer is `Int32`, not `Int`. So `FName("Int")` finds no registered data type,
`AddGraphVariable` returns null, and the handler emits the generic
`[VARIABLE_FAILED]` (line 174-175).

The bug is masked from the handler's own validation. The plugin's
`MakeDefaultLiteralForMetaSoundType` helper (`MetaSoundLiteralFromTypeName.cpp:14`)
explicitly accepts `Int` as an alias for `Int32` — but that helper only runs as a
fallback in `BuildVariableLiteral` when **no** explicit `*Value` param is given.
When the caller passes `intValue` (the normal case — the wiki documents
`intValue` for the initial value), `BuildVariableLiteral` returns early with a
valid int literal and never validates the type name. So the handler's own
`INVALID_TYPE` guard (line 138) never fires for `Int`, the literal is built fine,
and the only thing that fails is the registry lookup inside the builder — with no
alias mapping. The literal helper "knows" `Int == Int32` but the builder call
does not.

## What it should do

Map the documented convenience type name `Int` to the registry's canonical
data-type key `Int32` before calling `AddGraphVariable`, reusing the same alias
set `MakeDefaultLiteralForMetaSoundType` already encodes. Of the four documented
types, only `Int` is a genuine mismatch: the engine registers `float`/`bool`/
`FString` as `"Float"`/`"Bool"`/`"String"` (verbatim matches to the documented
names — `MetasoundPrimitives.cpp:114,116,117`) but registers `int32` as `"Int32"`
(`MetasoundPrimitives.cpp:115`), so `Float`/`Bool`/`String` already round-trip and
need no normalization. Equivalently, the documented type list could be corrected
to the registry spelling — but mapping is preferable since `Int` is the natural,
documented form and the helper already treats it as canonical. Either way the
wiki's advertised `(Float, Int, Bool, String)` must round-trip.

**Fix:** Add a `CanonicalizeMetaSoundTypeName` alias map (next to
`MakeDefaultLiteralForMetaSoundType`, sharing its alias set) that rewrites `Int`
-> `Int32` (and accepts the other documented names as-is), and call it on
`VariableType` before `Builder.AddGraphVariable` in `MetaSoundVariableHandler.cpp`.
Scope the change to `Int` only — do NOT rewrite `Bool` to `Boolean`, which would
break the already-working `Bool` path since the registry key is `"Bool"`.

## Verbatim repro (replay-confirmed live)

1. `audio.authoring.create_metasound`
   `{name:"MS_OracleReplay", path:"/Game/Audio/MetaSounds"}` -> ok.
2. `audio.authoring.add_metasound_variable`
   `{assetPath:"/Game/Audio/MetaSounds/MS_OracleReplay", variableName:"WindLayerCount", variableType:"Int", intValue:3}`
   -> **`[VARIABLE_FAILED] Failed to add variable 'WindLayerCount' of type 'Int'`**
   (`Int` is the wiki's own documented type name).
3. Same call with `variableType:"Int32"`
   -> succeeds: `{"message":"Variable 'WindLayerCount32' added to MetaSound", "variableType":"Int32", ...}`.
4. Control — `variableType:"Float"` (also documented) succeeds:
   `{"message":"Variable 'WindGain' added to MetaSound", "variableType":"Float", ...}`.

Wiki page (`Saved/.../wiki/audio.authoring.add_metasound_variable.md`) line for
the param verbatim: `variableType (string, optional): Type name (Float, Int, Bool, String) Default: Float.`

## Impact

Medium. A caller can recover by guessing `Int32`, but the documented form fails
with a generic error that names no accepted alternative, so an integer graph
variable is unreachable by following the docs. Among the four documented types,
`Int` is the only registry mismatch (`Float`/`Bool`/`String` match their registry
keys verbatim, so they already work). The existing regression test
`TestMetaSoundVariables.cpp` only exercises `Float`, so the `Int`/`Int32` path is
untested.

## History
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed live against `mcp__editor-automation__call`: `add_metasound_variable {variableType:"Int", intValue:3}` returns `[VARIABLE_FAILED] Failed to add variable '...' of type 'Int'`; the same call with `variableType:"Int32"` succeeds; `Float` succeeds. Root cause grounded in source: handler passes the raw `variableType` string to `Builder.AddGraphVariable` (`MetaSoundVariableHandler.cpp:154-155`), whose registry keys integers as `Int32`; the `Int` alias is honored only by `MakeDefaultLiteralForMetaSoundType` (`MetaSoundLiteralFromTypeName.cpp:14`), which is bypassed when an explicit `intValue` is supplied, so no alias mapping reaches the builder call. Wiki documents `variableType` as `(Float, Int, Bool, String)` — `Int` is a valid documented input that is wrongly rejected. Deduped against `F-metasound-no-variables-or-validate` (DONE — introduced the handler, not this contract bug), `B-add-metasound-node-rejects-registry-classnames` / `E-metasound-node-add-docs-misleading` (different RPC `add_metasound_node`, node class names not variable data-type names); genuinely new.
- `#2-reword` `IN-REVIEW` developer — Reworded: dropped the body/Impact `Bool` -> `Boolean` normalization claim, which is false for this engine. Verified against `C:\UE_5.7\...\MetasoundPrimitives.cpp:114-117`: the registry keys `bool`/`float`/`FString` as `"Bool"`/`"Float"`/`"String"` (verbatim matches to the documented names) and only `int32` as `"Int32"`. So among the four documented types (Float, Int, Bool, String) only `Int` is a genuine mismatch; rewriting `Bool` to `Boolean` would have broken the already-working `Bool` path. Scoped the **Fix:** to `Int` -> `Int32` only.
- `#3-fix` `IN-REVIEW` developer — Added `CanonicalizeMetaSoundTypeName(TypeName)` to the shared alias module (`MetaSoundLiteralFromTypeName.h/.cpp`): rewrites `Int` -> `Int32` (case-insensitive), passes everything else through unchanged so the builder still emits its own "unregistered DataType" diagnostic for genuinely bad input. Called it on `VariableType` before `Builder.AddGraphVariable` in `MetaSoundVariableHandler.cpp` (was passing the raw `FName(*VariableType)`). Files: `Source/EditorAutomationRpcGateway/Private/Handlers/Audio/MetaSound/MetaSoundLiteralFromTypeName.h`, `.../MetaSoundLiteralFromTypeName.cpp`, `.../Handlers/Audio/MetaSound/MetaSoundVariableHandler.cpp`. Regression test: extended `Tests/Assets/TestMetaSoundVariables.cpp` with `FAddMetaSoundVariableIntAliasTest` — asserts the production canonicalizer maps `Int`/`int` -> `Int32` and leaves `Int32`/`Float`/`Bool`/`String` unchanged, then (UE 5.6+) proves raw `FName("Int")` is rejected by `Builder.AddGraphVariable` (returns null) while the canonicalized name adds the variable. Reverting the canonicalizer fails the "Canonicalized 'Int' adds an integer graph variable" assertion. Not yet compiled/tested (later phase).
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 1 citation sits in history rows and is left verbatim per the append-only rule, mapping by the same rule; the mapped path was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
