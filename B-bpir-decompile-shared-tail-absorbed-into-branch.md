---
id: B-bpir-decompile-shared-tail-absorbed-into-branch
title: "Decompiler absorbs shared-tail label into one branch's block, the other branch's exec -> @label then re-runs that branch's prior statements"
status: DONE
severity: High
category: bug
tags: [bpir, decompiler, reconvergence, round-trip, silent-corruption]
---

# Decompiler absorbs shared-tail label into one branch's block, the other branch's `exec -> @label` then re-runs that branch's prior statements

When two branches converge on the same downstream node (graph has two predecessors on one exec input), the decompiler's `WalkExecChain` walks the first-encountered branch greedily, **absorbs the shared join node inline as a tail statement of that branch's label block**, and records `NodeToEmittedLabel[JoinNode] = <first-branch-label>`. When the second branch's walk reaches the same (now-visited) join node, the reconvergence handler emits `exec -> @<first-branch-label>` (BpirDecompiler.cpp:957-1003). That target label's block contains the first branch's body **before** the shared tail — so a compile of the decompiled BPIR re-wires the second branch through the first branch's statements. Net effect: silent control-flow corruption on decompile -> recompile round-trip.

The compiler is fine. The graph the compiler builds from the original BPIR is correct (two predecessors fanning into one successor — standard reconvergence). The bug is purely in the decompiler's choice of *which* label to name as the reconvergence target. The right label is the one that starts **at** the join node; the current code names the upstream-branch label that merely *contains* the join node inline.

The bpir-language-reference §2.8 documents this shape explicitly: an authored `@else:` block with no terminator "falls through to @done (next label)", and `@then` jumps with `exec -> @done`. Both branches converge on `@done`'s first node. A faithful decompile must surface `@done` as its own label; the current decompiler erases it.

## Repro

Session repro on `/App/App/UI/W_PhotoPopup.W_PhotoPopup` `ShowPopup`.

**Input BPIR (round-trip-equivalent shape; matches what an author would write per the language reference):**
```
%b = branch(%allowed) [true -> @hide_reason, false -> @show_reason]

@show_reason:
    call SetVisibility(Target: $ReasonText, InVisibility: ESlateVisibility::SelfHitTestInvisible)
    exec -> @show_self

@hide_reason:
    call SetVisibility(Target: $ReasonText, InVisibility: ESlateVisibility::Collapsed)
    # falls through to @show_self (documented language behavior, §2.8)

@show_self:
    call SetVisibility(InVisibility: ESlateVisibility::SelfHitTestInvisible)
```

After `blueprint.compile_bpir` followed by `blueprint.decompile`, the decompiler emits:
```
@then:                                                          # was @hide_reason
    call SetVisibility(Target: $ReasonText, InVisibility: Collapsed)
    call SetVisibility(InVisibility: SelfHitTestInvisible)      # @show_self absorbed inline

@else:                                                          # was @show_reason
    call SetVisibility(Target: $ReasonText, InVisibility: SelfHitTestInvisible)
    exec -> @then                                               # WRONG — should be exec -> @<show_self_label>
```

Recompiling that output produces a graph where the `@else` branch's exec-out wires to the *first* node of `@then`, not to the shared SetSelf-visible node. Runtime symptom with `allowed = false` (player photographed the wrong target): the player should see the reason text appear. Observed sequence: `SetReason=Visible` → `exec @then` → `SetReason=Collapsed` (immediately overrides) → `SetSelf=Visible`. Reason text never displays.

**Workaround used in this session:** hoist `SetVisibility(InVisibility: SelfHitTestInvisible)` (the shared tail) *above* the branch so the branch has no shared join.

## Root cause location

`Source/PinWright/Private/Decompiler/BpirDecompiler.cpp`:

