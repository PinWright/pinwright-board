---
id: B-widget-set-slot-declared-integer
title: "`widget.set`'s `slot` parameter is declared `integer` while the handler reads it with `GetObjectField`, so every slot payload is refused with PARAM_TYPE_MISMATCH before the handler runs and the one shape the gate admits returns a silent success that sets nothing"
status: IN-REVIEW
severity: High
category: bug
tags: [widget, widget-set, umg, slot, canvas-panel-slot, param-spec, declared-type, param-type-mismatch, silent-false-success, regression]
encounters: 4
lastSeen: 2026-09-24T01:25:00Z
---

# The declared type contradicts the parameter's own description, one line below a correct one

Observed 2026-09-17 authoring `W_HUD_Sumo_Game` through the MCP against UE 5.8, host project
`X:\src\unreal\unreal-fpv`, plugin at `c86eb581`. Every `widget.set` call carrying a `slot`
object was refused with `PARAM_TYPE_MISMATCH` before the handler body ran. The whole slot
half of the verb is unreachable: there is no JSON shape a caller can send that reaches it.

## Evidence

The declaration and the reader disagree inside one file:

`Source/PinWright/Private/Handlers/UI/WidgetSetHandler.cpp:121-122`

```cpp
        RPC_PARAM_OPT("properties", "object", "Key-value pairs of widget property names to values"),
        RPC_PARAM_OPT("slot", "integer", "Key-value pairs of slot property names to values")
```

Line `:122` is the defect. Its own description says "key-value pairs", the line above declares
the identically shaped `properties` as `object`, and the reader one screen down takes an object:

- `WidgetSetHandler.cpp:141` - `TSharedPtr<FJsonObject> SlotObj = GetObjectField(Payload, TEXT("slot"));`
- `WidgetSetHandler.cpp:143` - `bHasSlotProperties = SlotObj.IsValid() && SlotObj->Values.Num() > 0`
- `Handlers/UI/WidgetAuthoringUtils.cpp:43-50` - `GetObjectField` returns the field only when
  `HasTypedField<EJson::Object>` holds, and `nullptr` otherwise. There is no integer path anywhere
  in the file.

The refusal happens in the dispatcher's declared-type gate, before the handler body:

- `Source/PinWright/Private/Dispatch/RpcDispatcher.cpp:215` - `CollectDeclaredTypesByWireName`
- `RpcDispatcher.cpp:229` - per-key `PinWrightCheckDeclaredType`
- `RpcDispatcher.cpp:252` - `SendError(ErrorCodes::ERR_PARAM_TYPE_MISMATCH, ...)`, `:259` `return false`
- `Source/PinWright/Private/Handlers/ParamTypeCheck.h:349-358` - `integer` delegates to
  `TryParseStrictJsonInteger`; an `EJson::Object` is never integral-parsable. The header's own
  acceptance table (`:26`) lists `object` as refused by `integer`.

**The one value that clears the gate is worse than the refusal.** `integer` accepts a lossless
integral *string* (`ParamTypeCheck.h:26`), so `{"slot": "5"}` passes validation, `GetObjectField`
then returns `nullptr` because the field is not an object, `bHasSlotProperties` is false, and with
no `properties` alongside it the handler takes `WidgetSetHandler.cpp:159-166` and answers
`{"success": true, "propertiesSet": 0}`. A caller who satisfies the schema is told the call
succeeded while nothing was written - the silent false-success class, not merely a blocked verb.

**The generated wiki inherits the wrong token and contradicts itself.**
`Saved/PinWright/wiki/widget.set.md:18` renders `` `slot` (`integer`, optional): Key-value pairs of
slot property names to values ``, while the overlay prose merged onto the same page
(`docs/wiki-src/widget.md:488-510`) documents three JSON *object* / string shapes for slot values.
An agent reading the page has no way to tell which half is true, and the half it can act on is wrong.

## Cause

