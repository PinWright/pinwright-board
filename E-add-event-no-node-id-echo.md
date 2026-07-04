---
id: E-add-event-no-node-id-echo
title: "blueprint.add_event returns only eventName, never the created entry node's GUID — forcing a get_nodes readback before you can connect_pins to it (its sibling create_node echoes nodeId)"
status: OPEN
severity: Low
category: ergonomic
tags: [blueprint, add_event, node-id, no-echo, readback, response-shape, creation-verb-no-node-id]
encounters: 1
lastSeen: 2026-07-04T19:17:43.9468829+03:00
---

# `blueprint.add_event` omits the created node's id — you can't wire the event without a follow-up `get_nodes`

`blueprint.add_event` creates a real EventGraph entry node (a `UK2Node_Event` /
`UK2Node_CustomEvent`), but its success response reports only the event's *name*
and *type* — never the created node's GUID. So the very next natural step —
wiring the event's `then` exec pin into downstream logic via
`blueprint.graph.connect_pins` — cannot be done from the `add_event` result
alone: `connect_pins` needs the entry node's id, and the caller has to make a
separate `blueprint.graph.get_nodes` round-trip purely to recover the GUID the
handler already had in hand.

This is inconsistent with the sibling node-creation verb in the same namespace,
`blueprint.graph.create_node`, which *does* echo `nodeId` (the created node's
GUID) so its result is immediately connectable. `add_event` is the odd one out.

## What it should do

