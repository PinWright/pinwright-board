---
id: B-bpir-upsert-skips-inputkey-entries
title: "compile_bpir upsert (append/replace) silently duplicates key_pressed/key_released InputKey entries instead of replacing them"
status: DONE
severity: High
category: bug
tags: [bpir, compile-bpir, phase0, upsert, input-key, remove-event, silent-duplicate]
---

# compile_bpir upsert skips InputKey entries → silent duplicate

`blueprint.compile_bpir` in `append`/`replace` mode is documented as an idempotent
upsert: matching existing entry nodes are deleted before re-creation so that
re-authoring the same entry does not accumulate duplicates. This contract holds
for `custom_event`, `event`, `override`, `function`, `macro`, and `construction`
entries — but **not** for `key_pressed` / `key_released` entries
(`UK2Node_InputKey`). Re-authoring an existing `entry key_pressed <Key>()` adds a
**second** `UK2Node_InputKey` node for the same key instead of replacing the first,
and the compile reports `success: true` with no error and no warning. The graph
ends up with two input-key event nodes that both fire on the same key press.

## Symptom (observed in a live session)

`compile_bpir` (default `mode: "append"`) was called to re-author an entry that
already existed on `/App/HELIOS/Framework/HELIOS_BP`:

```
entry key_pressed Tab() { ... new guarded body ... }
```

The BP already had an `entry key_pressed Tab()` (a `UK2Node_InputKey` event node).

- **Expected (per docs):** the existing `key_pressed Tab` node + its exec subgraph
  are deleted, then the new body is compiled — net one Tab handler.
- **Actual:** compile returned `{success:true, compiled:true, errors:[], createdNodes:[8 GUIDs]}`
  with no error/warning, and the OLD Tab event was NOT deleted. The graph ended up
  with **two** `key_pressed Tab` `UK2Node_InputKey` nodes (old at y=6045, new at
  y=6496). Both fire on Tab press. The old one had to be removed manually with
  `blueprint.graph.delete_node`.

**Secondary symptom (same root cause, different handler):**
`blueprint.remove_event {eventName:"Tab", nodeId:"<old InputKey GUID>"}` returned
`{removedNodeCount:0, note:"Event not present; treated as removed (idempotent)."}`
— `remove_event` also does not recognize `UK2Node_InputKey` nodes, so even
targeting the exact node GUID fails to remove it.

## Minimal repro

1. On any Blueprint, compile `entry key_pressed Tab() { call PrintString(InString: "a"); }`
   in default (`append`) mode → one `UK2Node_InputKey` (key=Tab) is created.
2. Compile the same (or a different body) `entry key_pressed Tab() { ... }` again
   in default mode.
3. **Expected:** still one Tab `UK2Node_InputKey`. **Actual:** two Tab
   `UK2Node_InputKey` nodes, no error/warning. (`blueprint.decompile` shows two
   `entry key_pressed Tab` blocks; `blueprint.graph.find_nodes query:"K2Node_InputKey"`
   returns two.)
4. Try `blueprint.remove_event {eventName:"Tab", nodeId:"<one InputKey GUID>"}` →
   `removedNodeCount:0`, idempotent no-op note, node not removed.

## Root cause (confirmed against source)

`key_pressed` / `key_released` BPIR entries compile to `UK2Node_InputKey` (with the
key stored in the `InputKey` FKey property), not `UK2Node_CustomEvent`:

- `BpirCompiler.cpp:4519` — `SetupKeyEvent` calls
  `NodeEmitter->CreateInputKeyNode(FName(*KeyName), bReleased)`.
- `CodeNodeEmitter.cpp:843-851` — `CreateInputKeyNode` creates a `UK2Node_InputKey`
  and sets `Node->InputKey = FKey(KeyName)`.
- UE class hierarchy: `UK2Node_InputKey : public UK2Node` — it does **not** derive
  from `UK2Node_Event` or `UK2Node_CustomEvent`.

The Phase 0 (replace-mode) deletion pass groups `KeyPressed`/`KeyReleased` under the
same branch as `CustomEvent`, but that branch only matches `UK2Node_CustomEvent`:

- `BpirCompiler.cpp:2629-2649` — for `Block.Kind == CustomEvent || KeyPressed || KeyReleased`,
  the loop does `UK2Node_CustomEvent* CE = Cast<UK2Node_CustomEvent>(Node); if (!CE) continue;`
  then matches `CE->CustomFunctionName == EntryName`. An existing `UK2Node_InputKey`
  is never a `UK2Node_CustomEvent`, so the cast returns null, the node is skipped,
  and nothing is added to `NodesToDelete`. The key (`InputKey` FKey) is never
  consulted, and the entry name (e.g. "Tab") is never compared against an InputKey
  node's key.

With nothing deleted, the normal entry-emit loop runs `SetupKeyEvent`
unconditionally — and unlike `SetupCustomEvent` (which has a duplicate-detection /
reuse path at `BpirCompiler.cpp:4231-4341`), `SetupKeyEvent`
(`BpirCompiler.cpp:4511-4546`) has **no** reuse/duplicate check. It always calls
`CreateInputKeyNode`, producing a fresh second `UK2Node_InputKey`.

**Why it is silent:** the post-compile duplicate safety-net
(`BpirCompiler.cpp:3196-3218`) only counts `UK2Node_CustomEvent` nodes by
`CustomFunctionName`; it never inspects `UK2Node_InputKey`. So duplicate input-key
entries trigger no warning, and the failure mode is a clean
`success: true` with a silent duplicate.

## Secondary gap: remove_event ignores UK2Node_InputKey

