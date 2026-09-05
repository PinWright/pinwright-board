---
id: F-author-enhanced-input-action-node
title: "No way to author a bound K2Node_EnhancedInputAction: BPIR has no entry kind for it, create_node cannot set InputAction, set_node_property rejects it, Python refuses it as protected — and after property.set the node keeps STALE pins because no verb reconstructs a node"
status: IN-REVIEW
severity: High
category: feature
tags: [blueprint, enhanced-input, K2Node_EnhancedInputAction, bpir, create_node, set_node_property, reconstruct-node, stale-pins, ue5.8]
---

# Authoring an Enhanced Input event node needs a four-step workaround, and step 3 is `asset.save` + `asset.reload`

## The gap

Enhanced Input is the only supported input path in UE5, and in a Blueprint-only
project the **only** way to receive it is a `UK2Node_EnhancedInputAction` entry node
per `UInputAction`. Four surfaces that look like they should author one all fall
short:

1. **BPIR has no entry kind for it.** `bpir.entry-points` lists `event`,
   `custom_event`, `function`, `macro`, `construction`, `component_event`,
   `widget_event`, `key_pressed`, `key_released`. There is no
   `enhanced_input`/`input_action` entry, and `entry event <IA name>()` does not
   produce one. So the single richest authoring surface cannot express the node
   every UE5 input graph starts with.
2. **`blueprint.graph.create_node` creates the node but cannot bind it.** Passing
   `nodeType: "K2Node_EnhancedInputAction"` works (generic class fallback) and
   returns a node id, but the parameter list has no `inputAction` field — only
   `inputAxisName` for the *legacy* `InputAxisEvent`. The node is created with
   `InputAction == nullptr`, and `AllocateDefaultPins` therefore gives it the
   fallback shape: `ActionValue` typed **bool**.
3. **`blueprint.graph.set_node_property` refuses the property.**
   `set_node_property {propertyName: "InputAction", value: "/Game/.../IA_Move.IA_Move"}`
   -> `[PROPERTY_NOT_SUPPORTED] Unsupported node property 'InputAction'`. Its wiki
   page documents the parameter only as "Property name (Comment, X, Y, etc.)"; the
   "etc." is misleading — it is a small whitelist, not reflection.
4. **UE Python refuses it too.** `node.set_editor_property('InputAction', ia)` ->
   `Property 'InputAction' for attribute 'InputAction' on 'K2Node_EnhancedInputAction'
   is protected and cannot be set` (and the snake_case `input_action` does not exist
   at all). `unreal.BlueprintEditorLibrary` has no `refresh_all_nodes` in UE 5.8, and
   `ReconstructNode` is not a `UFUNCTION`, so `object.call_function` cannot reach it.

The generic reflected writer **does** work — `property.set` on the node's object path:

```
call("property.set", {
  objectPath: "/Game/FPS/Player/BP_FPSCharacter.BP_FPSCharacter:EventGraph.K2Node_EnhancedInputAction_0",
  propertyName: "InputAction",
  value: "/Game/FPS/Player/Input/IA_Move.IA_Move"})
-> {"applied": true, "markedDirty": true}
```

…but that is where the second half of the defect starts.

## The stale-pin trap

After `property.set` the node's **title** updates (`"EnhancedInputAction IA_Move"`)
while its **pins do not**. `blueprint.compile` does not fix it:

```
get_node_details -> nodeTitle: "EnhancedInputAction IA_Move"
                    pins: [... {"pinName":"ActionValue","pinType":"bool"} ...]   # WRONG, IA_Move is Axis2D
```

`AllocateDefaultPins` already ran with `InputAction == nullptr`; nothing re-runs it.
There is **no PinWright verb that reconstructs a node**:
`blueprint.graph.replace_node` explicitly refuses "bound events
(component/actor/generated/input)", `set_node_property` has no reconstruct, and
`blueprint.compile` leaves the pins alone. A caller who stops here silently wires a
`bool` where a `Vector2D` belongs.

The only thing that fixes it is a **package round-trip**:

```
call("asset.save",   {assetPath: ".../BP_FPSCharacter", force: true})
call("asset.reload", {assetPath: ".../BP_FPSCharacter"})
get_node_details -> pins: [... {"pinName":"ActionValue","pinType":"struct","pinSubType":"Vector2D"},
                               {"pinName":"InputAction","pinType":"object","defaultValue":"IA_Move"} ...]
```

Post-reload the pins are right and `blueprint.insert_bpir_at_node` can wire the
`Triggered` / `Started` / `Completed` exec pins, with `ActionValue` reachable by bare
name (that part works well and is correctly documented).

## Asked for

