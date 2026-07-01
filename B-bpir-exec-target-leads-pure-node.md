---
id: B-bpir-exec-target-leads-pure-node
title: "BPIR exec-target to a label that leads with a pure-emitting `call` node (e.g. K2Node_ConvertAsset) is rejected — label resolution picks the pure node and leaves the exec edge unwired"
status: OPEN
severity: Medium
category: bug
tags: [bpir, compiler, wireexecpins, find-first-impure, pure-node, exec-target, convertasset]
encounters: 1
lastSeen: 2026-06-26T08:37:31Z
---

# Exec-target label leading with a pure node leaves the edge unwired

When a `cast<T>(...) [success -> @label]` (or any labeled exec target) points at a
block whose FIRST instruction is a `call` to a node that emits a PURE K2Node — one
with no exec pins, e.g. `%x = call K2Node_ConvertAsset(...)` — the compiler hard-fails:

```
[COMPILE_FAILED] Line N: Post-wire verification: labeled exec target 'success -> @label'
on Cast was not wired (target label resolves to inst[M])
```

This is valid IR: a pure binding floats as a data dependency, so the exec edge should
skip it and attach to the first IMPURE instruction in the block. Instead the compiler
selects the pure node as the target, finds it has no exec INPUT pin, silently drops the
edge, then the post-wire verification (added by `B-bpir-statement-cast-success-unwired-replace-shared-topology`)
catches the unwired pin and errors. The error is also opaque — `inst[M]` index is not
visible to the author.

## Root cause

`FindFirstImpureAtLabel` (`BpirCompiler.cpp:7395-7445`) selects a target when
`bBpirImpure || bNodeImpure` (line 7438). For `call K2Node_ConvertAsset`, the authored
opcode is `Call`, and `IsImpure(Call) == true` (`BpirTypes.h:72`), so `bBpirImpure` is
true even though the emitted node is pure (`NodeHasExecPins` false — it checks any exec
pin, `BpirCompiler.cpp:133`). The function returns the pure node's index. `ResolveTargetExecInput`
(`BpirCompiler.cpp:6971`) then calls `GetExecInputPin` (`BpirCompiler.cpp:7451`), which
returns null (no exec INPUT pin), so Step 3 hits `if (!TargetExecIn) continue;`
(`BpirCompiler.cpp:7188-7192`) and skips the wire. The disjunction was added to fix the
CONVERSE case (opcode flipped to Pure but node HAS exec pins); the `bBpirImpure` half
trusts the authored opcode and so mis-selects an impure-opcode node that emitted pure.

## Repro

`blueprint.insert_bpir_at_node` on `/App/App/UI/LobbyAndMenu/W_LyraFrontEnd` with a block
where `@closedo:` leads with `%overlay = call K2Node_ConvertAsset(...)` followed by
`call PopContentFromLayer(...)`, targeted by an upstream `cast<...> [success -> @closedo]`,
fails with the error above (`resolves to inst[6]`, the ConvertAsset binding).

**Workaround:** Move the pure `call K2Node_ConvertAsset(...)` binding ABOVE the `@closedo:`
label (into the predecessor block) so the labeled block leads with the impure
`call PopContentFromLayer(...)`. Same logic, compiles clean (`compiled:true, UpToDate`).

**Fix:** In `FindFirstImpureAtLabel`, a node is a valid labeled exec TARGET only if it has
an exec INPUT pin — skip instructions whose emitted node lacks one even when the authored
opcode reports impure, so resolution advances to the first genuinely-impure instruction.
Secondary: make the error name that the resolved target is pure / has no exec input and
tell the author to lead the block with an impure instruction, instead of the opaque
`inst[M]` index.

## History
- `#1-initial-repro` `OPEN` reporter — Verified against compiler source: `FindFirstImpureAtLabel` (`BpirCompiler.cpp:7438`) selects a target on `bBpirImpure || bNodeImpure`; for `call K2Node_ConvertAsset` the `Call` opcode is `IsImpure`-true (`BpirTypes.h:72`) but the emitted node is pure (no exec input pin), so it is mis-selected, `GetExecInputPin` returns null (`BpirCompiler.cpp:7451`), Step 3 silently skips the wire (`BpirCompiler.cpp:7188-7192`), and the post-wire verification (`BpirCompiler.cpp:7287`) errors with `resolves to inst[M]`. Distinct from the DONE `B-bpir-statement-cast-success-unwired-replace-shared-topology` (opposite direction of the same disjunction: that one needed `bNodeImpure` ADDED for an opcode-flipped-to-pure node that HAD exec pins, silent corruption requiring shared cross-entry topology; this one needs the `bBpirImpure` half GATED on a real exec input pin, loud compile failure on a single fresh insert). Repro: `blueprint.insert_bpir_at_node` on `/App/App/UI/LobbyAndMenu/W_LyraFrontEnd` with `@closedo:` leading with the pure ConvertAsset binding. Workaround proven: move the pure binding above the label.
- `#2-exec-input-pin-gate` `IN-REVIEW` developer — Fixed in `FindFirstImpureAtLabel` (`Private/Compiler/BpirCompiler.cpp`): label-target selection now gates on a real exec INPUT pin via a new file-local `NodeHasExecInputPin` helper (added next to `NodeHasExecPins`), replacing the `bBpirImpure || bNodeImpure` disjunction. A pure-emitting `call K2Node_*` (e.g. ConvertAsset / GetEnumeratorName) keeps opcode Call but has no exec pins, so it is now skipped and resolution advances to the first instruction that can actually receive the edge. This subsumes both halves of the old predicate and preserves the sibling DONE fix `B-bpir-statement-cast-success-unwired-replace-shared-topology`: an opcode-flipped K2Node_DynamicCast still has an exec input pin and is still selected. The rewrite also drops the stale `BpirCompiler.cpp:434` comment reference. Regression test added: `Private/Tests/Bpir/TestBpirExecTargetLeadsPureNode.cpp` (`PinWright.bpir.compiler.integration.ExecTargetLeadsPureNode`) compiles `cast<Character>(%pawn) [success -> @body]` where `@body` leads with `%name = call K2Node_GetEnumeratorName()` (pure, no exec input) followed by `set Counter` — asserts the compile succeeds (pre-fix the post-wire verification error fired), the cast's success exec pin is wired, and it lands on the impure `UK2Node_VariableSet`, skipping the pure node. Not compiled/tested here (later phase).
- `#3-attempt-failed` `OPEN` developer — Auto-fix attempt reached STUCK-COMPILE; reverted and NOT pushed (build/tests not green).