Almost certainly the discrete-index name-token sweep, the same false positive as
`B-layertag-declared-integer-blocks-tag-addressing`: `slot` is a literal member of the sweep's
`DiscreteTokens` set (`Source/PinWright/Private/Tests/Infra/TestParamTypeGate.cpp:376-379`,
alongside `index`, `count`, `lod`, `level`, `layer`, `frame`), and the exception list at `:393-405`
excludes `path` / `name` / `rate` / `time` / `duration` / `distance` / `tag` / `lodType` but nothing
that would spare a bare `slot` naming a property bag. `git log -S'"slot", "integer"'` resolves only
to `8748c637` ("PinWright 0.8.0: first open-source release"), the squash, so the pre-squash commit
is not recoverable in this repo - the attribution is by mechanism, not by revision.

**No test catches it.** `widget.set` has no gate-level test: the widget tests invoke handlers
through `Tests/TestUtils.h`'s `InvokeHandlerWithCapture`, which calls `Reg.Func(Ctx)` directly and
never runs `ValidateHandlerParams`, so a slot payload passes in-process against a declaration the
real dispatcher refuses. This is the identical blind spot recorded on the `layerTag` ticket.

## Fix

One token at `WidgetSetHandler.cpp:122`: `"integer"` -> `"object"`, then regenerate the wiki
(it regenerates at editor startup; do not hand-edit `Saved/PinWright/wiki/widget.set.md`).

The discrete-name ratchet `PinWright.infra.dispatcher.ParamTypeGate.DiscreteReadersHaveIntegerDeclarations`
cannot re-flip it: it only correlates names collected from `GetInt` / `GetIntOr` / `RequireInt` call
sites (`TestParamTypeGate.cpp:534-568`), and `widget.set`'s `slot` has no integer reader - the repo
has zero `GetInt*(TEXT("slot"))` sites. So nothing in the tree blocks the change.

The durable half is a test that routes a slot payload through a real
`FRpcDispatcher::ProcessRequest` (`Tests/Infra/DispatcherTestHelpers.h`) rather than through
`InvokeHandlerWithCapture`, asserting the answer comes from the handler body and not from the gate.
Worth adding a declaration assertion too: `slot` declared `object`, `properties` declared `object`.

Consider re-auditing the rest of the sweep for parameters whose name carries a discrete token but
whose value is a bag or a bundle rather than an ordinal.

## Workaround

Write the slot through the reflection path instead of the `slot` parameter:

- `property.set` with `objectPath` = the widget and `propertyName` = `Slot.LayoutData` (dotted
  nested path, `Handlers/Utility/UtilityPropertyHandler.cpp:1146-1152`). `UWidget::Slot` is a
  reflected `UPROPERTY` (`Runtime/UMG/Public/Components/Widget.h:263-264`), so the dotted resolver
  reaches the `CanvasPanelSlot` fields. This is what was used on `W_HUD_Sumo_Game`.
- `widget.set` with the dotted key inside `properties` (declared `object`, so it clears the gate):
  `{"properties": {"Slot.LayoutData": ...}}` routes through `ResolveNestedPropertyPath`
  (`WidgetSetHandler.cpp:253-257`). Source-verified, not exercised live in this session.
- `widget.import_xml` with `mode: add` also carries `Slot.LayoutData`, per
  `B-widget-set-slot-struct-fails`.

## Related

- `B-widget-set-slot-struct-fails` (DONE) - shipped and live-verified the struct-valued slot shapes
  that this declaration now makes unreachable. Its `#4` tester evidence (`slot:{"LayoutData":...}`
  returning `propertiesSet:1`) cannot be reproduced against the current declaration, so that
  verification has been silently invalidated.
- `B-layertag-declared-integer-blocks-tag-addressing` (IN-REVIEW) - same defect shape, same sweep,
  same untested gate; its `#2` note documents why the ratchet does not block the revert.
- `B-discrete-index-params-truncate-fractions` (IN-REVIEW) - the sweep whose name-token heuristic
  collects `slot`; claims semantic-name exceptions were restored, never names this parameter.
