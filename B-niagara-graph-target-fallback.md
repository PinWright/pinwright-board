---
id: B-niagara-graph-target-fallback
title: "Niagara graph mutators silently target Spawn for invalid script types and the last duplicate-title node"
status: IN-REVIEW
severity: High
category: bug
tags: [niagara, graph, wrong-target, connect-pins, remove-node, validation]
---

# Target resolution is permissive in two destructive dimensions

The shared `ResolveNiagaraGraph` selects Spawn first and changes to Update only when
`scriptType == "Update"` exactly (`NiagaraGraphHandler.cpp:47-81`). Any other non-empty value --
including case drift or a documented-stage spelling used elsewhere -- silently targets Spawn.
`connect_pins` and `remove_node` both use this resolver and then return success for that graph.

Inside `connect_pins`, a node may match by GUID, UObject name, or display title. The scan does not
stop or count matches; each later match overwrites the earlier pointer (`:282-293`). Duplicate
Niagara node titles are normal in real graphs, so a title resolves to the last matching node in
array order. The response then echoes the request strings rather than the resolved node GUIDs
(`:341-348`), hiding which nodes were actually wired.

These paths can edit a valid but unintended graph/node and report success. The GUID-only
`remove_node` lookup demonstrates the safe identity shape already in the same file.

## What it should do

Parse `scriptType` case-insensitively against an explicit Spawn/Update allow-list and reject
unknown values. Resolve node GUIDs first; if name/title support remains, require exactly one match
and return `AMBIGUOUS_NODE`. Include resolved graph usage and node GUIDs in the success response.

## Workaround

Pass exact `Spawn` or `Update` and use node GUIDs, never display titles.

## Related

- `B-niagara-connect-pins-sends-no-graph-notification`

## Fix

Root cause: `ResolveNiagaraGraph` treated every script type other than the exact
case-sensitive string `Update` as `Spawn`, so unknown targets silently selected a
valid graph; connect-node lookup also overwrote earlier alias matches and success
responses echoed request identity. It now trims the optional value, accepts only
case-insensitive `Spawn` or `Update`, and rejects every other supplied value with
`TARGET_NOT_FOUND` naming the requested value and both valid candidates before graph
selection or mutation. Graph extraction uses `NiagaraJsonHelpers::GetGraphFromScript`;
connect-node lookup is GUID-first, accepts a non-GUID name/title only when unique,
and rejects ambiguous aliases with `AMBIGUOUS_NODE` plus candidate GUIDs before pin,
schema, notification, or mutation work. Successful connect/remove responses report
canonical graph usage and node GUID identity. Because real Spawn and Update scripts
share one graph, resolved connect endpoints and GUID-only removals are also checked
against `UNiagaraGraph::BuildTraversal` for the selected script usage and usage ID.
Nodes reachable only from another output return `TARGET_NOT_FOUND` naming the requested
canonical usage and every actual reachable usage before pin/schema/removal work. Nodes
reachable from both outputs remain valid for either target, and disconnected nodes
remain available for authoring.

The suite also exposed a GUID-looking alias edge: UE's `FGuid::Parse` accepts 22-character
Base64 short GUID strings, so `SharedGraphSpawnToNode` was parsed as a GUID and returned
`NODE_NOT_FOUND` before its exact name/title alias could be considered. The resolver now
keeps real GUID matches first, but falls through to exact alias collection when a parsed
GUID matches no node; `NODE_NOT_FOUND` is returned only when both lookup forms fail.

Files changed:

- `Plugins/PinWright/Source/PinWright/Private/Handlers/Niagara/NiagaraGraphHandler.cpp`
- `Plugins/PinWright/Source/PinWright/Private/Tests/Assets/TestNiagaraHandlers.cpp`
- `Plugins/PinWright/docs/wiki-src/niagara.graph.md`

Behavioral test ids:

- `PinWright.niagara.graph.connect_pins.RejectsUnknownScriptTypeWithoutMutation`
- `PinWright.niagara.graph.connect_pins.RejectsAmbiguousAliasWithoutMutation`
- `PinWright.niagara.graph.connect_pins.DefaultsToSpawnAndReturnsCanonicalIdentity`
- `PinWright.niagara.graph.connect_pins.TargetsMixedCaseUpdate`
- `PinWright.niagara.graph.connect_pins.RejectsSpawnNodeForUpdateOnSharedGraph`
- `PinWright.niagara.graph.connect_pins.AcceptsNodeSharedBySpawnAndUpdate`
- `PinWright.niagara.graph.remove_node.RejectsUnknownScriptTypeWithoutMutation`
- `PinWright.niagara.graph.remove_node.RejectsSpawnNodeForUpdateOnSharedGraph`
- `PinWright.niagara.graph.remove_node.ReturnsCanonicalIdentity`

Deliberately unchanged scope: unrelated `NiagaraEditHandler.cpp` fallback findings
remain out of scope; duplicate-node mutation behavior beyond connect-node alias
resolution and unrelated Niagara handlers were not changed.

## History
- `#1-source-scan` `OPEN` reporter -- Both wrong-target branches and the request-echo response were
  confirmed in current source; no asset mutation was performed.
- `#2-strict-graph-target-resolution` `IN-REVIEW` developer -- Changed `NiagaraGraphHandler.cpp`
  to reject unknown script targets with `TARGET_NOT_FOUND` before graph selection; added handler
  tests proving rejected `connect_pins` and `remove_node` requests do not mutate Spawn or Update
  graphs. Duplicate-node ambiguity and canonical success responses remain deliberately unchanged.
- `#3-canonical-node-resolution` `IN-REVIEW` developer -- Corrected the first implementation to
  cover the ticket's full contract: `NiagaraGraphHandler.cpp` now uses
  `NiagaraJsonHelpers::GetGraphFromScript`, resolves connect aliases GUID-first, refuses ambiguous
  aliases with candidate GUIDs, and reports canonical `scriptUsage`/node GUIDs on success. Added
  mixed-case Update, omitted-Spawn, ambiguous-alias, and canonical remove-node handler coverage;
  unrelated `NiagaraEditHandler.cpp` fallback findings remain out of scope.
- `#4-shared-graph-usage-scope` `IN-REVIEW` developer -- Corrected the production topology gap:
  connect endpoints and GUID removals now use output traversal membership to reject nodes owned
  only by another usage while accepting shared and disconnected nodes. Replaced the distinct-graph
  fixture with one shared source/graph and added counterfactual Spawn-via-Update refusal plus
  shared-node acceptance coverage.
- `#5-guid-looking-alias-fallback` `IN-REVIEW` developer -- Suite failure
  `PinWright.niagara.graph.connect_pins.DefaultsToSpawnAndReturnsCanonicalIdentity` traced the
  missing Spawn result to `FGuid::Parse` accepting the 22-character Base64 alias
  `SharedGraphSpawnToNode`; `ResolveNiagaraGraphNode` now preserves GUID-first precedence but
  falls through to exact name/title aliases when no node matches the parsed GUID, returning
  `NODE_NOT_FOUND` only after both forms fail.
