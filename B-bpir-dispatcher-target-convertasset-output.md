---
id: B-bpir-dispatcher-target-convertasset-output
title: "BPIR `bind_dispatcher` / `clear_dispatcher` with a `K2Node_ConvertAsset` (Resolve Soft Reference) output as Target fails with 'Failed to create delegate node for dispatcher', although the decompiler emits exactly that shape"
status: OPEN
severity: Medium
category: bug
tags: [bpir, bind-dispatcher, clear-dispatcher, convert-asset, soft-reference, round-trip, insert-bpir-at-node]
encounters: 1
lastSeen: 2026-10-01T08:22:00Z
---

# Dispatcher ops on a Resolve Soft Reference output do not compile

UE 5.8 Linux, host `/sdb-disk/src/unreal/unreal-fpv` (PDS), plugin `1044f6de`.
`blueprint.insert_bpir_at_node` into `/App/App/UI/LobbyAndMenu/TrackMenu/W_RaceResultsHandler`
(a UWidgetBlueprint) with this body:

```
%frame: object<W_RaceOnlineResultsForPilotFrame_C> = call K2Node_ConvertAsset(Input: %dgi.AsBDroneGameInstance.RaceOnlineResultsForPilotFrame)
...
clear_dispatcher OnShowResults(Target: %frame.Output)
bind_dispatcher OnShowResults(Target: %frame.Output, event: @OnShowResults)
```

failed with:

```
[COMPILE_FAILED] Line 17: Failed to create delegate node for dispatcher: OnShowResults; Line 17: EmitInstruction failed for opcode 32; ...
Line 20: Failed to create delegate node for dispatcher: OnShowResults; Line 20: EmitInstruction failed for opcode 27; ...
```

The same Blueprint's own graph already contains this shape, and `blueprint.decompile` prints it:
`clear_dispatcher OnClose(Target: %n1.Output)` where `%n1 = call K2Node_ConvertAsset(Input: $sender)`.
So decompiled BPIR does not round-trip through the compiler. The dispatchers are declared on the
target widget BP class (`W_RaceOnlineResultsForPilotFrame_C`); the declared `%frame` type names that
class, so the class is known. The likely cause is that the dispatcher lookup reads the
ConvertAsset output pin's type before the node is wired, when it is still a wildcard.

**Workaround (worked):** cast the resolved object first and use the cast output as Target:
`%typed: object<W_X_C> = cast<W_X_C>(%frame.Output) [success -> @ok]`, then
`bind_dispatcher OnShowResults(Target: %typed.AsWX..., event: @OnShowResults)`. This costs a redundant
cast node in the shipped graph.

## Expected

Resolve the dispatcher owner class for a `K2Node_ConvertAsset` output from its softobject input's
subtype (or from the declared `object<T>` type), so the decompiled shape compiles without a cast.

## History
- `#1-convertasset-target-dispatcher` `OPEN` reporter - Filed from the PDS results-button fix (UE 5.8 Linux, plugin `1044f6de`). Six dispatcher lines failed as above; inserting a `cast<>` and targeting its output made the same insert compile cleanly (`success: true, status: UpToDate`). Not costly: one extra call.
