---
id: B-widget-apply-style-silent-noop
title: "`widget.apply_style` is a no-op stub — returns success:true + 'Style binding created' but writes no WidgetStyle and creates no binding"
status: IN-REVIEW
severity: High
category: bug
tags: [widget, apply-style, create-style, silent-noop, fake-success, umg]
---

# `widget.apply_style` reports success but applies nothing

`widget.apply_style` is a no-op stub. It loads the blueprint, finds the target
widget, looks up an `FProperty` by `styleName` on the generated class, calls
`MarkBlueprintAsStructurallyModified`, then unconditionally returns
`success:true` with the note **"Style binding created. Actual style application
requires runtime binding setup."** It **never writes the target widget's
`WidgetStyle` property** and **never creates a binding**. The looked-up
`FProperty* StyleProp` is reported as `styleFound` but is otherwise unused — so
the documented task "apply this style to this button" silently does nothing.

Handler source confirms the no-op (`Handlers/UI/WidgetStyleHandler.cpp:184-231`):
after `FindWidgetByName` + `FindPropertyByName`, the body builds a result object
and returns — there is no mutation of `TargetWidget`, no `Modify()` on it, no
binding creation. The wiki page (`widget.apply_style`) advertises it plainly as
"Apply a style to a widget" with no caveat that it is a stub.

Compounding the silent no-op, the create→apply round-trip the docs imply can
never set `styleFound:true` from a freshly-created style:
`widget.create_style styleType=Button` names the member variable
`<styleName>_ButtonStyle` (e.g. `PrimaryButtonStyle_ButtonStyle`), while
`widget.apply_style` looks up the bare `<styleName>` (`PrimaryButtonStyle`) on
`GeneratedClass`. So the natural sequence returns `styleFound:false`. Even when
the call is fed the exact created variable name, it still returns
`styleFound:false` (the just-added member variable is not yet on the compiled
`GeneratedClass`) — and either way nothing is applied.

This is a silent success-with-no-effect tool bug (same class as
`B-spawn-category-silent-noop-fake-existsafter`,
`B-input-trigger-modifier-stub-silent-success`,
`B-create-pose-library-noop-fake-success`): the response misreports a "binding
created" that does not exist, and `describe`/`export_xml` readback shows no
change, so an agent that trusts the success cannot tell the style was never
applied.

## What it should do

Either:
- Actually apply the style — write the named style struct onto the target
  widget's `WidgetStyle` (for a `Button`, copy the `FButtonStyle` value into
  `UButton::WidgetStyle`), wrapped in the existing `FScopedTransaction`, mark
  the BP modified, so `widget.describe`/`export_xml` reflect the override; or
- Create a real property binding from the widget's style property to the style
  member variable; or
- If neither is feasible in-editor, return a clean `NOT_SUPPORTED`/`NOT_IMPLEMENTED`
  error (with the runtime-binding guidance) instead of `success:true` with a
  "Style binding created" note. Do not fake-success a method that applies
  nothing — and drop or fix the `styleName` vs `<styleName>_ButtonStyle`
  naming mismatch between `create_style` and `apply_style` so the lookup can
  succeed.

## Verbatim repro (replay-confirmed via `mcp__editor-automation__call`)

Setup:
- `widget.create_widget_blueprint` `{name: WBP_ApplyStyleReplay, folder: /Game/UI}` -> ok
- `widget.add` `{widgetPath: /Game/UI/WBP_ApplyStyleReplay, type: Button, name: MyButton, parentName: RootCanvas}` -> ok
- `widget.create_style` `{widgetPath: /Game/UI/WBP_ApplyStyleReplay, styleName: PrimaryButtonStyle, styleType: Button}`
  -> `{success:true, createdVariables:["PrimaryButtonStyle_ButtonStyle","PrimaryButtonStyle_NormalColor","PrimaryButtonStyle_HoveredColor","PrimaryButtonStyle_PressedColor"], variableCount:4}`
  (note: the variable is named `PrimaryButtonStyle_ButtonStyle`, not `PrimaryButtonStyle`)

The no-op, both with the bare name and the exact created variable name:
- `widget.apply_style` `{widgetPath: /Game/UI/WBP_ApplyStyleReplay, widgetName: MyButton, styleName: PrimaryButtonStyle}`
  -> `{success:true, styleName:"PrimaryButtonStyle", styleFound:false, note:"Style binding created. Actual style application requires runtime binding setup."}`
- `widget.apply_style` `{widgetPath: /Game/UI/WBP_ApplyStyleReplay, widgetName: MyButton, styleName: PrimaryButtonStyle_ButtonStyle}`
  -> `{success:true, styleName:"PrimaryButtonStyle_ButtonStyle", styleFound:false, note:"Style binding created. Actual style application requires runtime binding setup."}`

Readback confirming nothing was applied:
- `widget.describe` `{widgetPath: /Game/UI/WBP_ApplyStyleReplay, widgetName: MyButton, include_bindings: true, include_defaults: true}`
  -> `MyButton.props.WidgetStyle` is the engine default `FButtonStyle` (default RoundedBox Normal/Hovered/Pressed brushes, default cyan foregrounds) — no override from the created style. `bindingTargets` lists only the three default UserWidget events (`Event Pre Construct`, `Event Construct`, `Event Tick`) — no property binding to any `*_ButtonStyle` variable.

