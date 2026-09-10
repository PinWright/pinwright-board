---
id: B-layertag-declared-integer-blocks-tag-addressing
title: "`layerTag` is declared `integer` on all four `ui.activatable_*` verbs while the handler reads it as a gameplay-tag string, so every tag value is refused with PARAM_TYPE_MISMATCH before the handler runs — layer addressing is unreachable"
status: IN-REVIEW
severity: High
category: bug
tags: [ui, activatable_push, activatable_pop, get_active_widget, list_stack_widgets, common-ui, layer-tag, gameplay-tags, param-spec, declared-type, param-type-mismatch, regression, pie]
encounters: 1
lastSeen: 2026-09-10T00:00:00Z
---

# The declared type contradicts the parameter's own description, one line apart

Observed 2026-09-10 driving a PIE session through the MCP against UE 5.8, host project
`X:\src\unreal\unreal-fpv-new`, plugin at `d5dfb11b`. `ui.activatable_push`,
`ui.activatable_pop` and `ui.get_active_widget` all rejected
`{"layerTag":"UI.Layer.Menu"}` with `PARAM_TYPE_MISMATCH`. Only `host`+`stack`
addressing worked, which is the surface `F-activatable-push-by-layer-tag` shipped
`layerTag` to replace.

## Evidence

One shared macro declares the parameter for all four verbs:

`Source/PinWrightCommonUI/Private/Handlers/UI/UiActivatableStackHandler.cpp:120-124`

```cpp
#define ACTIVATABLE_TARGET_PARAMS \
    RPC_PARAM_OPT("host",  "string",  "..."), \
    RPC_PARAM_OPT("stack", "string",  "..."), \
    RPC_PARAM_OPT("layerTag", "integer", "CommonGame/Lyra UI layer gameplay tag (e.g. UI.Layer.Menu) ..."), \
    RPC_PARAM_OPT("playerIndex", "integer", "Local player index for layerTag resolution (default 0)")
```

Line `:123` is the defect — the type token and the description on the same line
disagree. The macro expands into `ui.activatable_push` (`:135`),
`ui.activatable_pop` (`:191`), `ui.list_stack_widgets` (`:247`, affected too and not
noticed in the session) and `ui.get_active_widget` (`:283`). `playerIndex` as
`integer` is correct.

The handler reads it as a string and there is no integer path anywhere:

- `UiActivatableStackHandler.cpp:88` — `const FString LayerTag = Ctx.GetString(TEXT("layerTag"));`
- `UiActivatableStackHandler.cpp:92,101` — non-empty string routes to `PinWrightUi::ResolveStackByLayerTagInPie(LayerTag, PlayerIndex, ...)`
- `Source/PinWrightCommonUI/Private/Handlers/UI/ActivatableLayerResolver.h:36-37` — signature takes `const FString& LayerTagStr`
- `Source/PinWrightCommonUI/Private/Handlers/UI/ActivatableLayerResolver.cpp:136` — `FGameplayTag::RequestGameplayTag(FName(*LayerTagStr), false)`, `:137-140` errors `LAYER_TAG_INVALID` when unregistered

The rejection happens in the dispatcher's declared-type gate, before the handler body:

- `Source/PinWright/Private/Dispatch/RpcDispatcher.cpp:218-260` — `CollectDeclaredTypesByWireName` (`:218`), per-key `PinWrightCheckDeclaredType` (`:231-232`), `SendError(ERR_PARAM_TYPE_MISMATCH, ...)` (`:255`), `return false` (`:259`)
- `Source/PinWright/Private/Handlers/ParamTypeCheck.h:349-357` — `integer` delegates to `TryParseStrictJsonInteger`; `"UI.Layer.Menu"` is not integral-parsable
- `Source/PinWright/Private/Handlers/ErrorCodes.h:1203` — `ERR_PARAM_TYPE_MISMATCH`

There is therefore **no value a caller can supply that works**. The lossless-string
coercion rule (`ParamTypeCheck.h:26`) lets `{"layerTag":"123"}` past the gate, where
it then dies with `LAYER_TAG_INVALID` — a second, different error that sends the
caller looking for a tag-registration problem instead of a schema one.

## Cause

A regression, not an original defect. `git log -S'"layerTag", "integer"'` gives
`f41e086d` ("Fix 38 High board tickets, wave 9", 2026-09-05), whose only change to
this file is the one-token flip from `"string"` to `"integer"`; its message records
"discrete index parameters (83 schema params now integer)". That sweep is
`B-discrete-index-params-truncate-fractions`'s fix, whose name-token heuristic
collected `layerTag` as a discrete index. The parameter was correct as `"string"`
when the feature shipped in `b02e4fc7`.

**No test caught it, and no test can catch it as written.**
`Source/PinWright/Private/Tests/TestUtils.h:231-246` — `InvokeHandlerWithCapture`
calls `Reg.Func(Ctx)` directly and never runs `ValidateHandlerParams`. So
`Tests/UI/TestUiActivatableLayerTag.cpp:123`, which sets `layerTag` as a string
field, passes green against a declaration the real dispatcher refuses. The declared-type
gate is untested for this verb family.

## Fix

