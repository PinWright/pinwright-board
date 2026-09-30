---
id: F-graph-layout-core
title: "PinWright's own engine-agnostic layered graph formatter (PwGraphLayout) with per-graph-type adapters, replacing the BA-derived BPIR engine and the fixed-grid MGIR/AGIR/CRIR engines"
status: OPEN
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
