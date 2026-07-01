---
id: F-behavior-tree-decompile-btir
title: "Behavior Tree decompile-only text IR (BTIR)"
status: DONE
severity: Medium
category: feature
tags: [behavior-tree, ir, decompile, blackboard, ai]
---

# Behavior Tree decompile-only text IR (BTIR)

There is no Behavior Tree text IR today. Only imperative
`behavior_tree.*` and `ai.*` RPCs exist for spawning nodes and wiring
parents, plus generic `properties.json` falls out of asset dump because
`AssetDumpHandler.cpp:1666-1667` has no BT-specific sidecar registration.
For agents, that means BTs are write-only: you can mutate them blindly
but cannot read back a compact, diffable representation of "what does
this tree actually do?" the way BPIR/MGIR/AGIR give you for Blueprints,
materials, and anim graphs.

This ticket adds **decompile-only** (read) BTIR — a tree-shaped, nested
text grammar that mirrors `UBehaviorTree`'s editor graph topology 1:1.
Authoring (parse → mutate → recompile) is explicitly out of scope here;
the parallel `F-bt-create-blueprint-node-classes` and
`F-bt-attach-decorator-service-to-parent` tickets cover the imperative
write path. Together those three make BTs fully agent-authorable:
imperative for mutation, BTIR for inspection.

## Grammar sketch (tree-shaped, nested)

```
behavior_tree `BT_Patrol` {
  blackboard /Game/AI/BB_Patrol.BB_Patrol

  root selector `Root` @(0,0) {
    decorator BTDecorator_Blackboard `HasTarget` {
      blackboardKey: `Target`
      basicOperation: IsSet
      notifyObserver: OnResultChange
    }
    service BTService_DefaultFocus { keyName: `Target` }

    child sequence `Attack` @(-200,150) {
      decorator BTDecorator_Cooldown { coolDownTime: 0.5 }
      task BTTask_MoveTo `MoveToTarget` @(-260,300) {
        blackboardKey: `Target`
        acceptableRadius: 50.0
      }
      task /Game/AI/BTT_Strike.BTT_Strike_C `Strike` @(-140,300)
    }
  }
}

blackboard `BB_Patrol` {
  parent /Game/AI/BB_Base.BB_Base
  key Object  `Target`       { baseClass: /Script/Engine.Actor; instanceSynced: true }
  key Vector  `PatrolPoint`
}
```

## Design choices

- **Nested blocks, not parent-id flat lists.** BT is a strict tree;
  indentation maps 1:1 to graph topology, which makes diffs scan
  cleanly and removes the entire class of "dangling parent id" bugs
  AGIR/BPIR have to defend against.
- **Stock short forms** (`Sequence`, `Selector`, `SimpleParallel`,
  `Wait`, `MoveTo`, etc.) for engine classes; **full asset paths**
  (`/Game/AI/BTT_Strike.BTT_Strike_C`) for custom Blueprint subclasses.
  Class resolution reuses `ResolveClassByName` from
  `BehaviorTreeHandler.cpp:247-271`, which already handles both cases.
- **Decorators and services attach as inner blocks before the first
  `child`**, mirroring `FBTCompositeChild::Decorators[]` and the
  composite's `Services[]` — emission order matches in-editor order.
- **`FBTDecoratorLogic` (Test/And/Or/Not RPN):** emit bare when
  implicit-AND, trailing `decorator_logic:` block when explicit. This
  is the only non-tree-shaped sub-feature in BT.
- **Decompiler walks `BT->BTGraph->Nodes`** (editor-side
  `UBehaviorTreeGraphNode`) **not runtime `RootNode`** — the graph
  carries node positions, comments, and pre-compile state that the
  runtime tree drops.
- **`FBlackboardKeySelector` custom emit shim:** extract
  `SelectedKeyName: FName` and render as `` `KeyName` ``; suppress the
  ten-field property bag that generic reflection would otherwise dump.

## Implementation sketch

- **New files**
  - `Private/BTIR/BTIRDecompiler.{h,cpp}` — graph walker
  - `Private/BTIR/BTIRTextEmitter.{h,cpp}` — text formatter
  - `Private/Handlers/AI/BehaviorTreeDecompileHandler.cpp` — RPC entry
- **Reuse**
  - `IrCore/IrTextUtils` (identifier quoting, indentation, position
    annotations) — same primitives as AGIR/MGIR/BPIR
  - AGIR text-emitter pattern as structural template
  - Class resolver already in `BehaviorTreeHandler.cpp:247-271`
