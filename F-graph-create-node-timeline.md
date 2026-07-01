---
id: F-graph-create-node-timeline
title: "blueprint.graph.create_node factory has no Timeline branch"
status: DONE
severity: Low
category: feature
tags: [blueprint-graph, mcp-rpc, timeline, node-factory]
---

# blueprint.graph.create_node factory has no Timeline branch

`blueprint.graph.create_node` recognises `nodeType="Timeline"` via the
`GetNodeTypeAliases()` map (`BlueprintGraphHandler.cpp:759`,
`Timeline → K2Node_Timeline`) and falls through to the generic
`NewObject<UEdGraphNode>` path at the end of the handler
(~`BlueprintGraphHandler.cpp:1139-1163`). That path produces a
`UK2Node_Timeline` *instance* but never registers a backing
`UTimelineTemplate` on the owning Blueprint's `Timelines` array — so the
resulting node has no `TimelineName` link to a registered template. Core
timeline pins are allocated by `UK2Node_Timeline`, but the node is not tied
to Blueprint-owned timeline data and compiles to a non-functional timeline.
`F-create-node-collapse-target-param` explicitly listed Timeline
as out-of-scope for the `target`-collapse pass, so the gap is known but
unticketed.

Today the only working path to add a timeline through MCP is BPIR's
`timeline NAME(...)` instruction, which routes through
`FBpirCompiler::EmitTimeline`
(`BpirCompiler.cpp:4577-4605`) → `CodeNodeEmitter::CreateTimelineNode`.
That emitter calls the canonical UE-internal sequence
(`FBlueprintEditorUtils::AddNewTimeline` + assign `UK2Node_Timeline::TimelineName`),
appends the new `UTimelineTemplate` to `Blueprint->Timelines`, and
optionally seeds `FTTFloatTrack` entries before reconstructing the node so
the per-track output pins appear. Imperative callers that only want one
timeline (animation prototypes, tween setup, retainer-driven UI flows)
currently have to drop into BPIR for a single instruction.

**Proposed:** add an explicit `Timeline` / `K2Node_Timeline` branch to the
`create_node` factory, ahead of the dynamic fallback, that:

1. Accepts `target="<TimelineName>"` (preferred — vocabulary parity with
   `F-create-node-collapse-target-param`) or legacy
   `timelineName="<TimelineName>"`; default to a unique
   `Timeline_N` if neither is supplied.
2. Calls `FBlueprintEditorUtils::AddNewTimeline(Blueprint, FName(*Name))`
   to create and register a fresh `UTimelineTemplate` on
   `Blueprint->Timelines`.
3. Spawns a `UK2Node_Timeline` via `FGraphNodeCreator<UK2Node_Timeline>`,
   assigns `TimelineName` before `FGraphNodeCreator::Finalize`, sets the
   requested position, and reports the node id / name / class plus
   timeline name / template path in the success payload.
4. Honours the `x` / `y` mandatory-position rule from
   `E-mandatory-node-position` like every other branch.
5. Leaves track authoring (`addFloatTrack`, `addColorTrack`, curve
   keyframes, `bLoop` / `bAutoPlay`) to a follow-up
   `blueprint.timeline.add_track` RPC — keep this ticket scoped to node
   creation parity, not full timeline authoring.

**Out of scope:** track authoring (float / vector / color / event
tracks), curve keyframe editing, timeline property flags (`bLoop`,
`bAutoPlay`, `bIgnoreTimeDilation`, length, `ReplicationType`). BPIR
already covers float-curve seeding for the multi-track scripted case; the
imperative parity ticket here is only about getting a wired-up empty
timeline node into a graph in one call.

**Files affected:**
- `Plugins\EditorAutomationRpcGateway\Source\EditorAutomationRpcGateway\Private\Handlers\Blueprint\BlueprintGraphHandler.cpp`
  (`create_node` factory, new explicit Timeline branch before the dynamic
  fallback ~line 1139)
- `Plugins\EditorAutomationRpcGateway\Source\EditorAutomationRpcGateway\Private\Tests\Blueprint\TestBlueprintHandlers.cpp`
  (regression: create timeline node, assert `Blueprint->Timelines.Num()`
  grew by one, assert `Play` / `Stop` / `Update` pins exist)
- `Plugins\EditorAutomationRpcGateway\docs\wiki\blueprint.graph.md`
  (`create_node` Timeline example + payload schema)

## History
- `#1-initial-feature-request` `OPEN` reporter — `blueprint.graph.create_node`
  resolves `nodeType="Timeline"` to `K2Node_Timeline` via the alias map but
  the dynamic `NewObject<UEdGraphNode>` fallback produces a node with no
  backing `UTimelineTemplate` and no `Play`/`Stop`/`Update`/`Reverse`/`Finished`
  pins. `F-create-node-collapse-target-param` explicitly carved Timeline
  out as a separate ticket; today BPIR's `timeline NAME(...)` instruction
  (`BpirCompiler.cpp:4577`) is the only working path. Proposal: add an
  explicit Timeline branch to the factory that reuses the BPIR emitter's
  template-registration path (`AddNewTimeline` + assign `TimelineName`
  before node finalization), accepts `target` / `timelineName`, and honours
  `E-mandatory-node-position`. Track authoring stays out of scope —
  follow-up `blueprint.timeline.add_track` RPC.
- `#2-analysis-corrected-fix` `OPEN` developer — Source review confirmed the Timeline branch is needed, but UE 5.6 has no SetTimelineTemplate API and core timeline pins are allocated even without a template. Implement by registering a UTimelineTemplate with AddNewTimeline, assigning TimelineName before node finalization, and testing Blueprint->Timelines plus pins.
- `#3-register-timeline-template` `IN-REVIEW` developer — Added an explicit blueprint.graph.create_node Timeline branch that registers a UTimelineTemplate, assigns TimelineName before node finalization, returns template metadata, and covers template registration plus pins in TestBlueprintHandlers.cpp.
- `#4-verify-timeline-node` `DONE` tester — Verified: created `/Game/App/UI/Test/W_McpVerifyTemp_F_graph_create_node_timeline`, ran `blueprint.graph.create_node` with `nodeType="Timeline"` and `target="McpVerifyTimeline"`, observed `nodeClass: K2Node_Timeline`, `timelineName: McpVerifyTimeline`, non-empty `timelineTemplatePath`, and `blueprint.compile` returned `compiled: true` with no errors or warnings; deleted the temp asset afterward.
