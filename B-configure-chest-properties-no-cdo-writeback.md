---
id: B-configure-chest-properties-no-cdo-writeback
title: "interaction.configure_chest_properties reports locked/openAngle/openTime as configured but never writes them to the CDO (echo-only; values read back as defaults)"
status: IN-REVIEW
severity: High
category: bug
tags: [interaction, configure-chest-properties, configure-interaction-trace, configure-interaction-widget, cdo, silent-noop, success-no-effect, blueprint]
encounters: 3
lastSeen: 2026-07-02T09:14:38.1349323+03:00
---

# `interaction.configure_chest_properties` echoes `locked`/`openAngle`/`openTime` as configured but never writes them to the CDO

`interaction.configure_chest_properties` returns a fully successful response that
echoes the supplied `locked` / `openAngle` / `openTime` (and `lootTablePath`)
alongside `configured:true`, strongly implying the chest's gameplay values were
applied. They are **not**. The handler adds the member variables
(`bIsLocked`, `bIsOpen`, `LidOpenAngle`, `OpenTime`, `LootTable`) to the
Blueprint but **never writes the supplied values onto the class default object
(CDO)** — so after compile every configured value reads back as the type's zero
default (`bIsLocked=false`, `LidOpenAngle=0`, `OpenTime=0`). The response numbers
are pure echo of the input args.

This is **silent success-with-no-effect** on the method's only normal path: the
caller asks for "chest starts locked, lid opens 95° over 0.5s", the RPC says
`{"locked":true,"openAngle":95,"openTime":0.5,"configured":true}`, and the asset
that lands on disk has a lid angle of 0 and an unlocked chest. A "configure then
read back to confirm" workflow is misled — the configure looks applied, but the
CDO holds defaults.

The member VARIABLES themselves are created correctly (an agent's
`blueprint.inspect` and a `property.get` both find `bIsLocked`/`LidOpenAngle`
etc. post-compile) — the defect is specifically that their configured DEFAULT
VALUES are dropped. `lootTablePath`, when supplied, is likewise echoed but never
written to the `LootTable` soft-object var.

## Source confirmation (ground truth)

`Plugins/PinWright/Source/PinWright/Private/Handlers/Interaction/InteractionHandler.cpp`,
`interaction.configure_chest_properties` (L699-802). After adding the five
member variables the handler jumps straight to building the echo response —
there is **no CDO-write block at all**:

```cpp
// L787-800 — Result is built purely from the input args; no CDO is ever touched
TSharedPtr<FJsonObject> Result = MakeShareable(new FJsonObject());
Result->SetBoolField(TEXT("locked"), Locked);
Result->SetNumberField(TEXT("openAngle"), OpenAngle);
Result->SetNumberField(TEXT("openTime"), OpenTime);
if (!LootTablePath.IsEmpty())
{
    Result->SetStringField(TEXT("lootTablePath"), LootTablePath);
}
Result->SetBoolField(TEXT("configured"), true);
Result->SetStringField(TEXT("chestPath"), ChestPath);
```

Contrast the sibling `interaction.configure_door_properties` (L568-598), which
DOES contain a CDO-write block (`Blueprint->GeneratedClass->GetDefaultObject()`
→ `FindPropertyByName` → `ApplyJsonValueToProperty` for `OpenAngle`/`OpenTime`/
`bIsLocked`). The chest handler simply omits that block. (`configure_switch_properties`
has the same omission, but the chest is what this task exercised.)

Note (avoids a misleading fix): even the door's CDO-write block is fresh-BP
fragile — see repro step 4 below — because it writes to the CDO immediately
after `AddMemberVariable`, before the just-added var exists on the recompiled
generated class, so `FindPropertyByName` returns null and the write is skipped.
The correct fix should not merely copy the door block; it should set the value
durably on the variable description (assign `FBPVariableDescription.DefaultValue`
before the compile, the engine-native `FBlueprintEditorUtils::AddMemberVariable(BP,
Name, Type, DefaultValue)` path) so the single compile bakes it onto the CDO —
the same mechanism landed for `B-add-variable-default-value-ignored` #3.