- `B-param-type-never-validated` - the gate itself, working as designed here.
- `E-widget-canvas-slot-no-padding-docs`, `E-widget-describe-slot-truncation` - adjacent slot
  ergonomics, unaffected.

## History
- `#1-slot-object-refused-by-declared-type` `OPEN` reporter - Filed while editing `W_HUD_Sumo_Game` on UE 5.8, host `X:\src\unreal\unreal-fpv`, plugin `c86eb581`. Every `widget.set` carrying a `slot` object was refused with `PARAM_TYPE_MISMATCH`. Verified in source both sides: declaration `"integer"` at `WidgetSetHandler.cpp:122`, consumption as an object at `:141` via `GetObjectField` (`WidgetAuthoringUtils.cpp:43-50`, object-typed field or `nullptr`), refusal at `RpcDispatcher.cpp:252` before the handler runs, `integer` refusing `EJson::Object` at `ParamTypeCheck.h:349-358`. Also traced the one admitted shape: `{"slot":"5"}` clears the lossless-string coercion, yields `SlotObj == nullptr`, and returns `{"success":true,"propertiesSet":0}` at `:159-166` - a success payload for a write that never happened. Generated page `Saved/PinWright/wiki/widget.set.md:18` publishes the wrong type beside overlay prose (`docs/wiki-src/widget.md:488-510`) documenting object shapes. Attributed to the discrete-index sweep by mechanism (`slot` is in `TestParamTypeGate.cpp:376-379`'s `DiscreteTokens`, not in the `:393-405` exception list); `git log -S` reaches only the 0.8.0 squash, so no pre-squash revision is citable. Severity High: the schema admits no working value, the single value it does admit is a silent false-success rather than an error, the verb is a primary every-session UMG authoring surface, and it invalidates `B-widget-set-slot-struct-fails`'s verified fix. Workaround used: `property.set` on `Slot.LayoutData` through the dotted resolver. No fix attempted.
- `#2-reproduced-slot-autosize` `OPEN` reporter - Reproduced on UE 5.8, host `X:\src\unreal\unreal-fpv-new`, plugin `8748c637`: `widget.set {widgetName: W_AppUserPanel, slot: {bAutoSize: true}}` on W_LyraFrontEnd returned `PARAM_TYPE_MISMATCH: 'slot' is declared integer and was sent as object`; the `slotProperties` spelling is rejected as `UNKNOWN_PARAMS`. Workaround: `python.execute` on `/App/App/UI/LobbyAndMenu/W_LyraFrontEnd.W_LyraFrontEnd:WidgetTree.W_AppUserPanel`, `get_editor_property('slot').set_auto_size(True)`, then `blueprint.compile` and `asset.save`.
- `#3-reproduced-padding-with-key-workaround` `OPEN` reporter - Reproduced on UE 5.8, host `X:\src\unreal\unreal-fpv-new`, plugin `8748c637`: `widget.set {widgetName: LessonBlock, slot: {Padding: {...}}}` on W_AppUserPanel refused with `PARAM_TYPE_MISMATCH: 'slot' is declared integer and was sent as object`; `slotProperties` is rejected as UNKNOWN_PARAMS. Working workaround for other callers: put slot fields in `properties` with a `Slot.` prefix (`properties: {"Slot.Padding": {...}, "Slot.LayoutData": {...}, "Slot.HorizontalAlignment": "HAlign_Fill"}`); all applied and survived compile and save. Cheap (one retry).
- `#4-reproduced-canvas-layoutdata` `OPEN` reporter - Reproduced on UE 5.8, host `X:\src\unreal\unreal-fpv-new`, plugin `8748c637`: `widget.set {widgetName: SB_SchoolLogin, slot: {LayoutData: {...}}}` on a CanvasPanel child refused with PARAM_TYPE_MISMATCH (`slot` declared integer). The `properties: {"Slot.LayoutData": {...}}` key workaround applied it (`propertiesSet: 2`). Cheap: one retry.
- `#5-slot-declared-object-and-non-object-refused` `IN-REVIEW` developer - Baseline on the unmodified build (live `mcp__pinwright__call`, probe path `/Game/_Test/W_FixerHProbeDoesNotExist`): `widget.set slot:{bAutoSize:true}` -> `PARAM_TYPE_MISMATCH` `'slot' is declared integer and was sent as object`; `slot:"5"` cleared the gate and reached the body (`NOT_FOUND` from `LoadWidgetBlueprint`), i.e. the path that answers `propertiesSet:0` on a real widget. Fix: `WidgetSetHandler.cpp` declares `slot` as `object`; the body now refuses a present-but-non-object `properties`/`slot` with `INVALID_PARAMETER` before `GetObjectField` can drop it (the `object` gate admits arrays by design, `ParamTypeCheck.h`, so `slot:[]` was the remaining silent-success shape). Sibling audit (every `RPC_PARAM_*(..., "integer")` whose file reads the same key through a non-integer reader): three more instances of the same sweep false positive, all boolean flags whose `true` the integer gate refused - `widget.duplicate.copySlot`, `widget.describe.include_slot`, `widget.export_xml.omit_slot_chain` (baseline: `copySlot:true` / `include_slot:false` -> `PARAM_TYPE_MISMATCH ... declared integer and was sent as boolean`); all redeclared `boolean`. No other integer declaration in `Handlers/` has a non-integer reader. Overlay `docs/wiki-src/widget.md` `### widget.set` named a non-existent `slotProperties` param and showed `LayoutData` under `properties`; corrected to `slot`. Tests (new `Tests/Widget/TestWidgetSetSlotDispatch.cpp`, routed through a real `FRpcDispatcher::ProcessRequest`): `PinWright.widget.set.SlotObjectReachesHandlerThroughDispatcher` (slot object -> success, `propertiesSet:1`, `CanvasPanelSlot.bAutoSize` written; `slot:"5"` -> `PARAM_TYPE_MISMATCH`; `slot:[]` / `properties:[]` -> `INVALID_PARAMETER`, nothing written), `PinWright.widget.SlotNamedParamsDeclaredByShape` (declared types of the five params). Run `PinWright.widget.set+PinWright.widget.SlotNamedParamsDeclaredByShape+PinWright.infra.dispatcher.ParamTypeGate`: `check_suite_log` COMPLETED_CLEAN, 18/18. Post-fix live: `slot` object and all three booleans now reach their handler bodies (`NOT_FOUND` for the probe path), `slot:"5"` -> `'slot' is declared object and was sent as string`; regenerated `Saved/PinWright/wiki/widget.set.md:18` reads `` `slot` (`object`, optional) ``. Not changed: an entirely empty call (no `properties`, no `slot`) still answers `propertiesSet:0`, which is an honest count of a request that asked for nothing.
- `#6-literal-code-keeps-file-non-adopting` `IN-REVIEW` developer - `#5`'s `ErrorCodes::ERR_INVALID_PARAMETER` made `WidgetSetHandler.cpp` a registry-adopting file (any `ErrorCodes::ERR_` citation), which then failed `PinWright.core.error_codes.RegistryAdoptingFilesUseConstantsOnly` on the file's 14 pre-existing hand-spelled codes. Switched the new refusal to `TEXT("INVALID_PARAMETER")` (registered, matches the file's existing style) and dropped the added `ErrorCodes.h` include; the full literal-to-constant conversion of the file is left as a separate cleanup. Run `PinWright.core.error_codes+PinWright.widget.set+PinWright.widget.SlotNamedParamsDeclaredByShape+PinWright.infra.dispatcher.ParamTypeGate`: `check_suite_log` COMPLETED_CLEAN, 22/22, including `RegistryAdoptingFilesUseConstantsOnly` and `AllEmittedCodesAreRegistered`. Plugin commit `3a4523f8`.