- **Preferred:** a BPIR entry kind, e.g.
  `entry input_action /Game/FPS/Player/Input/IA_Move [Triggered -> @move, Completed -> @stop] { ... }`,
  so the node can be authored and round-tripped like every other entry.
- **Minimum:** an `inputAction` parameter on `blueprint.graph.create_node` (sibling of
  the existing legacy `inputAxisName`), applied **before** `AllocateDefaultPins` the
  way `compile_bpir`'s `node_props` shape-determining table already does for other
  K2Nodes.
- **Either way:** something that reconstructs a node after a property write — a
  `reconstruct: true` flag on `set_node_property`, or a
  `blueprint.graph.reconstruct_node` verb. Telling callers to save-and-reload the
  whole package to fix one node's pins is a trap, and on a large Blueprint it is slow
  (the BP here was 935 KB by that point).
- **Docs:** `blueprint.graph.set_node_property`'s `propertyName` description should
  say it is a fixed whitelist and name the members, instead of "Comment, X, Y, etc.".

## Workaround used (works, verified)

1. `blueprint.graph.create_node {nodeType:"K2Node_EnhancedInputAction", x, y}` x10
2. `property.set` `InputAction` on each node's object path
3. `asset.save {force:true}` then `asset.reload` — pins reallocate
4. `blueprint.insert_bpir_at_node {nodeId, execPin:"Triggered"|"Started"|"Completed", code:"call Handler(Value: ActionValue)"}`

severity rationale: impact=blocks the standard UE5 input path in Blueprint-only
projects, with a silent wrong-pin-type failure mode in the obvious partial workaround
x reach=every UE5 project that takes player input -> High

## History
- `#1-filed` `OPEN` reporter — Hit building the first-person player for the FPS stream on UE 5.8 / EAContentExamples58, where `Docs/fps/PLAN.md` mandates Blueprint-only authoring so `K2Node_EnhancedInputAction` is the only route to player input. Walked all four candidate surfaces and recorded each verbatim: BPIR has no matching `entry` kind (`bpir.entry-points` list checked); `blueprint.graph.create_node` accepts `nodeType:"K2Node_EnhancedInputAction"` and returns a node id but exposes only the legacy `inputAxisName`, so the node is born with `InputAction == nullptr` and a bool `ActionValue`; `blueprint.graph.set_node_property` answers `[PROPERTY_NOT_SUPPORTED] Unsupported node property 'InputAction'`; UE Python answers `Property 'InputAction' ... is protected and cannot be set`, `unreal.BlueprintEditorLibrary.refresh_all_nodes` does not exist on 5.8, and `ReconstructNode` is not a UFUNCTION so `object.call_function` cannot reach it either. `property.set` against the node's object path DOES apply (`applied:true`) and updates the node title, but leaves the pins stale — `get_node_details` still reported `ActionValue` as `bool` for an `Axis2D` action after a successful `blueprint.compile`, which is the silent-wrong-wiring failure mode. The only fix found is `asset.save {force:true}` + `asset.reload`, after which `get_node_details` reports `ActionValue` as `struct/Vector2D` plus a new `InputAction` output pin, and `blueprint.insert_bpir_at_node` wires the exec pins cleanly with `ActionValue` reachable by bare name. Ten input actions were authored this way. Distinct from `B-bpir-input-event-entry-signatures-unknown` (DONE; that is the *decompiler* emitting `UnknownEntry`, this is the authoring side) and from `E-level-bp-node-verbs-cant-author-bound-nodes` (level-BP stub nodes).
- `#2-workaround-now-prohibited` `OPEN` reporter — The `asset.save` + `asset.reload` step this ticket documents as the only way to reconstruct the node’s pins is **no longer available**. `asset.reload` crashed the shared UE 5.8 editor at 2026-09-02 20:38:13Z (fault inside `ReloadPackages` while a loaded Blueprint still referenced the asset), the third editor loss of the session, and the project has since banned the verb outright (`Docs/fps/PLAN.md` § editor discipline rule 9: "never call `asset.reload`"). That removes the last route from `property.set`-ing `InputAction` to a node with correctly typed pins, so on this host a Blueprint-only project now has **no** supported way to author a bound `K2Node_EnhancedInputAction` at all — `create_node` cannot bind it, `set_node_property` rejects the property, Python refuses it as protected, and the reconstruct step is prohibited. Raising the practical urgency of the two asks above (a BPIR `entry` kind, or an `inputAction` parameter on `blueprint.graph.create_node` applied before pin allocation); a `blueprint.graph.reconstruct_node` verb would also close it without touching package reload. The ten nodes in `/Game/FPS/Player/BP_FPSCharacter` were authored before the ban and are correct on disk; they are not reproducible today.
- `#3-enhanced-input-authoring` `IN-REVIEW` developer — Added BPIR input_action compile/decompile round-trip, bound-before-allocation inputAction create_node support, safe in-memory graph-node reconstruction, corrected whitelist docs, BPIR aspect version 8, and automation coverage with a real Axis2D UInputAction asset.
- `#4-reject-unlabeled-map-body` `IN-REVIEW` developer — Added a pre-mutation compiler hard error for executable statements before the first label in explicit input_action event maps, with source-line guidance and regression coverage proving no partial graph mutation.
- `#5-fix-comment-fixture-order` `IN-REVIEW` developer — Corrected the non-K2 reconstruction rejection fixture to assign its sentinel comment after FGraphNodeCreator finalization, which invokes UEdGraphNode_Comment::PostPlacedNewNode and otherwise replaces the sentinel with the engine default before the handler runs; production already rejected before transaction or mutation.

