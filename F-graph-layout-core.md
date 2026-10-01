---
id: F-graph-layout-core
title: "PinWright's own engine-agnostic layered graph formatter (PwGraphLayout) with per-graph-type adapters, replacing the BA-derived BPIR engine and the fixed-grid MGIR/AGIR/CRIR engines"
status: DONE
severity: High
category: feature
tags: [layout, blueprint, material, anim, controlrig, bpir, mgir, gap-analysis-2026-09-30]
encounters: 1
lastSeen: 2026-09-30T12:00:00Z
---

# PinWright's own engine-agnostic layered graph formatter

Build one layout core and a thin adapter per graph type. It replaces every layout
engine PinWright ships today:

- **Blueprint (BPIR compile).** The current engine is BA-derived and must be
  replaced (`B-layout-engine-ba-derived`).
- **Material, anim and Control Rig.** `MGIRLayoutEngine.cpp`, `AGIRLayoutEngine.cpp`
  and `CRIRLayoutEngine.cpp` use a fixed depth × lane grid (320×180). They ignore
  node sizes and order each column by GUID/path, so nodes overlap and wires cross
  arbitrarily.

**Copying policy (user decision, 2026-09-30).** Reading Blueprint Assist source,
and reading PinWright's current layout code, is allowed. What is not allowed is
direct copying without change: no verbatim or near-verbatim code, identifiers or
structure. Use your own names and decomposition. The current engine's function and
type names and its pass structure must not carry over. This ticket and the tickets
it links are the specification.

## Model

The core works on an abstract graph, so it is testable without assets:

- **Nodes:** `{id, size, fixed?, pins[]}`. Each pin has a side (in/out), a row
  offset from the node top, and a kind (control-flow/exec or data).
- **Edges:** pin to pin.

Adapters build the model and write positions back:

| Adapter | Root / flow | Position write |
|---|---|---|
| K2 `UEdGraph` (event, function, macro graphs) | roots = events / function entry / tunnels; flow left→right | `Schema->SetNodePosition` |
| Material / material function expressions | root = the material output node at `Material->EditorX/Y`; graph grows leftwards from the root | `Expression->Modify()` + editor X/Y |
| Anim graph `UEdGraph` | root = output pose / result node; grows leftwards | `Schema->SetNodePosition` |
| RigVM | via `URigVMController::SetNodePosition` | with undo |

Niagara comes later, only when a caller needs it.

## Algorithm (generic layered layout, exec-spine first)

1. **Scope and components.** Split the scope into one tree per root. Every root is
   laid out; there is no single-anchor limit (`B-bpir-layout-single-anchor-per-graph`).
2. **Cycle breaking.** Edges found as DFS back edges are ignored for layering.
3. **Control-flow X.** Longest-path layering using real node widths: a child's X is
   the maximum over its predecessors of (predecessor right edge + horizontal gap).
   Re-converging paths land to the right of their right-most predecessor.
4. **Data (pure) node X.** Data-only nodes cascade leftwards (upstream) from their
   consumer, one column per dependency level, so no data wire runs backwards
   (`B-bpir-layout-pure-chain-single-column`). A data node feeding several
   consumers is placed once, with the first consumer in traversal order.
5. **Ordering.**
   - Control-flow branches follow the source pin order.
   - Data DAGs (materials, anim) get 2–4 barycenter sweeps to reduce crossings.
   - Every tie breaks on pin index, then node GUID, never on `Graph->Nodes`
     order, so the output is deterministic.
6. **Y, pin-aligned.**
   - The first control-flow edge out of a node, and each data edge where possible,
     is made horizontal: target Y = source pin Y − target pin row offset.
   - Sibling subtrees are packed by comparing their outlines (contours), so they
     cannot overlap by construction. No iterative push-down loop is needed.
7. **Packing trees.** Trees are stacked in their original root order with a fixed
   gap. The first root keeps its current position. Nodes outside the scope are
   fixed obstacles.
8. **Grid.** Snap final positions to the editor grid (16).
9. **Extensions.** Comment re-fit (`F-graph-layout-comment-fit`) and optional
   reroute insertion (`F-graph-layout-reroute-tracks`).

