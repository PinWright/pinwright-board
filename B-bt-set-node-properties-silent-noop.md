---
id: B-bt-set-node-properties-silent-noop
title: "behavior_tree.set_node_properties reports success when every property fails to apply (unknown key OR unconvertible value silently swallowed)"
status: IN-REVIEW
severity: High
category: bug
tags: [behavior-tree, set-node-properties, silent-noop, misleading-success, struct-property, value-or-bbkey, ai]
---

# `behavior_tree.set_node_properties` silently swallows failed/unknown property writes and still reports success

`behavior_tree.set_node_properties {assetPath, nodeId, properties}` applies each
key in `properties` to the node's `NodeInstance` via `ApplyBTNodeProperties`
(`BehaviorTreeHandler.cpp:222`), which for each key calls
`FindPropertyCI(...)` then `ApplyJsonValueToProperty(...)`. Both failure paths
are discarded and the handler unconditionally returns success:

- **Unknown property key** — `FindPropertyCI(Target->GetClass(), key)` returns
  `nullptr`, the loop `continue`s (`BehaviorTreeHandler.cpp:233-237`). No error.
  `FindPropertyCI` (`PropertyInspection.cpp:19`) only matches a *top-level*
  property name (exact or case-insensitive); it does **not** resolve dotted
  paths, so a key like `WaitTime.DefaultValue` is treated as one unknown
  property name and dropped.
- **Unconvertible value** — `ApplyJsonValueToProperty(...)` returns `false` with
  an error string, but the caller ignores both the bool and `ApplyError`
  (`BehaviorTreeHandler.cpp:245-248` — only `bModified = true` on success; the
  `else` is empty). For a struct-typed `FProperty` fed a bare JSON **number**,
  the struct branch falls through all of Array/Object/String and returns
  `"Unsupported JSON type for struct property"` (`PropertyImport.cpp:876`).

The handler (`BehaviorTreeHandler.cpp:893,896-904`) OR-accumulates only
successful applies into `bModified` and then **always** calls
`Ctx.SendSuccess(...)`. So when **every** supplied key fails — unknown name,
unconvertible value, or both — the call returns `ok:true` with no error, no
warning, no `dropped`/`skipped` list, and the node is unchanged. This is a
silent success-with-no-effect write.

## Why it bites a real task (UE 5.7 `Wait` regression)

In UE 5.7, `UBTTask_Wait::WaitTime` changed from a bare `float` to an
`FValueOrBBKey_Float` struct. The obvious, documented call to set a wait time —
`properties:{WaitTime: 2}` — now hits the struct branch with a bare number and
silently no-ops while reporting success. An agent that then "corrects" to
`properties:{"WaitTime.DefaultValue": 2}` *also* silently no-ops, because
`FindPropertyCI` does not resolve the dotted key — yet that call ALSO returns
success. Both reasonable attempts leave the Wait at its default time, and the
RPC reports success for both, so the agent reasonably (but wrongly) believes the
value landed. `behavior_tree.decompile` does not emit `WaitTime`, and
`asset.dump` only captures the top-level `UBehaviorTree` UPROPERTYs (just
`BTGraph`), so there is no readback that would reveal the silent drop.

This is the same misleading-success / silent-drop defect class already filed for
`data_table.add_row`/`set_row` (`B-data-table-row-values-silent-drop`),
`ai.configure_slot_behavior` (`B-configure-slot-behavior-ignores-behavior-and-tags`),
and the GAS/water tag-write families — a write RPC that drops part (or all) of
its input must not report unqualified success. It is distinct from
`B-widget-set-slot-struct-fails` (DONE): that fixed `ApplyJsonValueToProperty`'s
struct branch to *accept* nested-object / string-literal struct shapes AND it
surfaces the failure as an `INVALID_PROPERTY` error — whereas
`behavior_tree.set_node_properties` swallows the failure entirely.

## Verbatim repro (live, replay-confirmed via `mcp__editor-automation__call`)

Asset: `/Game/McpOracle/BT_OracleSvcReplay` (BT with a `Wait` task node
`64E397CE4F4B6ABD164696A9B9BE656C`).

