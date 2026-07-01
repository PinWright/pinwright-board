---
id: B-bpir-dynamic-cast-unknown-diagnostic
title: "BPIR decompiler emits cast<Unknown> for bad DynamicCast nodes without structured diagnostics"
status: DONE
severity: Low
category: bug
tags: [bpir, decompiler, diagnostics, cast]
---

# BPIR decompiler emits cast<Unknown> for bad DynamicCast nodes without structured diagnostics

Some invalid DynamicCast nodes decompile as `cast<Unknown>` with no structured context. The visible marker is useful, but without node id, graph, or title it is not actionable from dump output alone.

**Workaround:** Use live graph search and node details to identify the bad cast node.

**Fix:** Preserve the visible `cast<Unknown>` signal but also emit a structured warning with asset path, graph name, node id, node title, and the missing target-type reason.

## History
- `#1-fresh-app-game-dump-audit` `OPEN` reporter — Fresh dumps contain 3 `cast<Unknown>` lines: `/Game/Audio/_Script/Aud_Trigger_Spawner`, `/Game/Audio/_Script/Aud_Music_Trigger`, and `/Game/Blueprints/GamePlay/BP_RobotHend`. Live `get_node_details` on `Aud_Music_Trigger` confirms the node is `K2Node_DynamicCast_0` titled `Bad cast node` with no target output pin subtype. BPIR should preserve the visible `cast<Unknown>` signal but also emit a warning with nodeId, graphName, asset path, and title so this is actionable as asset corruption/invalid cast state rather than a silent type-resolution hole.
- `#2-structured-warning-on-null-targettype` `IN-REVIEW` developer — Added structured warning emission in BpirDecompiler.cpp WalkExecChain Cast branch when UK2Node_DynamicCast::TargetType is null; warning carries asset path, graph name, class, node title, nodeId following the orphan-warning convention. Visible cast<Unknown> emission in BpirTextEmitter::EmitCast unchanged. Regression test TestBpirDecompilerCastUnknownDiagnostic.cpp validates both channels.
- `#3-verify-fix` `DONE` tester — Verified: blueprint.decompile on /Game/Audio/_Script/Aud_Music_Trigger returns BPIR containing `cast<Unknown>($OtherActor)` (visible signal preserved) AND warnings array contains structured entry: "Cast node has no resolved target type (emitted as cast<Unknown>): asset=/Game/Audio/_Script/Aud_Music_Trigger.Aud_Music_Trigger graph=EventGraph class=K2Node_DynamicCast 'Bad cast node' nodeId=8009E042-403C-7333-16EF-049CE9CCF391". Both channels confirmed.