- `WalkExecChain` lines 907-1003 — the chain walker eagerly absorbs whatever's reachable through the current exec output, regardless of whether that node has additional incoming exec predecessors. There is no pre-pass that identifies multi-predecessor join nodes and forces them to start a fresh label.
- Lines 957-1001 — reconvergence handler. When the second branch's walk hits a visited node, the code emits `exec -> @<NodeToEmittedLabel[Node]>`. That mapping points to whatever label was active when the node was first absorbed, which for a shared tail is the wrong label — it's the upstream branch's label, not a label that starts at the join.
- Lines 968-991 (the `NewLabel = State.AllocLabel(TEXT("merge"))` retroactive-insert path) **only** handles the case where the join node was emitted in the pre-label (initial) block. It does not handle the case where the join was emitted inside a labeled block. That's the gap.

## Fix sketch

In `WalkExecChain`, before emitting an unvisited non-tunnel node into the current label, check whether the reached target exec input pin is shared (`EGPD_Input`, exec category, `LinkedTo.Num() > 1`).

If the join node does not already have a reserved label, allocate a fresh `merge_N` label, record `NodeToEmittedLabel[JoinNode] = merge_N`, queue that label with the current source exec output pin, emit `exec -> @merge_N`, and stop the current branch walk. Do not queue the target exec input pin; `PendingLabels` stores source exec output pins.

If another branch reaches the same unvisited reserved join before merge processing, emit `exec -> @merge_N` without queueing a duplicate label. When the queued `merge_N` walk runs, let the reserved node emit normally under that label and continue through the shared tail.

## Test plan

Add a round-trip test under `Source/PinWright/Private/Tests/`:
1. Compile the input BPIR from the repro section.
2. Decompile.
3. Assert the decompiled output contains a label whose block holds only the shared `SetVisibility(InVisibility: ...)` call (not preceded by either `SetVisibility(Target: $ReasonText, ...)` variant).
4. Assert both `@then` and `@else` (or however the branch labels are named) terminate with `exec -> @<shared_label>`, not `exec -> @<each_other>`.
5. Compile the decompiled output and walk the resulting graph: assert the `false`-side exec-out of the branch reaches the shared `SetVisibility` node without traversing any `SetVisibility(Target: $ReasonText, InVisibility: Collapsed)` node.

## History
- `#1-initial-repro` `OPEN` reporter — Decompiler `WalkExecChain` (BpirDecompiler.cpp:907-1003) greedily absorbs shared join nodes inline into the first-walked branch, then routes the other branch's `exec ->` to that branch's label instead of to a fresh label at the join. Round-trip recompile produces a graph that re-runs the first branch's body when the second branch runs. Repro on `/App/App/UI/W_PhotoPopup.W_PhotoPopup ShowPopup` — reason text fails to show when `allowed=false` because the `else` branch jumps into the `then` branch and immediately collapses the text it just made visible. Existing retroactive-label-insert (lines 968-991) only covers join nodes in the pre-label block, not joins inside labeled blocks. Graph is correct; emission is wrong; round-trip corrupts behavior. See bpir-language-reference §2.8 for the documented language shape this should support.
- `#2-dedicated-merge-label` `IN-REVIEW` developer — `WalkExecChain` now reserves a dedicated merge label for unvisited shared exec-input join nodes using the current source exec output pin, emits branch-local `exec -> @merge_N`, lets the reserved merge label emit the shared tail normally, and adds `FDecompilerSharedTailReconvergenceLabelTest` to guard the decompile -> recompile control-flow shape.
- `#3-verify-fix` `DONE` tester — Verified: compiled BPIR `entry custom_event TestSharedTail(bool allowed)` with two-branch shared-tail shape (both `@show_reason`/`@hide_reason` doing `exec -> @show_self`) into temp BP `/Game/App/UI/Test/W_McpVerifyTemp_bpir_decompile_shared_tail`, then `blueprint.decompile` emitted a dedicated `@merge` label containing only the shared `PrintString("show_self")` call, with both `@then` and `@else` terminating in `exec -> @merge` (not `exec -> @<each_other>`). Matches the fix sketch and acceptance criteria from the ticket body.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 2 body citations repointed in place; every rewritten path was confirmed to exist at plugin HEAD `ef8a1f1b`. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
