---
id: B-configure-slot-behavior-ignores-behavior-and-tags
title: "`ai.configure_slot_behavior` ignores `behaviorType` entirely and silently drops unregistered `activityTags` — success-with-no-effect"
status: IN-REVIEW
severity: High
category: bug
tags: [ai, smart-object, smart-object-definition, configure-slot-behavior, behavior-definition, activity-tags, gameplay-tags, silent-noop, success-no-effect]
---

# `ai.configure_slot_behavior` never attaches a behavior and silently drops unregistered tags

`ai.configure_slot_behavior` is documented as "Configure behavior for a Smart
Object slot" and advertises a `behaviorType` ("Type of behavior") param. In
practice the handler has two success-shaped no-op defects:

1. **`behaviorType` is read into a local and then never used.** Nothing maps it
   to a `FSmartObjectSlotDefinition::BehaviorDefinitions` entry, so the slot's
   behavior is never configured — the response always reports
   `behaviorCount: <existing count>` (0 on a fresh slot) and
   `"Slot behavior configured"` regardless of what `behaviorType` is passed.
   Since this is the **only** verb in the `ai` namespace that touches smart
   object slot behavior (no sibling RPC assigns a `BehaviorDefinition`), the
   titular capability — attaching behavior to a slot — is unreachable, yet every
   call reports success.

2. **`activityTags` are silently dropped for any tag not already in the
   gameplay-tag registry.** The handler resolves each tag with
   `FGameplayTag::RequestGameplayTag(FName(*TagStr), /*ErrorIfNotFound=*/false)`
   and only adds it when `Tag.IsValid()`. An unregistered tag yields an invalid
   `FGameplayTag` and is skipped — no error, no warning, no `droppedTags` field
   in the response. A caller passing a not-yet-registered activity tag (the
   natural first attempt) gets `"Slot behavior configured"` with the tag
   silently absent; they must independently know to `gameplay_tags.add` the tag
   first and re-run.

The success-shaped echo (`{slotIndex, behaviorCount, "Slot behavior
configured"}`) makes both no-ops indistinguishable from a real write. This is
the same stub shape already documented for the perception family in
[B-configure-sense-config-silent-noop](B-configure-sense-config-silent-noop.md)
(different methods, different code path — that one is `ai.configure_*_config`
sense config; this is `ai.configure_slot_behavior`).

## Root cause (`Handlers/AI/AIHandler.cpp`, `ai.configure_slot_behavior` ~1767–1848)

- `FString BehaviorType = Ctx.GetString(TEXT("behaviorType"), TEXT(""));`
  (~1782) is the only use of `behaviorType`. It is never read again — no
  `BehaviorDefinitions.Add(...)`, no behavior-definition construction. The
  response's `behaviorCount` is just `Slot.BehaviorDefinitions.Num()` (~1836),
  which the handler never mutates.
- Tags (~1815–1823):
  ```cpp
  FGameplayTag Tag = FGameplayTag::RequestGameplayTag(FName(*TagStr), false);
  if (Tag.IsValid())
  {
      Slot.ActivityTags.AddTag(Tag);
  }
  ```
  The `false` (`ErrorIfNotFound`) plus the `IsValid()` guard silently skips any
  unregistered tag with no caller-visible signal.

## Replay-confirmed repro (live editor, `mcp__editor-automation__call`)

On an existing definition `/Game/AI/SmartObjects/SOD_ParkBench` (2 slots):

1. `ai.configure_slot_behavior {definitionPath:"/Game/AI/SmartObjects/SOD_ParkBench", slotIndex:0, behaviorType:"Sit", activityTags:["AI.Activity.NeverRegisteredFuzzTag"], enabled:true}`
   → `{"slotIndex":0,"behaviorCount":0,"message":"Slot behavior configured"}`
   (success; `behaviorCount` stays 0 despite `behaviorType:"Sit"`).
2. Readback `property.get {objectPath:"/Game/AI/SmartObjects/SOD_ParkBench", propertyName:"Slots"}`
   → slot 0 has `"BehaviorDefinitions":[]` (behaviorType had no effect) and
   `"ActivityTags":{"GameplayTags":[{"TagName":"AI.Activity.Sit"}], ...}` — the
   unregistered `AI.Activity.NeverRegisteredFuzzTag` from step 1 is **absent**
   (silently dropped); only the previously-registered `AI.Activity.Sit` remains.

Control isolating the tag drop: the same call with a tag already added via
`gameplay_tags.add` lands the tag (the attempt's `AI.Activity.Sit` persisted
only after registering it and re-running), proving the registry-membership gate
is the cause, not a readback artifact.

## Impact

The discovery-schema-obvious path to author a smart object slot's interaction —
`create_smart_object_definition` → `add_smart_object_slot` →
`configure_slot_behavior {behaviorType, activityTags}` — yields a slot with no
behavior definition and (on first attempt) no activity tags, while reporting
success for every step. There is no JSON-RPC signal that `behaviorType` was
ignored or that a tag was dropped.

