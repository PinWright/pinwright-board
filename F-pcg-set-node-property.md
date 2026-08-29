---
id: F-pcg-set-node-property
title: "pcg: generic per-node settings editor (UpdateNode-style property writes)"
status: OPEN
severity: High
category: feature
tags: [pcg, authoring, node-settings, node-configuration, property-write, stub-surface, procedural-vegetation, re-severity]
---

# pcg: generic per-node settings editor

Split from `F-pcg-authoring-parity`. **Low priority** — see "Why low priority" below.

PinWright has typed, node-class-specific settings setters (`pcg.add_slope_filter`,
`pcg.add_noise_filter`, `pcg.set_self_pruning_settings`) but no **generic** way to write an
arbitrary `UPCGSettings` property on any node — the analog of Epic's UE 5.8 PCGToolset
`UpdateNode(Node, JsonParams, NodeTitle)`.

## Why low priority
The gap is partly covered already:
- `property.set` (`Handlers/Utility/UtilityPropertyHandler.cpp`) is a generic reflection setter
  that works on arbitrary `UObject`s by `objectPath`; a PCG node's `UPCGSettings` sub-object has
  a real object path surfaced by `pcg.inspect`, so editing node settings is *reachable* today
  (if unpolished).
- The high-frequency node classes already have first-class typed setters (the three above).

So this is an ergonomic upgrade — a `pcg.set_node_property` that resolves a node by id + writes a
named `UPCGSettings` property (with a `PostEditChange`) — not a missing capability. File is here so
the intent survives the split rather than being lost as prose.

## Proposed scope
- `pcg.set_node_property(graphPath, nodeId, property, value)` — resolve the node's `UPCGSettings`,
  write the reflected property via the shared property-write path, `PostEditChange`,
  `MarkPackageDirty`. Reuse existing reflection helpers (`Utils/PropertyUtils`) rather than a new
  parser.

## Acceptance
An agent can set an arbitrary scalar/enum property on an existing PCG node by name and read the
new value back via `pcg.inspect`.

## History
- `#1-split-from-authoring-parity` `OPEN` reporter — Split off `F-pcg-authoring-parity`. Filed Low priority: `property.set` (generic reflection setter by objectPath) plus the shipped typed helpers (`add_slope_filter`/`add_noise_filter`/`set_self_pruning_settings`) already partially cover per-node settings, so a generic `pcg.set_node_property` is an ergonomic capstone, not a missing capability.
- `#2-re-severity-low-to-high` `OPEN` analyst — **Severity raised Low -> High.** The `#1` rationale is contradicted by current source and is superseded (original text left intact above, per append-only). It rests on "a PCG node's `UPCGSettings` sub-object has a real object path surfaced by `pcg.inspect`, so editing node settings is reachable today". `pcg.inspect` does not surface that path: per node it emits `id` (`Node->GetName()`, `Source/PinWrightPCG/Private/Handlers/PCG/PCGGraphInspect.cpp:47`) and `settingsClass` (`Settings->GetClass()->GetPathName()`, `:50-54`) — a CLASS path, not the settings object's path. A caller wanting `property.set` must therefore hand-construct `<graphPath>:<nodeName>.<settingsSubobjectName>`, and the sub-object name is emitted by no verb and documented on no page. So the stated workaround does not exist as described, which moves this out of the rubric's Medium row ("doable via a documented workaround") into the "hard blocker with no workaround" band. Taking the High end of that band rather than Medium, for reach: `pcg.add_node` resolves ANY `UPCGSettings` subclass by path with `IsChildOf` as its only gate and no module allowlist (`PCGGraphAuthoring.cpp:62`, `:65`, `:67`, `:75`), so the surface lets a caller add every node the engine defines and configure exactly three of them (`add_slope_filter`, `add_noise_filter`, `set_self_pruning_settings`). "Add anything, configure almost nothing" is a stub-shaped surface, and node configuration is required by essentially every non-trivial graph, so no rare-edge-path bump-down applies. Second reason, and the one that prompted the review: `F-pcg-create-graph-class-parameter` would make UE 5.8's Procedural Vegetation Editor reachable, and every PV node is a `UPCGSettings` (`UPVBaseSettings : UPCGSettings`, `Engine/Plugins/Experimental/ProceduralVegetationEditor/Source/ProceduralVegetation/Public/Nodes/PVBaseSettings.h:11-12`) whose entire value is its parameters — a PV graph you can assemble but not configure grows a default species nobody asked for. The three typed setters stay useful and are not what this asks to replace. Source-read only; the editor was not running and no `pcg.inspect` or `property.set` call was made to confirm the reconstructed-path route also fails in practice. Severity field changed; no status change, and no date in this entry per the board README's history rule.
