---
id: B-bpir-spurious-unknown-node-warning-lines
title: "Stale 'Unknown node type (generic fallback): K2Node_*' warnings still emit even after the generic decompile path produces a correct call line"
status: DONE
severity: High
category: bug
tags: [bpir, decompiler, warning, dead-code]
---

# Spurious "Unknown node type" warnings persist after generic-fallback fix

After `B-bpir-decompiler-emitter-coverage-gap` and
`F-bpir-add-generic-node-statement-form` shipped, the decompiler now
correctly emits a `call K2Node_<Type>(args)[exec_targets]` line for
unknown K2Nodes. **However, the old warning-only stub still fires in
parallel**, contaminating every affected `bpir.txt` with lines like:

```
Unknown node type (generic fallback): K2Node_SpawnActorFromClass
Unknown node type (generic fallback): K2Node_CreateWidget
```

**Counts in fresh 2026-05-04 dump:** 399 warning lines across 138
files. Top contributors:

| K2Node | Warnings | Call lines |
|--------|---------:|-----------:|
| `K2Node_SpawnActorFromClass` | 86 | 86 |
| `K2Node_CreateWidget` | 79 | 79 |
| `K2Node_AssignmentStatement` | 60 | 60 |
| `K2Node_GenericCreateObject` | 47 | 47 |
| `K2Node_VariableSetRef` | 46 | 46 |
| `K2Node_SetFieldsInStruct` | 30 | 30 |
| `K2Node_GetDataTableRow` | 15 | 15 |
| `K2Node_PlayAnimationTimeRange` | 13 | 13 |
| `K2Node_LatentAbilityCall` | 11 | 11 |

Warning count == call-line count, confirming the warning emitter and
the generic call emitter both fire on the same path.

**Inconsistency:** `K2Node_AsyncAction_*` subclasses do NOT emit the
warning (they go through the typed async-action path), so the warning
emit lives in the `Unknown` branch of the dispatch — and that branch
should now be silent because the generic emit handles the case.

**Fix:** delete the warning emit at the unknown-fallback site in
`Decompiler/BpirDecompiler.cpp` (~line 1227 per prior audit). The
generic emit path already produces a usable call; the warning is
dead code that should have been removed when the umbrella fix
landed.

## Repro
- Cache root: `C:\Unity\unreal-fpv\.editor-automation\asset-dumps\`
- Grep `'Unknown node type (generic fallback)'` in any `bpir.txt`
- Example: `App\App\LevelBlueprints\B_DroneGameMode\bpir.txt`

## History
- `#1-initial-repro` `OPEN` reporter — Fresh 2026-05-04 audit found 399 spurious warning lines across 138 `bpir.txt` files. Warnings fire alongside correct generic call emit; left-over from before `B-bpir-decompiler-emitter-coverage-gap` and `F-bpir-add-generic-node-statement-form` promoted the generic handler. Removing the warning emit is sufficient.
- `#2-removed-spurious-warning-emit` `IN-REVIEW` developer — Removed dead warning emit at `BpirDecompiler.cpp:1519-1522`. The smart generic fallback already produces a correct call/pure/async line; the warning was a leftover from before `F-bpir-add-generic-node-statement-form` promoted the generic handler. Test `FDecompilerGenericFallbackNoWarningTest` exercises a `K2Node_SpawnActorFromClass` (top contributor) via `FBpirDecompiler::Decompile()` and asserts the warning is gone.
- `#3-verify-warning-removed` `DONE` tester — Verified: `blueprint.decompile` on `/App/App/LevelBlueprints/B_DroneGameMode` still emits `call K2Node_CreateWidget(...)` and no longer returns `Unknown node type (generic fallback)` warnings; only the unrelated delegate-signature skip warning remains.
