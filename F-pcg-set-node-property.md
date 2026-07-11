---
id: F-pcg-set-node-property
title: "pcg: generic per-node settings editor (UpdateNode-style property writes)"
status: OPEN
severity: Low
category: feature
tags: [pcg, authoring, node-settings, low-priority]
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