Add the created (or found-existing) entry node's GUID to the `add_event`
response — e.g. `nodeId` (matching `create_node`'s field name) plus optionally
`nodeName` — so the returned event is directly addressable by
`connect_pins` / `set_pin_default_value` with no follow-up inspection call. The
handler already creates the node locally; it just lets it go out of scope
without recording its GUID.

## Affected methods

- `blueprint.add_event` — all event kinds. The response is built once at
  `BlueprintEventHandler.cpp:359-367` and both the custom-event branch
  (`bIsCustomEvent`) and the standard/lifecycle branch (BeginPlay/Tick/EndPlay)
  fall through to it, so no `add_event` variant returns a node id.

(Probed the sibling verbs per defect-family discipline: `blueprint.graph.create_node`
echoes `nodeId` and is NOT affected — this omission is confined to `add_event`,
so it is filed as a single-method ticket rather than a family one.)

## Guilty source

`Plugins/PinWright/Source/PinWright/Private/Handlers/Blueprint/BlueprintEventHandler.cpp:359-367`
— the response builder never captures the node it just created (`EventNode` is a
local scoped inside the `if (!bExists)` create block at lines 308-314 and is
never read back for its `NodeGuid`):

```cpp
    TSharedPtr<FJsonObject> Resp = MakeShared<FJsonObject>();
    Resp->SetBoolField(TEXT("success"), true);
    Resp->SetStringField(TEXT("blueprintPath"), RegistryKey);
    Resp->SetStringField(TEXT("eventName"), EventName.ToString());
    Resp->SetStringField(TEXT("eventType"), FinalType);
    Resp->SetBoolField(TEXT("saved"), bSaved);
    if (Params.Num() > 0) Resp->SetArrayField(TEXT("parameters"), Params);
    AddAssetVerification(Resp, BP);
    Ctx.SendSuccess(Resp);
```

Contrast the sibling that gets it right —
`Plugins/PinWright/Source/PinWright/Private/Handlers/Blueprint/BlueprintGraphCrudHandler.cpp:366`:

```cpp
            Result->SetStringField(TEXT("nodeId"), NewNode->NodeGuid.ToString());
```

## Verbatim repro

RPC path: `mcp__pinwright__call` method `blueprint.add_event`.

`blueprint.add_event` args `{"path":"/Game/BP_OracleReplayEvent","eventType":"BeginPlay","x":0,"y":0}` →

```json
{"success":true,"blueprintPath":"/Game/BP_OracleReplayEvent","eventName":"ReceiveBeginPlay","eventType":"BeginPlay","saved":true,"assetPath":"/Game/BP_OracleReplayEvent","assetName":"BP_OracleReplayEvent","existsAfter":true,"assetClass":"Blueprint"}
```

(no `nodeId` / node GUID anywhere in the result.)

For contrast, `blueprint.graph.create_node` args
`{"path":"/Game/BP_OracleReplayEvent","nodeType":"CallFunction","memberName":"PrintString","x":400,"y":0}` →

```json
{"nodeId":"0D6FE3C840093CD92407E1B37D66396F","nodeName":"K2Node_CallFunction_0","assetPath":"/Game/BP_OracleReplayEvent","assetName":"BP_OracleReplayEvent","existsAfter":true,"assetClass":"Blueprint"}
```

Workaround: after `add_event`, call `blueprint.graph.get_nodes` and find the
new event node by type/name to recover its GUID before connecting — one extra
round-trip per event you author. (On a large EventGraph that `get_nodes` readback
can itself overflow the inline limit and spill to disk — cf.
`E-get-nodes-pins-spill-no-projection` — compounding the cost.)

## Distinct from

- `E-add-event-then-default-compile-bpir-unundoable` (IN-REVIEW) — about
  undoability of `add_event` + default `compile_bpir`; nothing to do with the
  response shape / node id. Different root cause.
- `E-ai-bt-authoring-verbs-dead-end` (IN-REVIEW) — the `ai.*` BT-authoring verbs
  return "success but no node id", but there the surface is a *deprecated dead
  end* with no id-based alternative (the fix is to disable/redirect it). Here
  `add_event` is a *live, correct* authoring verb whose only gap is the missing
  id echo; the clean workaround (`get_nodes`) exists. Different namespace, different
  remedy.
- `E-compile-bpir-creatednodes-opaque` (OPEN) — `compile_bpir`'s `createdNodes`
  GUID list is present but opaque (no type/role). Different method; here the id
  is absent entirely, not merely unlabeled.

severity rationale: impact=friction (creation response omits the entry node's id,
forcing one `get_nodes` readback to address it) × reach=common BP event-graph
authoring but only bites when immediately wiring the event -> Low. Consistent with
the analogous `E-compile-bpir-creatednodes-opaque` (Low).

## History
- `#1-initial-repro` `OPEN` reporter — Filed from a realism task (build a small Actor BP `BP_SpawnBeacon`: BeginPlay -> PrintString -> Set bInitialized, wired end to end). The attempt completed but its friction note flagged that `blueprint.add_event` "returned only the event name (no node GUID), so I had to read the graph back to obtain the BeginPlay node GUID and exact pin names before connecting." Replay-confirmed on a fresh `/Game/BP_OracleReplayEvent` Actor BP: `blueprint.add_event {eventType:"BeginPlay"}` returned `eventName:"ReceiveBeginPlay"` with NO `nodeId`, while `blueprint.graph.create_node` on the same BP returned `nodeId:"0D6FE3C8…"`. Source-confirmed: `BlueprintEventHandler.cpp:359-367` builds the response without ever reading the created `EventNode`'s `NodeGuid` (the node is a local scoped inside the create block at 308-314); the sibling `BlueprintGraphCrudHandler.cpp:366` echoes `nodeId`. Both add_event branches (custom + lifecycle) share the one omitting response builder. Dedup: ripgrep across OPEN/closed — distinct from `E-add-event-then-default-compile-bpir-unundoable` (undo), `E-ai-bt-authoring-verbs-dead-end` (deprecated ai.* surface, no live alternative), and `E-compile-bpir-creatednodes-opaque` (opaque-not-absent id list). No existing ticket covers `add_event` omitting the created node's id. Proposed: echo `nodeId` (+ `nodeName`) on the `add_event` response so the event is directly connectable.
