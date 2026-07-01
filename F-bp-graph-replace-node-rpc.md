---
id: F-bp-graph-replace-node-rpc
title: "Add blueprint.graph.replace_node — substitute a node in place, preserving wiring and defaults"
status: DONE
severity: Medium
category: feature
tags: [blueprint-graph, ergonomics, mcp-rpc]
---

# Missing: in-place node replacement

`blueprint.graph` currently forces a delete+create+reconnect dance to swap one
node for a same-shape sibling — change a `CallFunction` target, retype a `Cast`,
swap `VariableGet` for `VariableSet`, replace a deprecated function call with a
new one. Each callsite must `get_node_details` to capture neighbors, `delete_node`,
`create_node` at the same position, then issue N `connect_pins` calls and N
`set_pin_default_value` calls to re-establish state — verbose, error-prone, and
typically eats five+ round-trips per swap. The orphan-cleanup and
delegate-cascade bookkeeping that `delete_node` does internally is invisible to
callers stitching this manually, which has produced corruption regressions in
the past (see related saved-state-corruption tickets).

The canonical UE-internal pattern already exists — `FBlueprintEditor::ConvertEventToFunction`
(`Editor\Kismet\Private\BlueprintEditor.cpp:6687`) and the event→custom-event
swap at `Editor\UnrealEd\Private\Kismet2\BlueprintEditorUtils.cpp:6649-6660` do
exactly this via `FEdGraphUtilities::GetPinConnectionMap` /
`ReconnectPinMap` / `CopyPinDefaults`, or per-pin
`UEdGraphSchema::MovePinLinks(*OldPin, *NewPin, false, /*bNotifyLinkedNodes=*/true)`.
All required APIs are DLL-exported in modules the plugin already depends on
(ENGINE_API, UNREALED_API, BLUEPRINTGRAPH_API).

**Proposed:** add `blueprint.graph.replace_node` (params: `nodeId`,
`newNodeType` + the same class-specific params as `create_node` — `memberName`,
`memberClass`, `variableName`, `eventName`, `targetClass` — plus `pinRemap`,
`preservePosition`, `transferConnections`, `transferDefaults`,
`cleanupNewOrphans`). Implementation reuses `create_node`'s class-resolution
switch, `delete_node`'s orphan/delegate cleanup path
(`SnapshotBlueprintOrphanGuids` + `CleanupNewBlueprintOrphans`,
`CascadeRemoveStaleCreateDelegates`, `GetEffectiveFunctionNameForRemoval`), and
the file-static `FindPinByName` resolver to transfer `LinkedTo` and
`DefaultValue` / `DefaultObject` / `DefaultTextValue` across matched pins.
Reports dropped connections explicitly so callers can audit. Wraps the whole
operation in `FScopedTransaction` for clean Ctrl+Z. Wiki delta is one-line on
`docs/wiki/blueprint.graph.md` mutate-nodes bullet.

**Fix sketch (call sequence):**

```cpp
FScopedTransaction Tx(LOCTEXT("ReplaceNode","Replace Node"));
BP->Modify(); Graph->Modify(); OldNode->Modify();
Schema->ReconstructNode(*OldNode);                              // refresh old pin set
TMap<FString,TSet<UEdGraphPin*>> PinMap;
FEdGraphUtilities::GetPinConnectionMap(OldNode, PinMap);
UK2Node* New = FEdGraphSchemaAction_K2NewNode::SpawnNodeFromTemplate(Graph, Template, FVector2D(OldX,OldY), false);
FEdGraphUtilities::ReconnectPinMap(New, PinMap);                // or per-pin MovePinLinks for type-aware transfer
FEdGraphUtilities::CopyPinDefaults(OldNode, New);
FBlueprintEditorUtils::RemoveNode(BP, OldNode, /*bDontRecompile=*/true);
FBlueprintEditorUtils::MarkBlueprintAsStructurallyModified(BP);
```

**Edge cases to handle in the handler:**
- Orphan pins (old has name not in new): abort with `PIN_REMAP_INVALID` listing
  offenders by default; opt-in mode synthesizes `bOrphanedPin=true` placeholders
  on the new node (red wires visible) following `UK2Node::RewireOldPinsToNewPins`
  at `K2Node.cpp:1397`.
