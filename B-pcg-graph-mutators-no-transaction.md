---
id: B-pcg-graph-mutators-no-transaction
title: "Eight of the ten mutating pcg verbs open no FScopedTransaction and call no Modify() — pcg.add_node and pcg.remove_node included — so editor.undo cannot reverse a node added, a node deleted with all its property configuration, or a graph parameter removed"
status: OPEN
severity: Medium
category: bug
tags: [pcg, add-node, remove-node, undo, transaction, mutator-no-undo-transaction, graph-authoring]
encounters: 1
lastSeen: 2026-08-29T22:00:00+03:00
---

# The `pcg` namespace mutates graphs outside the transaction system

`pcg.add_node` calls `UPCGGraph::AddNodeOfType` and `pcg.remove_node` calls
`UPCGGraph::RemoveNode` (`Source/PinWrightPCG/Private/Handlers/PCG/PCGGraphAuthoring.cpp`,
registrations at `:65` and `:280`). Neither opens an `FScopedTransaction`, and neither calls
`Modify()` on the graph or the node. Without an active transaction there is no undo record,
so `editor.undo` cannot reverse either edit.

`remove_node` is the one that costs something irrecoverable: a PCG node carries its settings
object, and deleting the wrong node throws away every property set on it. Re-adding the node
does not restore them — the caller has to remember and re-enter the configuration.
`add_node` leaves litter the caller must locate and delete by hand.

## Namespace census

Ten mutating verbs across the module; two have a transaction, eight do not.

| verb | file | transaction |
|---|---|---|
| `pcg.connect_pins` | `PCGGraphAuthoring.cpp:131` | **yes** (`:202`) |
| `pcg.set_node_property` | `PCGSetNodeProperty.cpp:173` | **yes** (`:277`) |
| `pcg.generate` | `PCGGenerateHandler.cpp:109` | **yes** (`:156`) |
| `pcg.add_node` | `PCGGraphAuthoring.cpp:65` | no |
| `pcg.remove_node` | `PCGGraphAuthoring.cpp:280` | no |
| `pcg.add_noise_filter` | `PCGAddNoiseFilter.cpp:45` | no |
| `pcg.add_slope_filter` | `PCGAddSlopeFilter.cpp:62` | no |
| `pcg.add_subgraph` | `PCGAddSubgraph.cpp:17` | no |
| `pcg.add_graph_parameter` | `PCGGraphParameters.cpp:246` | no |
| `pcg.remove_graph_parameter` | `PCGGraphParameters.cpp:380` | no |
| `pcg.set_self_pruning_settings` | `PCGSetSelfPruningSettings.cpp:41` | no |

(`pcg.generate` is listed for completeness — its transaction covers a generate, not a graph
edit.) Of the eight unguarded verbs, `PCGAddNoiseFilter`, `PCGAddSlopeFilter`,
`PCGAddSubgraph`, `PCGGraphParameters` and `PCGSetSelfPruningSettings` contain **zero**
`Modify()` calls of any kind, so they are not merely missing the transaction — they would
record nothing even inside one. `PCGGraphAuthoring.cpp` has two `Modify()` calls, both inside
the `connect_pins` transaction added by `B-pcg-connect-pins-silently-replaces-edge` `#2`.

## Filed against a ticket that declined to file it

`B-pcg-connect-pins-silently-replaces-edge` (IN-REVIEW, High) established the property and
routed it here:

> `PCGGraphAuthoring.cpp` wraps **no** mutation in an `FScopedTransaction` — not `add_node`,
> not [...] That is a namespace-wide gap deserving its own ticket rather than a [...]

That ticket then added a transaction to `connect_pins` only. This is the residue, widened past
`PCGGraphAuthoring.cpp` to the whole module by a fresh census.

Direct sibling: `B-foliage-mutators-no-transaction` (OPEN, Medium) — the identical property in
a different handler family, and the reason this is rated Medium rather than argued from
scratch. `B-paint-layer-destroys-other-layer-weights` carries a third instance. Three
independent handler families, found independently, all mutating outside the transaction
system; the plugin has no structural guard that would have caught any of them.

## Severity

**Medium**, matching `B-foliage-mutators-no-transaction`. Impact class is a soft blocker: the
work is doable, and the workaround (re-author the edit by hand, or roll the asset back through
source control) exists — but for `remove_node` that workaround requires the caller to have
recorded the node's settings before deleting it, which nothing in the response tells them to
do. Reach modifier not applied: PCG graph authoring is a deliberate, occasional workflow, not
an almost-every-session path; that argues a bump *down*, and the `remove_node` unrecoverability
argues a bump *up*, so they are recorded as cancelling rather than either being silently
applied.

Declined High: nothing is silently wrong and no result is a lie — the edits land exactly as
asked. Declined Low: this is not friction, it is a destructive edit with no reverse.

**Workaround:** save the graph asset before any `pcg.remove_node`, and recover with
`source_control.revert` (now package-resynchronising per
`B-source-control-revert-no-package-reload` `#6`) rather than `editor.undo`. Read the node's
properties with `pcg.inspect` before deleting it.

**Fix:** wrap each mutator in an `FScopedTransaction` and `Modify()` the graph before the
edit, matching the shape `connect_pins` (`PCGGraphAuthoring.cpp:202`) and `set_node_property`
(`PCGSetNodeProperty.cpp:277`) already use — the two in-module templates, so there is no design
to invent. Preferably in one commit across the eight, since a namespace where some verbs undo
and some do not is worse to reason about than one where none do.

## History
- `#1-namespace-wide-transaction-gap` `OPEN` reporter — Source-only, censused against the current working tree (which carries the uncommitted `connect_pins` transaction from `B-pcg-connect-pins-silently-replaces-edge` `#2`; that fix is why the module count is now 3 and not 1). `grep -rn FScopedTransaction Source/PinWrightPCG/` returns exactly three executable occurrences — `PCGGenerateHandler.cpp:156`, `PCGGraphAuthoring.cpp:202`, `PCGSetNodeProperty.cpp:277` — plus one mention inside a registration description string. `pcg.add_node`'s body reaches `Graph->AddNodeOfType(...)` and `pcg.remove_node`'s reaches `Graph->RemoveNode(Node)` with no `Modify()` and no transaction in either. `Modify()` count per file: `PCGGraphAuthoring.cpp` 2 (both inside the `connect_pins` transaction), `PCGAddNoiseFilter` / `PCGAddSlopeFilter` / `PCGAddSubgraph` / `PCGGraphParameters` / `PCGSetSelfPruningSettings` **0** each. **Explicitly source-only: `editor.undo` was NOT exercised against any `pcg.*` mutation this pass** — same limitation `B-foliage-mutators-no-transaction` `#2` records for itself. What is established is the mechanism (no transaction opened, so no undo record can be written), which is a strong inference and not an observation. Filed despite `B-pcg-connect-pins-silently-replaces-edge` explicitly declining to file it, per a standing instruction to file everything found; that ticket's own text names this as "a namespace-wide gap deserving its own ticket". Severity Medium, argued above and anchored to the sibling `B-foliage-mutators-no-transaction`, which carries the same impact class at the same rating.