1. **Bare-number struct value → silent no-op reported as success:**
   `behavior_tree.set_node_properties {assetPath:".../BT_OracleSvcReplay", nodeId:"64E397CE...", properties:{WaitTime: 3.5}}`
   → `{"assetPath":"...","assetName":"BT_OracleSvcReplay","existsAfter":true,"assetClass":"BehaviorTree"}` — `ok:true, is_error:false`.
   Internally `ApplyJsonValueToProperty` returns `false`
   (`"Unsupported JSON type for struct property"`); the result is unchanged but
   reports success. (Also reproduced with `WaitTime: 2`.)

2. **Dotted sub-field key → silent no-op reported as success:**
   `behavior_tree.set_node_properties {..., nodeId:"A974F3CD...", properties:{"WaitTime.DefaultValue": 7}}`
   → `ok:true` with the same shape. `FindPropertyCI(NodeInstance->GetClass(),
   "WaitTime.DefaultValue")` returns `nullptr` (no dotted resolution), the key is
   skipped, nothing is written, success is reported.

3. **Readback is blind:** `behavior_tree.decompile` after both calls emits
   `task Wait Wait @(...) ()` — no `WaitTime` in either case, so decompile cannot
   confirm OR refute the write; `asset.dump` writes a `properties.json` containing
   only `BTGraph`. The caller has no signal that the writes were dropped.

(Temp asset `/Game/McpOracle/BT_OracleSvcReplay` deleted after the replay.)

## What it should do

Adopt the board's established validate-before-mutate / surface-the-failure
convention. `ApplyBTNodeProperties` should collect, per supplied key: keys that
do not resolve to a property (`FindPropertyCI` null) and keys whose
`ApplyJsonValueToProperty` returns `false` (carry its `ApplyError`). If any key
failed, `behavior_tree.set_node_properties` should reject with `INVALID_PROPERTY`
/ `INVALID_PARAMS` carrying a `droppedFields` / `failed` list (key + reason) and
no partial write — or, at minimum, always echo a `droppedFields` / `skipped`
array alongside the result so the drop is visible. The all-valid path is
unchanged.

Separately (and lower priority, an ergonomic follow-on): to make the
`FValueOrBBKey_Float` family settable at all, either accept a bare number on the
struct branch by routing it into the struct's primary scalar sub-field
(`DefaultValue`), or have `FindPropertyCI`/the apply path resolve dotted
sub-field keys (`WaitTime.DefaultValue`). But the **bug** here is the silent
success regardless of which input shape is chosen.