## Sizes

Sizes come through the existing `INodeSizeAdapter` seam
(`Source/PinWright/Private/Layout/GraphLayoutMetrics.h`):

- An estimator is used until `F-graph-node-size-measured` lands; measured sizes
  then plug in behind the same seam.
- The core never guesses silently. It reports how many sizes were measured and how
  many were estimated.

## Callers

- **BPIR compile/insert.** Replaces `RunLayoutPass`'s engine. Scope = created nodes;
  pre-existing nodes are obstacles; each root keeps its position.
- **MGIR / AGIR / CRIR compile.** Replaces their grid engines whenever positions
  were not authored. Authored `@(x,y)` positions stay fixed.
- **Standalone verbs.** `blueprint.graph.auto_layout` (`F-blueprint-auto-layout-rpc`),
  `material.authoring.auto_layout`, and the anim / controlrig auto-layout tickets.
  All share one contract (see `F-blueprint-auto-layout-rpc`).

## Acceptance criteria

**Synthetic fixtures** (no host content):
- linear chain; Branch; Sequence
- diamond re-convergence; exec loop
- 3 events in one graph
- data node shared by 2 consumers; 3-deep data chain
- material with a texture sample and a multi-input math chain
- anim pose chain.

**For every fixture:**
- 0 node-rect overlaps.
- 0 backward edges, cycle back edges excepted.
- Every same-row edge has |source pin Y − target pin Y| ≤ 1 px.
- Deterministic: shuffling `Graph->Nodes` gives identical positions.
- Idempotent: a second run moves 0 nodes.
- Undo restores every original position.
- Failure direction: a deliberately overlapping or backward fixture must score
  worse in `GraphLayoutMetrics`.

**Clean-up:**
- The BA-derived files are deleted and the old grid engines removed.
- `B-layout-engine-ba-derived` acceptance is satisfied.
- A reviewer confirms the new formatter contains no verbatim or near-verbatim code,
  identifiers or structure from Blueprint Assist or from the replaced NodeLayoutEngine.

**Vision check** (rpc-design §16): the fixtures are captured with
`editor.frame_graph` + `editor.screenshot_window` in an offscreen run and looked at.

## Supersedes

- `F-material-layout-bounds-aware`
- `F-anim-layout-bounds-aware`
- `F-controlrig-layout-bounds-aware`

## Severity justification

**High.** Every graph PinWright authors goes through a layout engine. Today those
engines produce overlaps and backward wires, and the Blueprint one carries a
licensing exposure. The reach is every authoring session.

