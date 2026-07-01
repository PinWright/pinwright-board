---
id: E-decompile-unreachable-output-pin-labels
title: "`blueprint_decompile` emits exec output pin labels with no defined blocks, breaking round-trip recompile"
status: DONE
severity: Medium
category: ergonomic
tags: [bpir, decompile, round-trip, k2node-asyncaction]
---

# `blueprint_decompile` emits exec output pin labels with no defined blocks, breaking round-trip recompile

`blueprint_decompile` of a `K2Node_AsyncAction` call emits all three exec output pins in the label list — even when only one of them (e.g. `then`) has a downstream body wired. The decompile output looks like:

```
%n1 = call K2Node_AsyncAction(OwningPlayer: %n0, WidgetClass: ..., LayerName: ..., bSuspendInputUntilComplete: true) [AfterPush -> @afterpush, BeforePush -> @beforepush, then -> @then]
```

But `@beforepush` and `@then` are NOT emitted as block labels anywhere else in the body — only `@afterpush` is. The decompile compiles successfully through UE's built-in BP compiler (those unused exec pins just have no connections), but the BPIR compiler **rejects** the same text on recompile:

```
COMPILE_FAILED: Line 3: Undefined label reference: @beforepush; Line 3: Undefined label reference: @then
```

So the decompiled output is not round-trippable. The typical MCP workflow of "decompile existing → copy + edit → recompile to extend" requires manually stripping all unreferenced output pin labels from the output pin list before recompile will succeed.

Discovered a second facet while trying to fix: adding empty body blocks (`@beforepush: exec -> @then; @then: nop`) also fails — `nop` is not a recognized BPIR instruction.

## Repro

1. `mcp__editor_automation__.call path="blueprint.decompile" args={"assetPath":"/App/App/UI/LobbyAndMenu/W_LyraFrontEnd","graphName":"EventGraph"}` — note any `K2Node_AsyncAction` call; observe output pin list includes labels for pins with no downstream.
2. Copy the relevant event/function body verbatim, paste into `blueprint_compile_bpir` with `mode: append`.
3. Error: `Undefined label reference: @beforepush; @then`.
4. Workaround: delete the unreferenced labels from the output pin list, leaving only `[AfterPush -> @afterpush]`.

**Proposed fix:** `blueprint_decompile` should only emit output pin labels for pins that have a downstream body (i.e., labels it actually writes a block for). The current behavior likely mirrors UE's internal pin list one-to-one, but for round-trip it should omit unused-exec pins.

**Alternative:** BPIR compiler could accept "dangling label references" (exec output targeting an undefined label) as a no-op fall-through / terminal exec. Would preserve round-trip without changing the decompiler.

## History
- `#1-undefined-label-roundtrip-fail` `OPEN` reporter — Hit during replay subtask #4. Decompiled `W_LyraFrontEnd` click handler for `W_ReplayEditorButton`, copied the K2Node_AsyncAction pattern verbatim, recompile failed with undefined-label errors. Had to strip BeforePush/then labels. Adding empty blocks with `nop` also failed ("Unrecognized instruction: nop"). Shipped with `[AfterPush -> @afterpush]` only, which compiled cleanly.
- `#2-skip-disconnected-exec-labels` `IN-REVIEW` developer — In `BpirDecompiler::WalkExecChain` generic-unknown fallback (Private/Decompiler/BpirDecompiler.cpp around line 1172), added the same `ExecOut->LinkedTo.Num() == 0` skip that the Latent, Timeline, and DoOnce/Gate/FlipFlop paths already use. `K2Node_AsyncAction` classifies as `ENodeSemantics::Unknown` (it's not a `UK2Node_CallFunction`), so it lands in this fallback and was the only path emitting labels for disconnected exec pins. Fix also removes the unused `LabelMap.Add` call that happened before the connected-pin check.
- `#3-verified-only-connected-labels` `DONE` tester — Verified on `/Game/App/UI/Test/W_McpVerifyTemp`. Compiled a custom event containing `call K2Node_AsyncAction(..., bSuspendInputUntilComplete: true) [AfterPush -> @afterpush]` with only the `AfterPush` exec wired (BeforePush/then disconnected). `mcp__editor_automation__.call path="blueprint.decompile" args={"assetPath":".../W_McpVerifyTemp","graphName":"EventGraph"}` produced the K2Node_AsyncAction line with exec label list `[AfterPush -> @afterpush]` only — no `BeforePush` or `then` labels emitted. The decompile output is round-trippable without manual label-list surgery. Warning list correctly includes `"Unknown node type (generic fallback): K2Node_AsyncAction"` (the classification behind the fix) alongside the existing orphan-node warnings.
