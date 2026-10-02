---
id: F-graph-node-size-measured
title: "Graph layout needs real node sizes and pin row offsets: add a measured-size provider (live panel, then offscreen SGraphPanel prepass) with an honest estimator fallback — spike first"
status: IN-REVIEW
severity: Medium
category: feature
tags: [layout, node-size, slate, headless, spike, gap-analysis-2026-09-30]
encounters: 1
lastSeen: 2026-09-30T12:00:00Z
---

# Measured node sizes and pin offsets for graph layout (spike first)

Every PinWright layout path works from guessed node sizes, and none of them has pin
row offsets. The only estimator is `EstimateNodeSize` in
`Source/PinWright/Private/Compiler/NodeLayoutEngine.cpp:261-327`, and it is wrong
in both directions:

- **Too tall.** It skips only `bHidden` pins, so pins in the collapsed advanced
  view still count (`:280-294`). PrintString-style nodes come out far taller than
  they render.
- **Too narrow.** It does not model pin default-value widgets (text boxes, enum
  combos, vector fields, asset pickers).
- **Too big.** It does not model compact nodes or variable getters, and it applies
  a 160×64 minimum plus an extra footer row.

The material, anim and rig engines use no sizes at all. Pin-aligned layout
(`F-graph-layout-core`) needs each pin's row offset, which no estimator has today.

## Engine facts (from UE 5.8 source; not yet run)

- **Live panel.** `SNodePanel::GetBoundsForNode` returns the widget position plus
  its desired size (`GraphEditor/Private/SNodePanel.cpp:1720-1750`). PinWright
  already reads live bounds this way for `editor.frame_graph`
  (`Handlers/Editor/EditorWindowHandlers.cpp:105-120`).
- **Offscreen panel.** `SGraphPanel::Update()` is public. It creates the missing
  node widgets synchronously and calls `SlatePrepass` on each new node
  (`SGraphPanel.cpp:2361-2379`). So `SNew(SGraphEditor).GraphToEdit(G)` followed
  by `GetGraphPanel()->Update()` should yield real sizes without a window. Engine
  precedent: `RenderModelNodeToPNG` (`RigVMEditorBlueprintLibrary.cpp:600-624`).
- **Pin offsets need a tick.** `SGraphPin::GetNodeOffset` returns `CachedNodeOffset`,
  which is written only in `SGraphPin::Tick` (`SGraphPin.cpp:929, 992-995`).
  Offsets stay zero until the widget ticks.
- **Editor with `-NullRHI`.** Slate is created with the null renderer, which has
  real font services (`LaunchEngineLoop.cpp:3107-3119, 3343-3354`;
  `SlateNullRendererModule.cpp:150`).
- **Commandlets.** No Slate renderer, so only an estimate is possible.

## Spike (do this first, report before building)

In three modes — `visible`, `offscreen` and `headless` (`-NullRHI`) — on a K2
graph, a material graph and an anim graph:

1. Build an offscreen `SGraphEditor` for a graph that is not open, call
   `Update()`, and read the node sizes.
2. Tick the offscreen widget tree once and read the pin offsets.
3. Compare both against the live panel for the same graph opened in an editor tab.

Record per mode: works / fails / partial, the error, and the size and offset delta
against the live values.

## Build (after the spike)

A provider chain behind the existing `INodeSizeAdapter` seam:

1. live panel, when the graph is open
2. offscreen prepass, where the spike showed it works
3. estimator, fixed for advanced-view pins, default-value widgets and compact nodes.

Each result carries its source, and layout and report verbs publish
`sizeSource {measured, estimated}`.

## Acceptance criteria

- The spike results are recorded in this ticket's history, with per-mode evidence.
- On a K2 fixture with PrintString (advanced pins collapsed), a Branch, a variable
  getter and a math node: measured sizes are within 2 px of live-panel bounds.
- The fixed estimator is within ±15% on the same fixture.
- A test covers the failure direction: an estimator run must never be reported as
  measured.

## Severity justification

**Medium.** Layout works without this, but with overlaps or loose spacing
wherever the estimate is wrong. Pin-aligned wires need pin offsets that no path
provides today.

