---
id: E-bt-root-entry-node-undiscoverable
title: "Behavior Tree hidden Root entry node is undiscoverable; documented authoring flow yields an all-orphaned tree"
status: IN-REVIEW
severity: High
category: ergonomic
tags: [behavior-tree, discoverability, connect_nodes, add_node, root, ai]
---

# Behavior Tree hidden Root entry node is undiscoverable; documented authoring flow yields an all-orphaned tree

`behavior_tree.create` produces a `UBehaviorTree` whose graph already contains
a hidden entry **Root** composite node (created by
`CreateDefaultNodesForGraph` in `BehaviorTreeHandler.cpp:422`). For the tree to
actually run, the agent's top composite (e.g. the root Selector) must be wired
as a child of that Root entry node. But there is **no discoverable way to learn
the Root node's id or name** through the documented MCP surface:

- The wiki pages for `behavior_tree.create`, `behavior_tree.add_node`, and
  `behavior_tree.connect_nodes` never mention that a Root entry node exists or
  that it must be connected. They describe a clean "add nodes, connect parent →
  child" model with no entry-point node.
- `behavior_tree.create` returns only `{assetPath, name, saved, ...}` — it does
  **not** return the Root node's id or name.
- There is no `behavior_tree.list_nodes` / inspect RPC to enumerate the graph
  nodes and surface the Root. (`behavior_tree.decompile` is read-only output; it
  does not emit the Root node's addressable id/name either.)

The net effect: an agent that follows the documented `create` → `add_node` →
`connect_nodes` flow builds a tree where **every node is orphaned and the tree
does not run**, with no documented signal of what went wrong or how to fix it.
The only repair — connecting the Root — requires knowing the magic name
`BehaviorTreeGraphNode_Root_0`, which `connect_nodes` accepts via
`FindBTGraphNode` (`BehaviorTreeHandler.cpp:45-60`, matches on `Node->GetName()`)
but which is documented **nowhere**. In the reported attempt the agent only
recovered by reading the plugin C++ source to discover that name.

`behavior_tree.decompile` (the path that surfaces the problem) is working
correctly — its warning is accurate and is the only thing that reveals the
breakage. The ergonomic defect is on the **authoring** side
(`create` / `add_node` / `connect_nodes`): the mandatory Root entry node is
invisible, unreturned, and unlistable.

## Quotable demonstration (verbatim repro via mcp__editor-automation__call)

1. `behavior_tree.create { name: "BT_OracleReplay", savePath: "/Game/McpOracle" }`
   → success. Result contains **no Root node id**:
   `{"assetPath":"/Game/McpOracle/BT_OracleReplay","name":"BT_OracleReplay","saved":true, ...}`

2. `behavior_tree.add_node { assetPath: ".../BT_OracleReplay.BT_OracleReplay", nodeType: "Selector", x: 0, y: 200 }`
   → success, `{"nodeId":"A3670D1246C4FBFA4947BA8111165CCC", ...}`

3. `behavior_tree.decompile { assetPath: ".../BT_OracleReplay.BT_OracleReplay" }` → the tree is broken, but the only hint is:
   - `ir`: `"behavior_tree \`...\` {\n  orphan composite Selector Selector @(0, 200) ()\n}\n"`
   - `warnings`: `["Behavior Tree root has no child node.", "Behavior Tree graph node 'BehaviorTreeGraphNode_Composite_0' is unreachable from root and emitted as orphan."]`

   The warning `"Behavior Tree root has no child node."` names no fix: it does
   not say there is a Root entry node, does not give its name, and does not say
   to call `connect_nodes` with it.

4. The ONLY repair is the undocumented magic name:
   `behavior_tree.connect_nodes { parentNodeId: "BehaviorTreeGraphNode_Root_0", childNodeId: "<selector guid>" }`
   → success; a re-`decompile` then reports `root composite Selector ...` with
   `warnings: []`.

The string `BehaviorTreeGraphNode_Root_0` appears in no wiki page and is not
returned by any RPC — an agent cannot derive it without reading C++ source.

**Workaround:** call
`behavior_tree.connect_nodes` with `parentNodeId: "BehaviorTreeGraphNode_Root_0"`
to link the hidden Root entry node to your top composite.

**Fix (pick one or more):**
- Have `behavior_tree.create` return the Root node's id/name in its result
  (e.g. `rootNodeId`), so callers can immediately connect to it.
- Add a node-listing/inspection RPC (e.g. `behavior_tree.list_nodes`) that
  enumerates graph nodes including the Root entry node and its addressable
  name/GUID.
- Auto-connect the first added top-level composite/task to the Root when the
  Root has no child (or expose an explicit `connect_to_root` convenience), and
  document the entry-node model on the `create`/`add_node`/`connect_nodes`
  wiki pages.
- At minimum, extend the decompile warning text to name the fix
  ("connect node 'BehaviorTreeGraphNode_Root_0' to your root composite") and
  document the Root entry node in the `behavior_tree` overlay.

## History
- `#1-initial-repro` `OPEN` reporter — Replayed via mcp__editor-automation__call on `/Game/McpOracle/BT_OracleReplay`: `create` returns no Root id; `add_node` Selector + `decompile` yields an all-orphaned tree with warning `"Behavior Tree root has no child node."`; the only repair is connecting `parentNodeId: "BehaviorTreeGraphNode_Root_0"` — a name documented in no wiki page and returned by no RPC. The hidden mandatory Root entry node is invisible/unreturned/unlistable through the documented `create`→`add_node`→`connect_nodes` surface, so the documented authoring flow silently produces a non-running tree. Seed `behavior_tree.decompile` is correct (accurately surfaces the orphan); the ergonomic defect is on the authoring/discovery side. Temp asset deleted after replay.
- `#2-fix-root-discoverable` `IN-REVIEW` developer — Fixed via option 1 (surface the Root through the RPC result). `behavior_tree.create` now locates the seeded `UBehaviorTreeGraphNode_Root` (new `FindBTRootNode` helper, reusing the existing locator pattern) and returns `rootNodeId` (GUID), `rootNodeName` (the addressable `GetName()` that `connect_nodes` matches), and a `rootHint` string in its result; the handler summary now documents that the top composite must be connected to the Root. Documented the Root entry-node model with a new `## Root entry node` section in the `behavior_tree` wiki overlay (`Docs/wiki-src/behavior_tree.md`) showing the create result and the Root→composite connect call. Files: `Source/EditorAutomationRpcGateway/Private/Handlers/AI/BehaviorTreeHandler.cpp`, `Docs/wiki-src/behavior_tree.md`. Regression test: `FBTCreateReturnsRootNodeTest` (`EditorAutomationRpcGateway.behavior_tree.create.ReturnsConnectableRootNode` in `Private/Tests/Gameplay/TestBehaviorTreeAttachSubnodes.cpp`) asserts the create result carries non-empty `rootNodeId`/`rootNodeName`, then connects Root→Selector using ONLY the RPC-returned `rootNodeId` (no graph reflection) and verifies the runtime root composite gains one child — fails at the first assertion if the fix is reverted. Not compiled/run here (later phase).