One token at `UiActivatableStackHandler.cpp:123`: `"integer"` → `"string"`. No ratchet
test in the tree asserts that discrete-name params must be `integer`, so nothing blocks
the revert. The durable half is a test that exercises the declared-type gate rather than
the handler directly, so the next name-token sweep cannot silently unreachable a verb.

Worth re-auditing the other 82 params `f41e086d` converted for the same class of
false positive.

## Workaround

Address by `host`+`stack` as before the feature landed — which costs the ObjectIterator
discovery dance for the layout instance name and its child stack names, documented on
`F-activatable-push-by-layer-tag`.

## Related

- `F-activatable-push-by-layer-tag` (IN-REVIEW) — shipped `layerTag`; its verification cannot pass while this stands. Its `#2` note cites pre-split paths (`Source/PinWright/Private/Handlers/UI/...`), now `Source/PinWrightCommonUI/Private/Handlers/UI/...`.
- `B-discrete-index-params-truncate-fractions` (IN-REVIEW) — the sweep that caused this; claims semantic-name exceptions were restored, never names `layerTag`.
- `B-declared-param-guard-blind-spots` (IN-REVIEW) — lists the same verb family, but for declaration *coverage*, not type correctness.
- `B-niagara-set-parameter-type-name-rejects-its-own-type` (IN-REVIEW) — same defect shape on a different verb: schema token contradicts the value the verb documents.

## History
- `#1-tag-value-refused-by-declared-type` `OPEN` reporter — Filed from a PIE session on UE 5.8, host `X:\src\unreal\unreal-fpv-new`, plugin `d5dfb11b`. `{"layerTag":"UI.Layer.Menu"}` refused with `PARAM_TYPE_MISMATCH` on `ui.activatable_push` / `ui.activatable_pop` / `ui.get_active_widget`; source review found `ui.list_stack_widgets` equally affected through the same shared macro. Verified in source both sides: declaration `"integer"` at `UiActivatableStackHandler.cpp:123`, consumption as `FString` at `:88` into `ResolveStackByLayerTagInPie`, refusal at `RpcDispatcher.cpp:255` before the handler runs. Traced to `f41e086d`'s discrete-index sweep via `git log -S`; the param was `"string"` at `b02e4fc7`. Explained why the existing regression test passes anyway (`TestUtils.h:231-246` bypasses `ValidateHandlerParams`). Severity High: the schema admits no working value at all, the one that parses (`"123"`) fails later with a different code that misdirects diagnosis, and it silently killed a feature five days after it shipped. No fix attempted.
- `#2-declared-string-plus-gate-level-test` `IN-REVIEW` developer — One-token fix in `Source/PinWrightCommonUI/Private/Handlers/UI/UiActivatableStackHandler.cpp` (`ACTIVATABLE_TARGET_PARAMS`, the `layerTag` line): `"integer"` -> `"string"`, so the shared macro's declaration now matches the `Ctx.GetString(TEXT("layerTag"))` reader and the gameplay-tag value its own description documents. Fixes all four verbs the macro expands into (`ui.activatable_push` / `ui.activatable_pop` / `ui.list_stack_widgets` / `ui.get_active_widget`). Nothing else changed; `playerIndex` stays `integer`. Confirmed the discrete-name ratchet `PinWright.infra.dispatcher.ParamTypeGate.DiscreteReadersHaveIntegerDeclarations` cannot re-flip it: it only correlates names collected from `GetInt`/`GetIntOr`/`RequireInt` call sites, and `layerTag` has no integer reader. Durable half, both in `Source/PinWrightCommonUI/Private/Tests/UI/TestUiActivatableLayerTag.cpp` (next to the tests that could not see the defect): `PinWright.ui.activatable.LayerTagDeclaredAsString` asserts the registered `FParamSpec::Type` is `string` on all four verbs via `ParamSpecTestHelpers::FindParamSpec`; `PinWright.ui.activatable.LayerTagPassesDeclaredTypeGate` routes `{layerTag:"UI.Layer.Menu", widgetClass:"/Game/_Test/NoSuchClass"}` through a real `FRpcDispatcher::ProcessRequest` (`DispatcherTestHelpers::MakeDispatcher` + `Dispatch`) — the production path that runs `ValidateHandlerParams`, which `TestUtils.h`'s `InvokeHandlerWithCapture` bypasses — and asserts the answer is `CLASS_NOT_FOUND` from the handler body, not `PARAM_TYPE_MISMATCH` from the gate. Host-independent: `widgetClass` is resolved before any stack lookup, so no PIE, no CommonGame and no widget asset are needed. Counterfactual: restore `"integer"` and the first test reads `integer` on all four verbs while the second gets `PARAM_TYPE_MISMATCH` instead of `CLASS_NOT_FOUND` — both fail. No new harness helper was added: `Tests/TestUtils.h:200-213` already documents `Tests/Infra/DispatcherTestHelpers.h` as the route for gate-level questions and warns against a third invoke path, so a `ValidateAndInvoke` variant would duplicate it (`ValidateHandlerParams` itself is file-local to `RpcDispatcher.cpp` and cannot be called directly). `F-activatable-push-by-layer-tag` is UNBLOCKED: with `layerTag` declared `string`, tag addressing is reachable at the wire level and that ticket's verification can now run.
