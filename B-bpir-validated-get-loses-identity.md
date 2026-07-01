---
id: B-bpir-validated-get-loses-identity
title: "BPIR decompiler emits Validated Get (K2Node_VariableGet with exec) without the variable identity"
status: DONE
severity: Medium
category: bug
tags: [bpir, decompiler, variable-get]
---

# Validated Get decompile drops variable name

UE's "Validated Get" node is a `K2Node_VariableGet` that adds a `then`
exec output for the failure branch when the variable is null/invalid.
It is structurally a variable read, but the decompiler treats it like
an unknown exec node: it emits `call K2Node_VariableGet() [then -> @then]`
with no variable name, no type, and no link to the value pin.

The non-validated `K2Node_VariableGet` (pure data form) decompiles
correctly to `$VarName` in BPIR. Only the validated/exec variant
falls through to generic fallback. 23 files in the current cache
contain the broken form.

**Repro:** Open a BP that uses Validated Get on any variable (right-click
→ Convert to Validated Get) and decompile via `asset.dump` /
`blueprint.decompile`.
- Example: `Game\Audio\Blueprints\WeaponAudioMacros\bpir.txt`

**Expected:** Either an explicit `validated_get($VarName) [then -> @then]`
form, or a thin wrapper that re-uses the existing `$VarName` emit and
adds the exec edge.

**Actual:** `call K2Node_VariableGet() [then -> @then]` — variable name
gone, type gone, value pin unwired.

**Fix (proposed):** Branch in the variable-get emitter on
`bIsPureGet`/exec-pin presence; when an exec pin exists, reuse the
existing variable-name resolution but attach the exec edge.

## History
- `#1-initial-audit` `OPEN` reporter — 23 `bpir.txt` files in the cache contain `call K2Node_VariableGet() [then -> @then]`. Variable identity completely lost. Distinct from the broader emitter-coverage-gap entry because the fix is a one-line change in the existing variable-get emitter, not a new typed emitter.
- `#2-validated-get-dispatcher-case` `IN-REVIEW` developer — Extended `EmitVariableGet` in BpirTextEmitter.h/.cpp to accept a LabelMap and append exec targets. Added `case ENodeSemantics::VariableGet:` in BpirDecompiler.cpp before the generic fallback (mirrors the VariableSet pattern). Pure-VGet inlining via `ResolveInputValue` is unaffected. Round-trip caveat: validated-get authoring is not yet supported by the compiler, so recompile yields a pure get; identity is preserved in the dump (the bug). Test in TestDecompiler.cpp.
- `#3-verify-weaponaudiomacros` `DONE` tester — Live `blueprint.decompile` of `/Game/Audio/Blueprints/WeaponAudioMacros` now emits `%n0 = get Instigator [then -> @then]` for the validated get on `Pawn.Instigator`. No `call K2Node_VariableGet()` occurrences remain; variable identity preserved with exec edge attached.