- Self pin: skip by default when replacing variable accessors (mirror
  `FUpdatePastedNodes::MoveAllLinksExeptSelf` at `BlueprintEditorUtils.cpp:7402`)
  so the new node's own self context wins.
- Wildcard pins on the *other* side: pass `bNotifyLinkedNodes=true` so they
  re-resolve type.
- `bDontRecompile=true` on `RemoveNode` (otherwise O(N) compiles per replace).
- Split struct pins: iterate `OldNode->Pins[]` (parents before children); name
  match handles dotted sub-pin paths.

## History
- `#1-initial-feature-request` `OPEN` reporter — Missing imperative replace-node RPC; current delete+create+reconnect dance is verbose and skirts orphan/delegate cleanup that `delete_node` does internally. Canonical UE pattern available via `FEdGraphUtilities` + `MovePinLinks`; all APIs DLL-exported on existing module deps.
- `#2-implemented-replace-node-handler` `IN-REVIEW` developer — Added `blueprint.graph.replace_node` after the `delete_node` block in `BlueprintGraphHandler.cpp`: per-class spawn factory mirroring `create_node` (CallFunction / VariableGet/Set / Event / CustomEvent / DynamicCast / ClassDynamicCast / CreateDelegate / Branch / Sequence / Select / MakeArray), refusal list for graph terminators / composites / async / bound events, per-pin `MovePinLinks` with `(name, direction)` matching plus optional orphan placeholders, default-value transfer for unconnected pins, `bDontRecompile=true` `RemoveNode` + delegate cascade + UFunction scrub + orphan-delta sweep, all wrapped in a single `FScopedTransaction` that cancels on early-exit; wiki bullet updated on `docs/wiki/blueprint.graph.md`.
- `#3-applied-v1.1-cleanup` `IN-REVIEW` developer — Collapsed the param surface from nine class-specific params to a single unified `target` string (`bare` / `Class::Member` / `Class.Member`), dropped `preservePosition`/`transferConnections`/`transferDefaults`/`cleanupNewOrphans` toggles (now always-on); added cross-graph GUID lookup so omitting `graphName` scans every blueprint graph for the unique match; widened the refusal list to actor/generated bound events, every input-event variant, macro instances, deprecated `DelegateSet`, and dead-class placeholders; switched to `MarkBlueprintAsStructurallyModified` so dependent BPs rebuild; added split struct sub-pin support, deep-copied `UserDefinedPins` on CustomEvent swaps, carried `EnabledState`/`bUserSetEnabledState`/`AdvancedPinDisplay`, CallFunction-to-CallFunction Self-pin handoff, bare-target class inference from the old node, no-op short-circuit before the transaction, and a verbose log line plus `subPinsSplit` in the response JSON. Wiki page got a dedicated section with param table, vocabulary list, refusal list, and four JSON examples.
- `#4-v1.2-generic-fallback-pin-remap-and-tests` `IN-REVIEW` developer — Added the v1.2 contract for a generic fallback allow-list of no-config `UK2Node` classes, strict default pin preservation with `PIN_REMAP_INVALID`, explicit `pinRemap` handling before automatic matching, `allowOrphanPlaceholders` as the red-wire escape hatch, expanded result fields (`factoryPath`, `resolvedClass`, `pinRemapApplied`, `pinRemapUnmatched`, `targetIgnored`), `CanUserDeleteNode()` refusal coverage, focused automation coverage for explicit branches / generic fallback / pin migration / lifecycle refusal, and wiki documentation with Branch-to-Select examples.
- `#5-verify-branch-to-sequence` `DONE` tester — Verified: created temp BP `/Game/App/UI/Test/BP_McpVerifyTemp_replace_node`, compiled a Branch via BPIR, then called `blueprint.graph.replace_node` with `newNodeType:"Sequence"` and `pinRemap:{then:"then_0", else:"then_1"}`. Response: `connectionsRewired: 3`, `pinRemapApplied: 2`, `connectionsDropped: []`, `factoryPath:"explicit"`, `resolvedClass:"K2Node_ExecutionSequence"`, no orphans. First attempt without `pinRemap` correctly failed with `PIN_REMAP_INVALID` listing the unmatched wired pins, confirming the strict-preservation contract from `#4`. Temp BP deleted.