`blueprint.remove_event` (`BlueprintEventHandler.cpp:432-504`) scans
`UbergraphPages` and matches only `UK2Node_CustomEvent` (by `CustomFunctionName`),
`UK2Node_ComponentBoundEvent`, `UK2Node_ActorBoundEvent`, and `UK2Node_Event` (by
`EventReference.GetMemberName()`). `UK2Node_InputKey` derives from `UK2Node`, not
`UK2Node_Event`, so none of these casts match — even when `nodeId` exactly matches
the InputKey node's GUID, the node is never added to `EventRootNodes`, yielding
`removedNodeCount:0` and the idempotent "Event not present" note. This is the same
class-coverage gap as the compiler's Phase 0 matcher, in a second handler.

## Scope note

BPIR only emits `UK2Node_InputKey` today (via `key_pressed`/`key_released`); there
is no `input_action_pressed` entry kind in the compiler/parser
(`BpirParser.cpp:744-750`), so the immediate fix is scoped to `UK2Node_InputKey`.
If/when input-action entry compilation is added (the decompiler already emits
`input_action_event` etc. — see `B-bpir-input-event-entry-signatures-unknown`),
the upsert matcher and `remove_event` must match `UK2Node_InputActionEvent` by
`InputActionName` and `UK2Node_InputKeyEvent` by its key/`InputKeyEvent` for the
same reason.

## Proposed fix direction (not implemented)

1. **Extend the Phase 0 upsert matcher** in `BpirCompiler.cpp:2629-2649` so the
   `KeyPressed`/`KeyReleased` kinds match existing `UK2Node_InputKey` nodes by their
   `InputKey` FKey (compare against the entry name, using the same key-name parsing
   `SetupKeyEvent`/`CreateInputKeyNode` use to build the FKey, e.g.
   `FBpirInputKeyHelpers::FormatInputKeyAsBpirIdentifier` for the reverse mapping).
   Split the InputKey case out of the `UK2Node_CustomEvent` cast branch — they are
   different node classes and cannot share a cast. On match, append
   `CollectSubgraphNodes(InputKeyNode)` to `NodesToDelete` so re-authoring is
   idempotent. (Pairs `Released` vs `Pressed`: optionally also disambiguate by the
   pressed/released sense, though a Tab key node carries both exec outputs, so
   matching by key alone is sufficient to dedup.)
2. **Surface non-replaced entries in the result** so silent duplication is
   impossible to miss: have `compile_bpir` report a per-entry
   `replacedEntries` / `newEntries` breakdown (or at minimum extend the existing
   post-compile duplicate safety-net at `BpirCompiler.cpp:3196-3218` to also count
   `UK2Node_InputKey` duplicates by key and emit a warning). Either makes the
   duplicate visible in the response.
3. **Fix `remove_event` symmetrically**: in `BlueprintEventHandler.cpp:432-504`
   add a `UK2Node_InputKey` branch that matches by the node's `InputKey` (and honors
   the `nodeId` disambiguator), so InputKey events can be removed at all.

**Fix:** extend the upsert entry matcher and `blueprint.remove_event` to recognize `UK2Node_InputKey` by the BPIR key identifier, and use the incoming entry kind's active exec pin (`Pressed` for `key_pressed`, `Released` for `key_released`) when deciding whether an existing InputKey node is the matching entry. Extend the duplicate safety-net to warn on duplicate `UK2Node_InputKey` entries by key + active exec sense.

## History
- `#1-initial-repro` `OPEN` reporter — Found in a live session re-authoring `entry key_pressed Tab()` on `/App/HELIOS/Framework/HELIOS_BP` via `compile_bpir` default append mode. Compile returned `success:true, errors:[]` but did not delete the pre-existing Tab `UK2Node_InputKey`; graph ended with two Tab input-key nodes (y=6045 old, y=6496 new). Root cause confirmed against source: Phase 0 deletion branch for `KeyPressed`/`KeyReleased` (`BpirCompiler.cpp:2629-2649`) only casts to `UK2Node_CustomEvent`, never matching `UK2Node_InputKey` (which derives from `UK2Node`, not `UK2Node_Event`); `SetupKeyEvent` (`BpirCompiler.cpp:4511-4546`) then creates a fresh node unconditionally with no reuse/duplicate check; post-compile duplicate safety-net (`BpirCompiler.cpp:3196-3218`) only counts `UK2Node_CustomEvent`, so the duplicate is silent. Same class-coverage gap makes `blueprint.remove_event` (`BlueprintEventHandler.cpp:432-504`) return `removedNodeCount:0` for an InputKey node even when given its exact `nodeId`. Distinct from `B-compile-bpir-retry-duplicates` (general Phase 0 upsert, never covered InputKey), `B-bpir-input-event-entry-signatures-unknown` (decompiler `UnknownEntry` only), and `E-remove-event-multi-match-dedup` (ComponentBoundEvent disambiguation only). Manual `blueprint.graph.delete_node` was required to repair.
- `#2-inputkey-upsert-remove` `IN-REVIEW` developer — Changed BpirInputKeyHelpers.h, BpirCompiler.cpp, and BlueprintEventHandler.cpp so append/replace upsert and blueprint.remove_event match UK2Node_InputKey by BPIR key identifier and active Pressed/Released entry sense, preventing silent duplicate input-key handlers without deleting unrelated keys. Added regression test FBpirInputKeyUpsertAndRemoveEventTest covering double compile and GUID-targeted removal.
- `#3-verify-fix` `DONE` tester — Verified live: created temp Actor BP, compiled `entry key_pressed Tab()` twice in default (append) mode (both `success:true`), then `blueprint.graph.find_nodes query:"K2Node_InputKey"` returned `matchCount:1` (single Tab node, GUID `56A96AAE48041B0DD2B24FBBE32C9AFD` = the second compile's created node), confirming the pre-existing InputKey node was deleted before re-creation instead of duplicated. Temp BP deleted afterward.