## Repro (replayed live against `mcp__pinwright__call`)

1. `blueprint.create {name:"BP_ChestReplayTest", parentClass:"Actor", savePath:"/Game/OracleReplay", waitForCompletion:true}` → `{"assetPath":"/Game/OracleReplay/BP_ChestReplayTest","saved":true,"existsAfter":true}`.
2. `interaction.configure_chest_properties {chestPath:"/Game/OracleReplay/BP_ChestReplayTest", locked:true, openAngle:95, openTime:0.5}` → `{"locked":true,"openAngle":95,"openTime":0.5,"configured":true,"chestPath":"/Game/OracleReplay/BP_ChestReplayTest"}` (success; echoes the inputs — looks applied).
3. `blueprint.compile {path:"/Game/OracleReplay/BP_ChestReplayTest"}` → `{"compiled":true,"status":"UpToDate","errors":[],"warnings":[]}` (clean compile, vars now real on the CDO).
4. `property.get {objectPath:"/Game/OracleReplay/BP_ChestReplayTest", propertyName:"bIsLocked", includeDefault:true}` → `{"propertyName":"bIsLocked","value":false,...,"defaultSource":"class_cdo","defaultValue":false}` — **CDO is `false`, not the requested `true`**.
   `property.get {... propertyName:"LidOpenAngle", includeDefault:true}` → `{"value":0,...,"defaultValue":0}` — **CDO lid angle is `0`, not `95`**.
   (Control done same session on `BP_DoorReplayTest` via `configure_door_properties {locked:true, openAngle:95}` + compile: `bIsLocked`→`false`, `OpenAngle`→`0` as well — the door's CDO-write block is itself fresh-BP-fragile, so it cannot be cited as "the one that works"; the durable fix above is required for both.)

## Fix (proposed)

In `configure_chest_properties`, set each configured value durably on the new
`FBPVariableDescription.DefaultValue` before the compile (engine-native
`AddMemberVariable(BP, Name, Type, DefaultValueString)`), so the single compile
bakes `bIsLocked`/`LidOpenAngle`/`OpenTime`/`LootTable` onto the CDO and they
survive recompiles. Echo only the values that actually landed (read them back
off the CDO), or emit a `warning` when a value cannot be coerced — never a bare
`configured:true` that mirrors un-applied input. Apply the same to
`configure_switch_properties` and harden `configure_door_properties`'
post-compile-timing gap.

severity rationale: impact=silent false-success (caller trusts `locked:true`/`openAngle:95`/`openTime:0.5` echoed as `configured:true`; the saved asset holds defaults) × reach=every-session (this is the method's sole normal path) -> High.

## History
- `#1-initial-repro` `OPEN` reporter — Replayed live against `mcp__pinwright__call` on a fresh `BP_ChestReplayTest` (Actor): `configure_chest_properties {locked:true, openAngle:95, openTime:0.5}` returned `{"locked":true,"openAngle":95,"openTime":0.5,"configured":true}`; after `blueprint.compile` (UpToDate, 0 errors) `property.get {propertyName:"bIsLocked", includeDefault:true}` read `value:false`/`defaultValue:false` and `LidOpenAngle` read `value:0`/`defaultValue:0` — the echoed config was silently dropped, the CDO holds type defaults. Source (`InteractionHandler.cpp` L699-802) confirms the chest handler adds the five member vars then builds the response from inputs with no CDO-write block, unlike `configure_door_properties` (L568-598) which has one. Related but distinct: `B-add-variable-default-value-ignored` (generic `blueprint.add_variable` `defaultValue` param dropped — different method, different param shape) and `B-interaction-on-actor-configure-echo-noop` (the on-actor `configure_*_on_actor` echo-noops — different methods, world-actor-tag mechanism, already IN-REVIEW). Not a dup of either.
- `#2-liveness` `OPEN` reporter — still reproduces at HEAD (fresh `BP_ChestOracleReplay2`: `configure_chest_properties {locked:true, openAngle:88, openTime:1.25}` echoed `configured:true`; after compile `property.get` read `bIsLocked=false`, `LidOpenAngle=0`).
- `#3-additional-sibling-verbs` `OPEN` reporter — Struggle-audit of the `BP_TreasureChest` build (namespace interaction, 29 RPCs, outcome tool_bug). The SAME echo-`configured:true`-but-no-CDO-writeback defect also affects the sibling Blueprint-target verbs `interaction.configure_interaction_trace` and `interaction.configure_interaction_widget`: `configure_interaction_trace {traceType:"sphere",traceDistance:200,traceRadius:100}` returned `configured:true` yet the compiled CDO held `TraceDistance:0` (default); `configure_interaction_widget {showOnHover:true,showPromptText:true,promptTextFormat:"Press E to open"}` returned `configured:true` yet the CDO held `bShowOnHover:false` and `PromptTextFormat:""`. The agent had to persist TraceDistance/TraceType/bShowOnHover/bShowPromptText/PromptTextFormat via `blueprint.set_default` — the identical fallback used for the chest values. **Scope note:** the durable-`FBPVariableDescription.DefaultValue` fix MUST extend to these two verbs (same `InteractionHandler.cpp`); do not close this ticket until their CDO writeback is verified, not just chest/switch/door. (The malformed `InteractionWidgetClass` var and the signature-less dispatchers that `configure_interaction_widget` / `add_interaction_events` ALSO emit are separate defects — filed as `B-configure-interaction-widget-invalid-widget-class` and `B-add-interaction-events-delegates-no-signature`.)
- `#4-cdo-writeback-fix` `IN-REVIEW` developer — Fixed the whole no-CDO-writeback family in `InteractionHandler.cpp`. Added two file-local helpers: `SetOrAddMemberVariableDefault` bakes each configured value onto the member variable's `FBPVariableDescription.DefaultValue` before a single in-handler `FKismetEditorUtilities::CompileBlueprint` (the durable engine-native `AddMemberVariable(BP,Name,Type,DefaultValue)` mechanism, same as `B-add-variable-default-value-ignored` #3 — fresh-BP-safe and survives recompiles), and `EchoCompiledDefault` reads each value back off the compiled CDO for the response so it reports what actually landed instead of mirroring the request. Applied to all five Blueprint-asset configure verbs: `configure_chest_properties` (bIsLocked/LidOpenAngle/OpenTime/LootTable), `configure_switch_properties` (SwitchType/bCanToggle/ResetTime), `configure_door_properties` (removed its fresh-BP-fragile pre-compile direct-CDO write, replaced with the durable path), and per #3 the two sibling verbs `configure_interaction_trace` (TraceDistance/TraceType) and `configure_interaction_widget` (bShowOnHover/bShowPromptText/PromptTextFormat). `InteractionWidgetClass` population and the dispatcher-signature issues are intentionally left to their own tickets (`B-configure-interaction-widget-invalid-widget-class`, `B-add-interaction-events-delegates-no-signature`). File: `Plugins/PinWright/Source/PinWright/Private/Handlers/Interaction/InteractionHandler.cpp`. Regression tests (`Private/Tests/Gameplay/TestInteractionHandlers.cpp`): `PinWright.interaction.configure_chest_properties.DefaultsLandOnCdo` — fresh in-code transient Actor BP, `configure_chest_properties {locked:true,openAngle:95,openTime:0.5}`, asserts the compiled CDO holds bIsLocked=true/LidOpenAngle=95/OpenTime=0.5 (pre-fix false/0/0) and the response echoes them; `PinWright.interaction.configure_interaction_widget.DefaultsLandOnCdo` — asserts the CDO holds bShowOnHover=true and PromptTextFormat="Press E to open" (pre-fix false/""). Reverting the `NewVar.DefaultValue` bake makes both read the zero default and fail. Not compiled/tested here (later phase).