## History
- `#1-initial-spec` `OPEN` reporter — Gap analysis 2026-09-30: specify PinWright's own layered formatter (cycle break, longest-path X with real widths, leftward data-node cascade, barycenter ordering for data DAGs, pin-aligned Y with contour packing, per-root trees, deterministic ties) with K2 / material / anim / RigVM adapters, replacing the BA-derived BPIR engine (B-layout-engine-ba-derived) and the fixed 320x180 grid engines; implementers must not have read Blueprint Assist source.
- `#2-copying-policy-correction` `OPEN` reporter — Correction to #1: reading Blueprint Assist source and PinWright's current layout code is allowed; the only rule is no direct copying without change (no verbatim or near-verbatim code, identifiers or structure).
- `#3-layered-formatter-implemented` `IN-REVIEW` developer — New core `Source/PinWright/Private/Layout/PwGraphLayout.{h,cpp}` (abstract nodes with key/size/movable/pin rows of side+kind, wires output pin -> input pin, explicit roots): (1) one tree per root, roots = explicit, then fixed nodes driving movable flow nodes, then movable flow sources, then cycle-only leftovers; trees grow depth-first along flow outputs in pin order and DFS back edges are ignored for layering; (2) data-only nodes are claimed once by the first consumer in traversal order (tree, flow depth, DFS preorder), leftovers by the fixed node they feed, else they root their own tree (sinks first); a consumer's block is one right-aligned column per longest-path dependency level, ordered by three barycenter sweeps (pin-fraction weighted); (3) longest-path X with real widths plus room for the node's own data block; (4) Y: pin-aligned target, then the first free offset against everything placed and every fixed node, found by one sweep over the blocked intervals sorted by their low end (no push-down loop; overlap-free by construction); preorder so later siblings settle below; (5) X snapped to the grid, Y snapped wherever the node is not pin-aligned (so aligned rows stay exact; a 22 px K2 row height is not a multiple of the 8/16 grid). Ties break on pin index then key; the report counts measured vs estimated sizes (all estimated until F-graph-node-size-measured). Adapters: `Layout/PwGraphLayoutEdGraph.{h,cpp}` (K2 / anim / state machine, exec category = flow, writes through `Modify()`; `ArrangeAnimBlueprint` moves (0,0) nodes in every anim graph AGIR authors), `Layout/PwGraphLayoutMaterial.{h,cpp}` (fixed virtual output root at `Material->EditorX/Y` with one row per connected property, named-reroute usage->declaration edge, only (0,0) expressions move, `Modify()`), `Layout/PwGraphLayoutRigVM.{h,cpp}` (execute-context pins = flow, IO pins on both sides). Callers replaced: `RunLayoutPass` in `Compiler/BpirCompiler.cpp` (every created event / function entry / entry tunnel is a root, insertion/body anchor first; upstream-walk fallback kept), `MGIR/MGIRCompiler.cpp`, `material.authoring.auto_layout` (`Handlers/Material/MaterialAuthoringHandler.cpp`, summary + README row reworded), `AGIR/AGIRCompiler.cpp`, `CRIR/CRIRCompiler.cpp`. Deleted: `MGIR/MGIRLayoutEngine.*`, `AGIR/AGIRLayoutEngine.*`, `CRIR/CRIRLayoutEngine.*`, `Compiler/NodeLayoutEngine.*`, `Compiler/NodeLayoutParameterFormatter.*`, `Tests/Bpir/TestNodeLayout.cpp` (its 25 tests exercised the deleted engine's internals). Size estimator rewritten in `Layout/BlueprintNodeSizeAdapter.{h,cpp}` (+`PinOffsetY`, `IsPinShown`; folded advanced pins no longer counted). `UBpirLayoutSettings` spacing knobs renamed `ColumnGapPx`/`RowGapPx`/`DataColumnGapPx`/`GridSnapPx`; `IntraParameterPadY`/`CollisionIterationCap`/`TraversalIterationCap` removed (`TestBpirInterfaceOutputOverride.cpp` and `TestGraphLayoutMetrics.cpp` updated for the rename/estimator, no coordinate assertions changed). Tests (all new): `PinWright.layout.core.{LinearChain,Branch,Sequence,DiamondReconvergence,ExecLoop,ThreeEvents,SharedDataNode,DataChainThreeDeep,MaterialMathChain,AnimPoseChain,FixedObstacleAvoided,MetricsScoreBrokenLayoutsWorse}`, `PinWright.layout.blueprint.{EventGraphFixture,CompilePassArrangesEveryEntry}`, `PinWright.layout.material.GrowsLeftFromOutput`, `PinWright.layout.anim.PoseChainGrowsLeftFromResult`, `PinWright.layout.controlrig.DataChainFlowsRight`; filter `PinWright.layout` (also covers the existing `layout.metrics.*`). Every core fixture asserts 0 overlaps, 0 backward wires (cycle back edge excused), identical positions after shuffling node order, and a second pass moving 0; plus per-fixture alignment / column / crossing-not-worse checks; K2 and material adapter tests also assert undo restores every position. The 12 core tests were also run outside the editor against a minimal UE-type shim (all pass; three seeded mutations — no first-fit, no block reach, no barycenter sweeps — each turn tests red). Compile-checked with UBT -SingleFile. Docs: `docs/bpir-compiler-internals.md` layout section rewritten, `docs/bpir-test-matrix.md` §2.6, `docs/test-organization.md`, `docs/lessons.md`, wiki-src `material.mgir.md` / `controlrig.md` runLayout lines, CHANGELOG. Not done in this pass: the vision check (needs an editor: `editor.frame_graph` + `editor.screenshot_window` offscreen), comment re-fit and reroute insertion (separate tickets), measured sizes, undo for RigVM (positions go through `SetNodePosition` without its undo bracket, same as the old CRIR engine, because the CRIR compile runs unbracketed), anim/rig undo tests. Known limit: a data node shared by consumers in different branches sits with the shallowest-by-flow-depth consumer, so a wire to another branch's consumer can run backwards. Clean room: written from standard layered-drawing techniques; Blueprint Assist source was not consulted; the deleted engine was read only for the caller contract. None of its identifiers (FormatX/FormatY, FFormatXInfo, parameter formatter, cluster-bounds registry, GetChildX, same-row marking, anchor reset, directional snap) or its pass structure (two X passes around a parameter-column pass, iterative Y collision loop, anchor translation) carries over — reviewer to confirm.
- `#4-round1-suite-fixes` `IN-REVIEW` developer — Round-1 suite found two defects, both fixed. (1) Core (`Layout/PwGraphLayout.cpp`, `AssignY`): fixed nodes were registered as obstacles at X=0 (their X was never copied), so every obstacle blocked the origin column; a movable root at (0,0) was pushed down on a repass (`layout.anim.PoseChainGrowsLeftFromResult` second pass moved 1) and roots near X=0 were pushed by unrelated nodes. (2) K2 adapter: an event's delegate output (`UK2Node_Event::DelegateOutputName`) is drawn in the title bar, but was counted as output row 0, so the event's exec output sat one row (22 px) low and the first call landed off the event's row (`bpir.compiler.InsertAnchorFallback`: 614 vs 592). `FBlueprintNodeSizeAdapter::IsPinInTitle` / `TitlePinOffsetY` now keep that pin out of the rows (size and pin offsets), used by `PwGraphLayoutEdGraph`. The InsertAnchorFallback expectation is a legitimate contract (event and its first call share a row) and is kept. New/extended tests: `PinWright.layout.core.OriginOnlyRepassMovesNothing` (new), `PinWright.layout.core.FixedObstacleAvoided` (root keeps its position), `PinWright.layout.blueprint.EventGraphFixture` (event and first call share NodePosY), `PinWright.layout.metrics.BlueprintNodeSizeAdapter` (delegate pin is a title pin); the two core assertions fail with fix (1) reverted in the offline harness.
- `#5-stacked-spine-fix` `IN-REVIEW` developer — Round-2 suite regression `PinWright.layout.blueprint.CompilePassArrangesEveryEntry` (second entry's body not horizontal). Root cause in the core, exposed by `#4`: a later tree's root was first-fitted on its own rect only. Once the title-bar fix made events (one row) shorter than their bodies (two rows), the second event fitted just under the first event while its body, pin-aligned beside it, hit the first body and was pushed off the row (in round 1 the extra delegate row had hidden it). Fix (`Layout/PwGraphLayout.cpp`, `FirstFreeY`): a node is placed together with its exec spine (chain of first flow children with their pin-aligned offsets), so it only lands where the whole straight chain fits; the first tree's root still moves only if it overlaps a fixed node itself (keeps its position otherwise). New test `PinWright.layout.core.StackedTreesKeepSpinesStraight` (three short events over taller calls) fails before the fix and passes after in the offline harness; all 14 core tests pass there, and a K2-shaped simulation of the failing test puts both bodies on their events' rows.
- `#6-clean-room-review` `IN-REVIEW` reviewer — **Verdict: CONFIRMED clean.** `PwGraphLayout*`, the rewritten `BlueprintNodeSizeAdapter` and `BpirLayoutSettings` contain no verbatim or near-verbatim code, identifiers or pass structure from the deleted NodeLayoutEngine / NodeLayoutParameterFormatter, which is the proxy for Blueprint Assist (its source was not available and was not fetched). **No must-fix items.** Method and numbers are in B-layout-engine-ba-derived `#3-clean-room-review`: longest shared raw run 16 tokens in non-test code (a UE API idiom), core `PwGraphLayout.cpp` 8 tokens / 0.3 % coverage, and normalized similarity at the level of unrelated control files. Also checked against the deleted MGIR/AGIR/CRIR grid engines, which are PinWright's own code, not BA: longest raw run 20 tokens (`for (UEdGraph* Graph : AnimBP->FunctionGraphs)`, the anim-graph collection), with no structural carry-over. **Shared ideas, recorded so the `#3` claims are accurate. None of them is copying.** (a) `#3` lists "directional snap" as not carried over. The idea does survive: data columns left of a consumer snap down (`Layout/PwGraphLayout.cpp:604`) and flow nodes right of their predecessor snap up (`:661`). This is the same floor-left / ceil-right rule as the old `GetChildX`/`SnapToGrid`. Any snap that keeps a minimum gap is forced into it. The new code applies it inline at placement, with no separate pass keyed on link direction. (b) The flow X is `max over flow predecessors (right + ColumnGap) + BlockReach` (`:658-661`). Arithmetically this is the old `GetChildX` output-branch result, `ParentRight + (ChildLeft − ClusterLeft) + NodePadX`, which the pre-cleanup code had labelled a BA port. Here it is the geometric consequence of this ticket's spec (§3 longest-path X plus §4 data blocks to the left). The expression differs: it takes the max over all predecessors, has no bounds-rect delta and no input-direction branch. (c) The first-consumer claim of data nodes by a BFS over input pins, and keeping the first flow child on the parent's row, are both required by §4 and §6 of this ticket. They are standard layered-layout ideas. **Shared facts and values in the estimator. Not BA-attributed; noted, not blocking.** The `BlueprintNodeSizeAdapter.cpp:20-26` constants (min 160×64, 40 px gutter, 8 / 7 px per character), the font style names, the `ListView` title and the display-name→`PinName` fallback are all the same as the old `EstimateNodeSize`. Its body is rewritten: the helpers are merged, width uses the widest input plus the widest output instead of per-row pairs, height is `rows + 0.5` instead of `rows + 1` plus a large-header bias, and title-bar and advanced pins are handled. The `UBpirLayoutSettings` defaults are unchanged at 80/48/32/8; these are parametric values, not expression. **Optional cleanup.** `Tests/Bpir/TestCompilerIntegration.cpp:3778-3785` and `:3819-3821` still describe the deleted engine's sibling stacking ("lowest-placed-sibling", "non-same-row siblings"). It is not a BA reference, but it now describes a mechanism that no longer exists. **Out of scope, user decision.** The published history (`8748c637`, the 0.8.0 MIT release, reachable from `origin/master` per local refs) still contains the BA-derived engine and its BA citations. Deleting the files at HEAD does not remove that history.
- `#7-vision-check` `IN-REVIEW` developer — Vision check (rpc-design §16) in an offscreen UE 5.8 editor on Linux. I compiled 11 scratch fixtures with layout on, then framed and screenshotted each one and looked at the images: BPIR linear, Branch, Sequence, diamond, exec loop, three events, shared data node, 3-deep data chain; MGIR texture sample + math chain; AGIR additive + slot pose chain; CRIR BeginExecution -> SetTransform -> 2x ParentConstraint with GetTransform. PNGs are in the session scratchpad `layout-vision/`. What held: wires run left to right, roots stay put, trees stack without overlap, a re-convergence lands right of its right-most predecessor, the material graph grows leftwards from the output node in right-aligned columns with the main chain straight, and the anim chain grows leftwards in pin order. All the defects were in node size / pin-row estimation; the core did not cause any of them. (1) Subtitled K2 nodes ("Custom Event", "Target is X", anim "Group ..."): a deeper header the estimator ignored, so event -> first call ran 16 px off-level. (2) The K2 row pitch / header defaults (22 / 44) did not match the editor (~32 / 24), so data wires sat 10 px per row off-level. (3) Default-value boxes were not counted in width: a Print String with a long literal overlapped the Delay beside it. (4) Compact nodes (+, x, conversions) were sized as headered nodes, so the exec wire ran straight through them. (5) The "Development Only" bar and the advanced expander were not counted in height. (6) The material preview thumbnail was not counted: a Texture Sample touched the node under it. (7) RigVM: widths were far too small (value editors) and expanded sub-pins took no rows, so GetTransform overlapped SetTransform, the ParentConstraints abutted, and the Transform -> Value wire was misaligned. Fixed in `Layout/BlueprintNodeSizeAdapter.{h,cpp}`: header + 16 px per extra title line; value boxes in width; compact nodes headerless with centred pins; footer bars; `PinOffsetY(const UEdGraphPin*)` replaces the row-index API. The K2 defaults are re-measured as `HeaderHeightPx` 24 / `PinRowHeightPx` 32 (`BpirLayoutSettings.cpp`). `Layout/PwGraphLayoutEdGraph.cpp` now uses the per-pin offsets. `Layout/PwGraphLayoutMaterial.cpp`: 24 / 28 px rows and +112 px for an open preview. `Layout/PwGraphLayoutRigVM.{h,cpp}`: 16 / 24 px rows, expanded sub-pin rows (a link to a collapsed sub-pin lands on its nearest shown parent), at least 260 px width. Tests: `PinWright.layout.metrics.BlueprintNodeSizeAdapter` (subtitle header, long-literal width, compact centring) and `PinWright.layout.blueprint.EventGraphFixture` now check that the event's exec output and the call's exec input are level on screen. `PinWright.bpir.compiler.InsertAnchorFallback` now does the same; it used to assert equal NodePosY, which is exactly the 16 px off-level the screenshots showed. The contract (event and its first call on one row) is unchanged, now measured at the pins. Undo: `PinWright.layout.anim.PoseChainGrowsLeftFromResult` now also checks that undo restores every pose node (`ArrangeAnimBlueprint` writes through `Modify()`). Control Rig undo is not supported on the CRIR compile path: the compile creates its nodes through the controller without undo, so a position-only undo entry would leave the nodes and is not offered; a standalone controlrig auto_layout verb would bracket it. All changed TUs compile with -SingleFile. These fixes are not yet re-checked visually (no build/editor slot). Other: the scratch-folder cleanup with `asset.delete {force:true}` crashed the editor (filed `B-asset-delete-force-folder-gc-in-frame-end-pump-crash`). The remaining scratch `.uasset`s were removed by hand, and the whitespace-only Config/*.ini churn was restored.
- `#8-fixture-enabled-state` `IN-REVIEW` developer — Round-4 regression `PinWright.layout.blueprint.EventGraphFixture` ("a reordered Graph->Nodes gives identical positions"). Root cause was not node order: the test fixture. `CompilerTestUtils::SpawnPrintStringCall` runs `PostPlacedNewNode` before the function reference is set, so the Print String nodes were never given the Development Only state the editor gives them on placement. The test's `UndoTransaction` deserializes the moved nodes, and `UK2Node_CallFunction::Serialize` fixes the state up to DevelopmentOnly. So the arrange after the undo saw Print String nodes 20 px taller (the estimator's new Development Only footer, which matches what the editor draws), and placed them differently. Fix (test only, assertion unchanged): `Tests/Layout/TestPwGraphLayoutAdapters.cpp` re-runs `PostPlacedNewNode` on the call nodes once their functions are set, mirroring the editor's spawn order, and asserts Print String is DevelopmentOnly before layout. The estimator and core are unchanged; the core's order-independence is still covered by `layout.core.*` (shuffled-order check on every fixture). This can't be reproduced in the offline harness (engine serialization); compile-checked with -SingleFile.
- `#9-vision-recheck` `IN-REVIEW` developer — Re-screenshotted the 11 fixtures after the round-4 rebuild, in an offscreen editor (PNGs `layout-vision/lv2_*.png` beside the first set). Confirmed fixed: an event's exec output is level with its first call's exec input; the long-literal Print String no longer overlaps the Delay; a pure node's Return Value is level with the Branch Condition and the conversion node with In String; the Texture Sample has clearance from the node below; the anim Slot (subtitled) is level with Apply Additive; and Control Rig nodes no longer overlap. Remaining defects found and fixed in code: (a) a data block straddled its consumer's incoming exec wire (the event -> Print exec line ran through the +, x compact nodes, through Random Integer, and through Get Transform in the rig). The core (`Layout/PwGraphLayout.cpp`, `ShapeBlock`) now keeps every block node of a flow owner below that wire (flow-input row + RowGap/4). A data wire bends where this clearance overrides pin alignment ("where possible" in the spec). (b) RigVM rows did not follow the editor's order (execute pins, outputs, other IO, inputs, per `URigVMEdGraphNode::AllocateDefaultPins`), so Get Transform's output was modelled below its inputs. `Layout/PwGraphLayoutRigVM.{h,cpp}` now orders rows that way. Tests: `layout.core.SharedDataNode` and `layout.core.DataChainThreeDeep` now assert that the data nodes stay below the incoming exec wire; they replace the consumer-edge horizontal checks this rule overrides, and both fail in the offline harness with (a) reverted. The in-chain wires are still asserted horizontal. `layout.blueprint.EventGraphFixture` asserts the pure chain stays below the exec wire (Sum -> ToText is no longer required horizontal; ToText -> Print still is). `layout.controlrig.DataChainFlowsRight` asserts the Result row sits above the A/B input rows. All 14 core tests pass in the harness, and the changed TUs compile with -SingleFile. Known and not fixed: the anim Output Pose node draws a pose image, so its Result pin sits ~50 px lower than estimated (the Slot -> Result wire bends); re-convergent exec wires (fall-through labels) run behind the nodes in between (reroute insertion is F-graph-layout-reroute-tracks). Cleanup: `editor.quit {discard:true}` on my editor (pid 3113328), scratch `.uasset`s removed from disk, whitespace-only Config/*.ini churn restored; no force delete.
- `#10-final-vision-check` `IN-REVIEW` developer — After the round-6 build, re-screenshotted only the fixtures `#9` touched (`layout-vision/lv3_*.png`): shared data node, 3-deep data chain, a Branch whose merge consumes Random Integer -> + -> ToString, and the Control Rig chain. Confirmed: the event -> Print exec wire now runs above the Random Integer, +, x and conversion nodes, with none straddled. The Branch "yes" -> merge wire clears its data block. Get Transform sits below the Forwards Solve exec wire, and its Transform output is level with Set Transform's Value input (rows in editor order). No overlaps, no backward wires. No code change this round. Follow-up notes, not defects of the `#9` change; both belong to F-graph-layout-reroute-tracks: (1) in a re-convergence, the second predecessor's exec wire (Branch "no" -> merge) curves up through the merge's data block (here the + node), because a block can only clear the wire that is level with its owner; (2) the data wire from a shared node to its later consumer (ToString -> second Print, across a Delay) runs behind the intermediate exec node. Both need reroute or track insertion rather than placement. Cleanup: `editor.quit {discard:true}` on my editor (pid 3236658), scratch `.uasset` removed from disk, whitespace-only Config/*.ini churn restored.
- `#11-verified-done` `DONE` tester — All 24 `PinWright.layout.*` tests plus `PinWright.bpir.compiler.InsertAnchorFallback` pass in the round-6 scoped run `636bc41c` (offscreen, Linux Vulkan, UE 5.8; 1183/1183, every skip a host `fixture-missing` marker outside layout) at PinWright `9bb70b90`; the core and adapter tests also passed in the final full suite `dc867ac2` before the estimator round. Acceptance: every synthetic fixture asserts 0 overlaps, no backward edges outside cycles, pin-level same-row edges, shuffle determinism and idempotence; failure direction via `MetricsScoreBrokenLayoutsWorse`; undo restores positions for Blueprint, material and anim (`PoseChainGrowsLeftFromResult`), not for Control Rig, whose compile path creates nodes with undo off (documented); BA-derived and grid engines deleted; reviewer confirmed clean (`#6-clean-room-review`); vision check done three times (`#7`, `#9`, `#10`), the last showing no overlap, backward wire or wire through a node in the re-shot fixtures. Reroute-only cases moved to `F-graph-layout-reroute-tracks`.