## History
- `#3-additional-fix-confirmed-live` `IN-REVIEW` reporter — Additional evidence (independent realism task building `/Game/AI/BT_PatrolGuard`; replay-confirmed live via `mcp__editor-automation__call` on temp `/Game/McpOracle/BT_OracleReplaySetProps`, a BT with `Wait` task `D384FE074E281565671871A87378DE6D`, deleted after replay): the `#2` fix is in effect and the original silent-success bug is gone. `set_node_properties {properties:{WaitTime:3.5}, comment:"cooldown between patrol legs"}` now returns `ok:false` / `[INVALID_PROPERTY] 1 property could not be set on the node and was dropped: WaitTime: Unsupported JSON type for struct property` with body `{"droppedFields":[{"name":"WaitTime","reason":"Unsupported JSON type for struct property"}],"partiallyApplied":false}` — surfaced, not swallowed. The nested-struct shape `{properties:{WaitTime:{DefaultValue:3.5}}, comment:...}` succeeds, and `behavior_tree.decompile` now emits `WaitTime: "(DefaultValue=3.500000)"` plus the comment — so the readback is no longer blind (contrast `#1`'s "decompile omits WaitTime"). The ONLY residual friction is the deferred ergonomic follow-on this ticket already names (body "Separately ... lower priority"): a caller has no way to learn the accepted shape from the surface — the `droppedFields` reason (`"Unsupported JSON type for struct property"`) does not say "this is an `FValueOrBBKey_Float`; pass `{DefaultValue: <n>}`", and the `behavior_tree.set_node_properties` wiki documents `properties` only as generic "Key-value properties to set on the node instance". The attempt agent had to read the engine header to discover `{DefaultValue:3.5}`. No new ticket filed — this is the documented ergonomic tail of the (now-resolved) silent-success bug; folding the discoverability fix (accept a bare number for `FValueOrBBKey_*` by routing into `DefaultValue`, and/or name the accepted shape in the dropped-field reason / wiki) into this ticket's follow-on rather than a separate E- file.
- `#2-fix-surface-dropped-writes` `IN-REVIEW` developer — Fixed the silent success-on-dropped-write. `ApplyBTNodeProperties` (`Source/PinWright/Private/Handlers/AI/BehaviorTreeHandler.cpp`) now collects per-key failures into an out `TArray<FBTNodePropertyFailure>` instead of discarding them: an unknown key (`FindPropertyCI` null) records `{key, reason}` rather than a bare `continue`, and an `ApplyJsonValueToProperty` `false` return records `{key, ApplyError}` (the converter's message, e.g. `"Unsupported JSON type for struct property"` for a bare number into UE 5.7's `FValueOrBBKey_Float WaitTime`) instead of being ignored. The `behavior_tree.set_node_properties` handler now, when any key failed, rejects with `SendError("INVALID_PROPERTY", ..., ErrResult)` carrying a structured `droppedFields` array (`[{key, reason}]`) plus `partiallyApplied`, mirroring the board's validate-before-mutate / surface-the-drop convention (`DataTableAuthoringHandler.cpp` `droppedFields`, `WidgetSetHandler.cpp` `INVALID_PROPERTY`). The all-valid path is unchanged. The lower-priority ergonomic follow-on (making `FValueOrBBKey_Float` settable via bare-number routing or dotted-key resolution) is intentionally NOT done here — the bug was the silent success, which is now an error. Regression test: `PinWright.behavior_tree.set_node_properties.RejectsSilentlyDroppedWrites` in `Source/PinWright/Private/Tests/Gameplay/TestBehaviorTreeAttachSubnodes.cpp` creates a BT, adds a `Wait` task, calls `set_node_properties` with `{WaitTime:3.5, TotallyUnknownProp:1}` (both keys fail), and asserts the response is `bSuccess==false` with `ErrorCode=="INVALID_PROPERTY"` and a `droppedFields` list naming `TotallyUnknownProp` (always) and `WaitTime` (5.7+ only, gated on `UE_VERSION_NEWER_THAN_OR_EQUAL(5,7,0)`); it would fail (silent `bSuccess==true`) if the fix were reverted.
- `#1-initial-repro` `OPEN` reporter — Realism task building `/Game/AI/BT_GuardAI`; the silent no-op surfaced on the patrol Wait time. Replay-confirmed live against `mcp__editor-automation__call` on `/Game/McpOracle/BT_OracleSvcReplay`: `set_node_properties {properties:{WaitTime:3.5}}` (and `WaitTime:2`) returns `ok:true` while `ApplyJsonValueToProperty` returns false (`"Unsupported JSON type for struct property"`, `PropertyImport.cpp:876`) because UE 5.7 made `UBTTask_Wait::WaitTime` an `FValueOrBBKey_Float` struct; `set_node_properties {properties:{"WaitTime.DefaultValue":7}}` also returns `ok:true` while `FindPropertyCI` (`PropertyInspection.cpp:19`, no dotted resolution) returns null and the key is skipped (`BehaviorTreeHandler.cpp:233-237`). The handler discards both failure paths (the false return at `:245-248` and the unknown-key `continue`) and always `SendSuccess` (`:896-904`), so every-key-failed writes report success with the node unchanged. Neither `decompile` (omits WaitTime) nor `asset.dump` (top-level BT props only) can reveal the drop. Same misleading-success class as `B-data-table-row-values-silent-drop` / `B-configure-slot-behavior-ignores-behavior-and-tags`; distinct from `B-widget-set-slot-struct-fails` (which surfaces the struct failure as `INVALID_PROPERTY` rather than swallowing it). Temp asset deleted after replay.
