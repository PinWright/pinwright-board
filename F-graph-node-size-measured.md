---
id: F-graph-node-size-measured
title: "Graph layout needs real node sizes and pin row offsets: add a measured-size provider (live panel, then offscreen SGraphPanel prepass) with an honest estimator fallback — spike first"
status: OPEN
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