## Workaround

Set the button's `WidgetStyle` directly via `widget.set` with a fully-spelled
`FButtonStyle` ExportText literal (same per-widget repetition the caller wanted
`create_style`/`apply_style` to avoid), or wire the style member variable to the
widget at runtime via a Blueprint property binding (BPIR/graph authoring). Do
not trust `widget.apply_style`'s success — verify with `widget.describe`.

## History
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed via `mcp__editor-automation__call` on a fresh `/Game/UI/WBP_ApplyStyleReplay` (Button `MyButton` under `RootCanvas`, `create_style PrimaryButtonStyle styleType=Button`). `widget.apply_style` returned `success:true, styleFound:false, note:"Style binding created. Actual style application requires runtime binding setup."` with both the bare `PrimaryButtonStyle` and the exact created variable `PrimaryButtonStyle_ButtonStyle`. `widget.describe` (include_bindings, include_defaults) showed `MyButton.WidgetStyle` unchanged from the engine default and `bindingTargets` containing only the three default UserWidget events — no override, no binding. Confirmed in handler source `WidgetStyleHandler.cpp:184-231`: the body finds the widget + looks up `FProperty` by `styleName` but never mutates the widget or creates a binding before returning fake success; `create_style` (line 87) names the var `<styleName>_ButtonStyle` while `apply_style` (line 217) looks up bare `<styleName>`, guaranteeing `styleFound:false` for the natural round-trip. Dedup: ripgrep over the board for `apply.style|styleFound|create_style|WidgetStyle|PrimaryButtonStyle|requires runtime binding` found no existing ticket about this verb — nearest neighbors `B-widget-set-partial-application` (widget.set rollback), `E-widget-anim-loop-speed-phantom-authoring-verbs` (animation loop/speed verbs), `F-image-brush-from-texture` (set_image_brush ergonomics) are different methods. Classified TOOL BUG (silent success-with-no-effect), same class as `B-spawn-category-silent-noop-fake-existsafter` / `B-input-trigger-modifier-stub-silent-success` / `B-create-pose-library-noop-fake-success`.
- `#2-fail-loud-not-implemented` `IN-REVIEW` developer — Applied the accepted honest-failure pattern (same as `B-input-trigger-modifier-stub-silent-success` Option 2 / `B-texture-create-placeholder-fake-success`): replaced the `widget.apply_style` no-op success body in `Handlers/UI/WidgetStyleHandler.cpp` (was lines 184-231) with a `SendError("NOT_IMPLEMENTED", …)` after the existing param-presence check. No more `success:true` + `styleFound`/"Style binding created" lie, and the discarded `FProperty` lookup / `FScopedTransaction` / `MarkBlueprintAsStructurallyModified` no-op were removed. The error message and the `REGISTER_RPC_HANDLER` summary now point callers at the real paths (set `WidgetStyle` directly via `widget.set`, or wire a runtime Blueprint property binding) — the `create_style` `<styleName>_ButtonStyle` vs bare-`<styleName>` mismatch is moot now that apply_style does no lookup. Chose fail-loud over actually applying the struct (Option 1) because real style application across each widget type's distinct `WidgetStyle` property + resolving the BP variable's default struct value is a large, risky change out of proportion to a leaf-handler bug, and the board precedent for this exact class is the minimum-safe honest-failure swap. Regression test added: `Private/Tests/Widget/TestWidgetApplyStyleNotImplemented.cpp` — `FWidgetApplyStyleReturnsNotImplementedTest` builds a transient WBP with a real `MyButton` (so the handler passes every prerequisite the old stub checked) and asserts via `InvokeHandlerWithCapture` that the production handler returns `bSuccess == false` and `ErrorCode == "NOT_IMPLEMENTED"` (fails if the silent-success branch is restored); `FWidgetApplyStyleRejectsMissingNameTest` guards that a missing `widgetName` still returns `MISSING_PARAMETER`, not `NOT_IMPLEMENTED`. Not compiled/run here (later phase verifies). Lens-validated by all three (correctness/adversarial/board-historian) as a real, currently-present defect; no duplicate (the docs companion `E-widget-style-workflow-wiki-advertises-stub` self-carves as distinct and survives this fail-loud fix).
- `#3-method-removed-in-cull` `IN-REVIEW` reporter — Note (superseded by removal): the `widget.apply_style` NOT_IMPLEMENTED tombstone was removed entirely in the RPC cull recorded in [`E-rpc-cull-151-record`](E-rpc-cull-151-record.md), coupled with the `widget.create_style` -> `blueprint.add_variable` supersession (the two were the surviving halves of the same import-era create/apply pair; per the supersession audit, apply_style's guard stub lost its remaining context once create_style went). The `#2` fail-loud fix is therefore moot — the method no longer exists — and its regression test (`Tests/Widget/TestWidgetApplyStyleNotImplemented.cpp`) was deleted with the handler. Recorded for traceability.