## Fix

The root cause was that the generic node factory allocated `K2Node_EnhancedInputAction` pins while its protected `InputAction` property was still null. Later reflected writes changed the property and title but did not reconstruct the node, while BPIR had neither an Enhanced Input entry kind nor branch-aware serialization for the node's five event outputs.

Files changed: `Compiler/BpirTypes.h`, `Compiler/BpirParser.cpp`, `Compiler/BpirSharedConstants.h`, `Compiler/CodeNodeEmitter.h`, `Compiler/CodeNodeEmitter.cpp`, `Compiler/BpirCompiler.h`, `Compiler/BpirCompiler.cpp`, `Decompiler/BpirTextEmitter.cpp`, `Decompiler/BpirDecompiler.cpp`, `Handlers/Blueprint/BlueprintGraphCrudHandler.cpp`, `Handlers/Asset/AssetDumpCache.cpp`, `Tests/EnhancedInputTestUtils.h`, `Tests/Bpir/TestBpirEnhancedInputAction.cpp`, `Tests/Blueprint/TestBlueprintGraphEnhancedInputAction.cpp`, `Tests/Utility/TestAssetDumpCache.cpp`, `Docs/wiki-src/bpir.entry-points.md`, and `Docs/wiki-src/blueprint.graph.md`.

Automation coverage added: `PinWright.BPIR.Parser.EnhancedInputAction`, `PinWright.BPIR.RoundTrip.EnhancedInputAction`, `PinWright.blueprint.graph.create_node.EnhancedInputActionBoundBeforePins`, and `PinWright.blueprint.graph.reconstruct_node.EnhancedInputActionRefreshesPins`. The fixtures use a real Axis2D `UInputAction`, verify same-Blueprint Replace/upsert leaves exactly one bound action root, reject mapped executable statements before the first label at their source line without graph mutation, and exercise the production parser, compiler/decompiler, and registered RPC handlers. The reconstruction rejection case preserves the non-K2 node GUID, pins, finalized sentinel comment, and clean package state while returning `INVALID_NODE_TYPE`.

Deliberate non-changes: `InputBlueprintNodes` remains optional and unlinked through dynamic class/property resolution; `set_node_property` remains a fixed presentation-property whitelist; reconstruction never calls `asset.reload`; and the `bpir.txt` aspect value was not incremented beyond 8 because another in-flight change had already set it to 8 before this patch. Build, Unreal automation, MCP, and editor runtime verification were intentionally not run under this ticket's static-check-only constraint.
- `#N-verified-in-fps-build` `IN-REVIEW` reporter — **Implemented; both halves verified.** *Decompile:* `blueprint.decompile {blueprintPath: "/Game/FPS/Player/BP_FPSCharacter", graphName: "EventGraph"}` now emits all ten Enhanced Input handlers as first-class entries, e.g. `entry input_action /Game/FPS/Player/Input/IA_Sprint.IA_Sprint() [Started -> @started, Completed -> @completed]`, with the action's full object path preserved and the exec map emitted whenever more than the `Triggered` chain is connected. *Compile:* tested on a throwaway `BP_ScratchInputProbe` (an ACharacter created for the test and deleted afterwards) rather than the shipped input graph, so a failure could not damage a working deliverable — `blueprint.compile_bpir` with that exact entry text returned `{"nodeCount": 4, "errors": [], "warnings": [], "compiled": true, "status": "UpToDate", "success": true}`. `bpir.entry-points` documents the supported outputs (`Triggered`, `Started`, `Ongoing`, `Canceled`, `Completed`), the required full object path, and that data outputs are typed from the action asset. That removes the four-step workaround this ticket was filed against (`create_node` + `property.set InputAction` + `asset.save` + `asset.reload` + `insert_bpir_at_node`) — which mattered doubly because the `asset.reload` half was later banned outright for crashing the editor, leaving the verb with no workaround at all. Left `IN-REVIEW`.