**Workaround:** (tags) `gameplay_tags.add` every activity tag first, then re-run
`configure_slot_behavior`, then verify via `property.get ... Slots`. (behavior)
none on the `ai` surface — `behaviorType` cannot currently produce a
`BehaviorDefinition`; the slot's behavior must be set by other means.

**Fix options (any of):**
1. **Implement `behaviorType`.** Map the string to the appropriate
   `USmartObjectBehaviorDefinition` subclass, `NewObject` it on the definition,
   and append to `Slot.BehaviorDefinitions` (replacing an existing one of the
   same class), then report the real `behaviorCount`. If only a fixed set of
   behavior types is supported, reject unknown values with
   `INVALID_PARAMS` listing the accepted types instead of silently ignoring.
2. **Surface dropped tags.** When `RequestGameplayTag` returns invalid, either
   auto-register the tag (mirroring `gameplay_tags.add`) or return the dropped
   tags in a `droppedTags` array / `INVALID_PARAMS` so the no-op is visible.
3. **Fail loud (interim).** Until `behaviorType` is implemented, omit it from the
   wiki or return `NOT_IMPLEMENTED` when it is supplied, so callers are not led
   to believe a behavior was attached.

## History
- `#1-initial-repro` `OPEN` reporter — Seed-mode fuzz around `ai.add_smart_object_component` (park-bench smart object task). Replay-confirmed live on `/Game/AI/SmartObjects/SOD_ParkBench`: `configure_slot_behavior {slotIndex:0, behaviorType:"Sit", activityTags:["AI.Activity.NeverRegisteredFuzzTag"], enabled:true}` returned `{"slotIndex":0,"behaviorCount":0,"message":"Slot behavior configured"}`, and `property.get ... Slots` showed slot 0 with `BehaviorDefinitions:[]` (behaviorType ignored) and the unregistered tag absent from `ActivityTags` (silently dropped). Root cause read from `AIHandler.cpp` ~1767–1848: `behaviorType` is read into a local and never used; tags resolved via `RequestGameplayTag(..., /*ErrorIfNotFound=*/false)` + `IsValid()` guard skip unregistered tags with no caller-visible signal. Distinct from `B-configure-sense-config-silent-noop` (perception `configure_*_config` family, different handlers). The only `ai`-namespace verb that touches smart object slot behavior, so `behaviorType` being ignored leaves the documented capability unreachable.
- `#2-implement-and-fail-loud` `IN-REVIEW` developer — Fixed both no-ops in `Handlers/AI/AIHandler.cpp` (`ai.configure_slot_behavior`, ~1805–1980) with a validate-before-mutate rewrite combining Fix Options 1+2+3. (1) `behaviorType` is now resolved to a concrete `USmartObjectBehaviorDefinition` subclass via reflection — `/Script/Module.Class` paths directly, otherwise a `TObjectIterator<UClass>` scan matching the exact name, the name with a `U` prefix, or a short framework alias expanding to `<Alias>SmartObjectBehaviorDefinition` (e.g. `Mass`, `GameplayInteraction`). A resolved class is `NewObject`'d with the Definition asset as outer (the array is `Instanced`), replaces any existing definition of the same class, is appended to `Slot.BehaviorDefinitions`, and the real `behaviorCount`/`behaviorAttached`/`behaviorClass` are reported. (2) A non-empty `behaviorType` that resolves to nothing instantiable (the base class is `Abstract` and every concrete subclass lives in an unlinked optional plugin — GameplayBehaviorSmartObjects/GameplayInteractions/MassSmartObjects) is rejected with `INVALID_PARAMS` plus an `availableBehaviorTypes` list, never silently ignored (fail-loud per Option 3). (3) `activityTags` are pre-resolved; any unregistered tag is collected and the call is rejected with `INVALID_PARAMS` carrying a `droppedTags` array instead of silently dropping it. Validation runs before any slot mutation so a rejected param leaves no partial write. Added include `UObject/UObjectIterator.h`. Files: `Source/EditorAutomationRpcGateway/Private/Handlers/AI/AIHandler.cpp`. Regression test: `EditorAutomationRpcGateway.ai.configure_slot_behavior.RejectsNoOpInputs` in `Source/EditorAutomationRpcGateway/Private/Tests/Gameplay/TestAIHandlers.cpp` creates a real `USmartObjectDefinition` + slot via the production handlers, then asserts both an unresolvable `behaviorType` and an unregistered `activityTag` are rejected with `INVALID_PARAMS` (tag surfaced in `droppedTags`) — it fails if either no-op is reverted (the old handler fake-succeeded both). Skips gracefully when SmartObjects headers are unavailable. Not compiled/run here (handled by a later phase).