- **Sidecar registration:** `btir.txt` for `UBehaviorTree`; for
  standalone `UBlackboardData` assets, emit just the `blackboard { … }`
  block. Add to `AssetDumpHandler.cpp:1666-1667` registry (closes the
  current test gap where BT assets fall through to generic
  `properties.json`).
- **RPC:** `behavior_tree.decompile` (param: asset path; result:
  `{ btir: string }`).

**Effort:** ~600–900 LoC, 2–3 dev-days.

## Cross-references

**Sibling BT tickets (imperative authoring path):**
- `F-bt-create-blueprint-node-classes` — spawn BT/BB nodes from
  Blueprint-derived task/service/decorator classes
- `F-bt-attach-decorator-service-to-parent` — attach decorators and
  services to composite parents

**Soft prereqs (refactor / shared infrastructure):**
- `R-ircore-reflected-property-emit` — shared reflected-property
  formatter used by every IR emitter; BTIR property bodies depend on it
- `R-ir-grammar-harmonize` — keeps BTIR grammar consistent with
  AGIR/MGIR/BPIR conventions (backticks for names, `@(x,y)` for
  positions, indentation rules)
- `F-ir-authoring-guide` — public IR overview doc; BTIR section lives
  there once shipped

**Asset-dump gap (independent but adjacent):**
- `R-asset-dump-sidecar-registry` — BT entry will land here when
  BTIR ships

## History
- `#1-initial-spec` `OPEN` reporter — Filed BTIR decompile-only text IR
  spec. Tree-shaped nested grammar, stock-class short forms vs
  full-path custom classes, decorator/service inner blocks, decorator
  logic RPN, blackboard sub-block. Walks `BT->BTGraph->Nodes` so node
  positions and editor-only state survive. New files under
  `Private/BTIR/` plus `BehaviorTreeDecompileHandler.cpp`; sidecar
  `btir.txt`; RPC `behavior_tree.decompile`. ~600–900 LoC, 2–3 dev-days.
  Pairs with siblings `F-bt-create-blueprint-node-classes` and
  `F-bt-attach-decorator-service-to-parent` to make BTs fully
  agent-authorable.
- `#2-dual-surface-mandate` `OPEN` reporter 2026-05-13 — Locking in the
  **dual-surface invariant** before implementation. BOTH surfaces are
  MANDATORY, neither is optional: (a) asset-dump sidecar `btir.txt`
  registered in `AssetDumpHandler.h` `DumpFileNames` and emitted from
  the `UBehaviorTree` / `UBlackboardData` dispatch branches in
  `AssetDumpHandler.cpp:1666-1667`, written under
  `.editor-automation/asset-dumps/.../<asset>/btir.txt`; (b) MCP RPC
  `behavior_tree.decompile` handler at
  `Private/Handlers/AI/BehaviorTreeDecompileHandler.cpp`, callable via
  `call("behavior_tree.decompile", { assetPath })`, returns
  `{ ir, warnings }`. Both surfaces MUST share ONE builder function
  `BuildBehaviorTreeIrText(UBehaviorTree*) → FIrResult { Text, Warnings, bSuccess }`
  (plus a sibling `BuildBlackboardIrText(UBlackboardData*) → FIrResult`
  for standalone blackboard assets) living in
  `Private/BTIR/BTIRDecompiler.cpp`. The sidecar pipeline calls it for
  dump emission; the RPC dispatch calls it for direct response. Zero
  divergence between the two — never ship one without the other, never
  let the two implementations drift. If the dump branch needs format
  differences, route them through builder options, not a parallel
  implementation.
- `#3-btir-decompile-surfaces` `IN-REVIEW` developer — Added
  shared `BTIRDecompiler::BuildBehaviorTreeIrText` and
  `BuildBlackboardIrText` builders, wired `behavior_tree.decompile` to
  return `{ ir, warnings }`, emitted asset-dump `btir.txt` for both
  Behavior Tree and standalone Blackboard assets while keeping
  `properties.json`, and added regression coverage for RPC/dump parity plus
  standalone Blackboard dumps.
- `#4-verify-btir-surfaces` `DONE` tester — Verified: created temp assets under `/Game/McpReview/BTIRVerify_FBehaviorTreeDecompileBtir`, ran `behavior_tree.decompile` on the Behavior Tree and observed `ir`/`text` BTIR plus warning `Behavior Tree root has no child node.`, then ran `asset.dump` with `outRoot=C:/tmp/mcp-verify-btir` and observed `btir.txt` in both Behavior Tree and standalone Blackboard dump `writtenPaths`; deleted the temp content folder afterwards.