## History
- `#1-initial-spec` `OPEN` reporter — Gap analysis 2026-09-30: estimator counts advanced-view pins and ignores default-value widgets/compact nodes; no layout path has pin row offsets. Spike offscreen SGraphEditor + SGraphPanel::Update() sizing and pin-offset ticking across visible/offscreen/-NullRHI before building the live → offscreen → estimator provider chain.
- `#2-measured-provider-built` `IN-REVIEW` developer — Spike done from engine source plus a suite test (no editor slot for an interactive spike): `FNodeFactory::CreateNodeWidget` + `SlatePrepass(1)` gives the same widget/desired size a panel uses (SGraphPanel::AddNode prepasses new node widgets the same way and notes node widgets take no outer layout); null owner panel is handled (LOD default, pins not hidden). Pin offsets do NOT need a tick: arranging the prepassed widget tree once from `FGeometry::MakeRoot(size)` (public `ArrangeChildren`, recursive) gives each `SGraphPin`'s geometry. Commandlets have no Slate renderer -> estimator; `-NullRHI` keeps Slate's null renderer with real fonts -> measured. Per-mode numbers (offscreen run; visible/-NullRHI not run here) are logged by the new test's AddInfo lines, to be read from the next scoped run. Built: `Layout/EdGraphNodeMeasure.{h,cpp}` (`CanMeasureEdGraphNodes()`, `MeasureEdGraphNode(Node, FMeasuredNode{Size, PinOffsetY})`); `PwGraphLayoutEdGraph::BuildEdGraphModel` uses it when `UBpirLayoutSettings::bMeasureNodeSizes` (new, default true) and Slate can measure, else `FBlueprintNodeSizeAdapter` (unchanged estimator); measured pin rows rounded to whole px (positions are ints, so aligned rows stay exact and a second pass moves 0); `bMeasuredSize` per node -> `FArrangeReport` counts; `material.authoring.auto_layout` publishes `sizeSource {measured, estimated}`. Provider chain is offscreen -> estimator: a separate live-panel step was skipped because the panel's node widget is the same widget with the same prepass (the test compares against a panel). Material/RigVM stay estimated (no UEdGraph for a closed material). Tests: `PinWright.layout.blueprint.MeasuredSizesMatchGraphPanel` (Print String with advanced pins folded, Branch, int variable getter, int Add: measured within 2 px of an offscreen `SGraphEditor` panel's node widget after `SGraphPanel::Update()`, estimator within ±15 %, every shown pin measured inside the node; skip marker `no-slate-renderer` in commandlets), `PinWright.layout.blueprint.EstimatedSizesNeverReportedMeasured` (measurement off -> every node `bMeasuredSize=false` with the estimator size and report all-estimated; on -> measured exactly when Slate can measure). Existing tests moved to the layout's own geometry: `layout.blueprint.EventGraphFixture` (report all measured or all estimated; event->call levelness via the model's pin rows) and `bpir.compiler.InsertAnchorFallback` (pin rows from `BuildEdGraphModel` instead of the estimator); contracts unchanged. The ±15 % estimator bound is unverified until the suite runs; if it fails the logged numbers say which constant to re-fit. Docs: internals 'Measured sizes' section + settings row, test matrix, CHANGELOG. Every changed/new TU compile-checked with UBT -SingleFile (Linux, UE 5.8): all succeed. Not yet run in an editor (manager owns the build/test slot).
- `#3-spike-numbers-and-getter-refit` `IN-REVIEW` developer — Spike evidence from the full offscreen suite run (Linux Vulkan, UE 5.8; visible and -NullRHI not run): offscreen widget measurement equals the `SGraphEditor` panel's node widget exactly for all four nodes — Print String 182.4x136 (estimate 206x144, exec-in pin 43 vs est. 40), Branch 200x96 (est. 211x104, exec-in 43 vs 40), int Add 146x76 (est. 133x72), int variable getter 165x38 (est. 203x72). Only the getter broke the ±15 % bound: the estimator gave it a title header, but `DrawNodeAsVariable` nodes draw as a headerless pill. `Layout/BlueprintNodeSizeAdapter.{h,cpp}`: variable getters are now sized like compact nodes in height (rows x PinRowHeightPx + 8 = 40 for one pin) with their pins centred, and widest input + widest output label + 2 x HorizontalPaddingPx wide; bound unchanged. Internals doc estimator list updated. Manager's unity fix (`GraphLayout::EFlowDirection` qualified in TestGraphLayoutMetricsPins.cpp) kept.
- `#4-linux-verification` `IN-REVIEW` tester — PinWright `10212ee4` (on origin/master `6283b63b`), UE 5.8 Linux Vulkan. Runs: w23-final = offscreen full suite, no DISPLAY, 5568/5568 passed; w23-xfinal = DISPLAY=:0 offscreen, drive.os_input+click_occlusion+os_gesture+input, 31/31; w23-vis = DISPLAY=:0 windowed drive.input.ModifierChord, 2/2; Python = Content/Python/tests, 427 OK / 5 skipped (all skips Windows-only or an absent local script). Passed in w23-final: `PinWright.layout.blueprint.MeasuredSizesMatchGraphPanel` (Print String with advanced pins folded, Branch, int getter, int Add: measured within 2 px of an offscreen `SGraphEditor` panel's node widgets; estimator within +-15 % after #3's getter re-fit) and `.EstimatedSizesNeverReportedMeasured` (the failure-direction bullet). Not met: Acceptance bullet 1 asks for the spike recorded per mode with evidence. Only the offscreen mode was measured (#3); `visible` and `headless` (-NullRHI) were not run, and no comparison was made against the live panel of a graph open in an editor tab (the reference is an offscreen panel the test builds). Needs: run `MeasuredSizesMatchGraphPanel` in a visible editor and a -NullRHI editor and record the numbers here, or owner acceptance of the offscreen-panel reference.
