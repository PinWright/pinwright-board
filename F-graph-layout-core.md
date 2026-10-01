---
id: F-graph-layout-core
title: "PinWright's own engine-agnostic layered graph formatter (PwGraphLayout) with per-graph-type adapters, replacing the BA-derived BPIR engine and the fixed-grid MGIR/AGIR/CRIR engines"
status: IN-REVIEW
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
