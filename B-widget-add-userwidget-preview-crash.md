---
id: B-widget-add-userwidget-preview-crash
title: "`widget.add` custom UserWidget rows can crash UMG designer preview"
status: DONE
severity: High
category: bug
tags: [widget-add, userwidget, designer-preview, crash, bpir]
---

# `widget.add` custom UserWidget rows can crash UMG designer preview

Adding custom `UUserWidget`-derived row widgets to an existing Widget Blueprint via `widget.add` can leave the UMG designer preview in a crashable in-memory state. The live repro added multiple `W_HotkeyInfo-Image_C` children under a new pause-menu hotkey layer in `/App/App/UI/LobbyAndMenu/HUD/W_HUD_DroneGameMenu`. Each `widget.add` / `widget.set` call returned success, and a follow-up `widget.describe` showed the newly added subtree in memory.

The next `blueprint.compile_bpir` call failed, then the editor crashed while the UMG designer attempted to rebuild the preview widget tree. The crash stack is not a normal compile failure; it runs through `UUserWidget::RebuildWidget`, `UWidget::TakeWidget`, panel slot rebuilds, and `SDesignerView::UpdatePreviewWidget`, with the top frame in `FObjectPtr::GetHandle()` / `TObjectPtr::operator bool()`.

Observed call sequence:

1. `widget.add` `HorizontalBox` named `HorizontalBox-Montage` under `Overlay_2`.
2. `widget.add` two `VerticalBox` columns under `HorizontalBox-Montage`.
3. `widget.add` ten `W_HotkeyInfo-Image_C` children under those columns.
4. `widget.set` row text/image/slot properties for each child.
5. `widget.describe` for `HorizontalBox-Montage` returned the expected in-memory tree.
6. `blueprint.compile_bpir` on `W_HUD_DroneGameMenu` failed while resolving `$HorizontalBox-Montage`.
7. Editor crashed shortly after during designer preview rebuild.

Relevant log lines before the crash:

```text
LogBpirValueResolver: Error: PreEmitDollarVar: VariableGet node for '$HorizontalBox-Montage' has no non-exec output pin
LogBpirValueResolver: Error: ResolveDollarVar: '$HorizontalBox-Montage' not found -- was PreEmitDollarVar called?
LogBpirCompiler: Error: ResolveTargetClass: could not resolve target '$HorizontalBox-Montage'
LogOutputDevice: Error: Ensure condition failed: CurrentTransaction [EditorTransaction.cpp] [Line: 1798]
LogOutputDevice: Error: Ensure condition failed: !Pin->LinkedTo.Contains(ReferencingPin) [EdGraphPin.cpp] [Line: 1846]
LogEditorAutomationRpcGatewaySubsystem: Warning: Automation request failed (COMPILE_FAILED): ...
LogHttpConnection: Error: errors.com.epicgames.httpserver.socket_send_failure
```

User-provided crash stack excerpt:

```text
FObjectPtr::GetHandle()
ObjectPtr_Private::IsObjectPtrNull(const FObjectPtr &)
TObjectPtr::operator bool()
UUserWidget::RebuildWidget()
UCommonActivatableWidget::RebuildWidget()
UWidget::TakeWidget()
UOverlaySlot::BuildSlot()
UOverlay::RebuildWidget()
...
SDesignerView::UpdatePreviewWidget(bool)
SDesignerView::Tick(...)
FSlateApplication::TickAndDrawWidgets(float)
FEngineLoop::Tick()
GuardedMain(...)
```

Expected behavior:

- `widget.add` should either create a fully valid `UWidget` instance for custom Widget Blueprint classes, or reject unsupported user-widget classes before mutating the tree.
- A failed `blueprint.compile_bpir` must not leave the edited Widget Blueprint or designer preview in a crashable transient state.
- If the newly added widget variable is not visible to BPIR yet, the error should explain the required compile/refresh step without destabilizing the editor.

**Workaround:** Avoid `widget.add` for custom `UUserWidget` child classes in complex existing widgets until this is fixed. Prefer duplicating an existing row manually in the editor, or use a safer purpose-built duplicate operation once available.

## History

- `#1-initial-repro` `OPEN` reporter -- During montage pause hotkey authoring, `widget.add` successfully inserted `W_HotkeyInfo-Image_C` rows under a new `HorizontalBox-Montage` subtree, then `blueprint.compile_bpir` failed resolving `$HorizontalBox-Montage`; the editor crashed in UMG designer preview rebuild (`UUserWidget::RebuildWidget` / `SDesignerView::UpdatePreviewWidget`).
- `#2-userwidget-authoring-construction` `IN-REVIEW` developer — Changed widget authoring construction to route custom UUserWidget classes through the initialized ConstructWidget<UUserWidget> path in WidgetAuthoringUtils, reused it from widget.add/XML import/replace paths, and added FWidgetAddUserWidgetInitializesNestedWidgetTest to guard the crash-prone raw-construction path.
- `#3-verified-temp-userwidget` `DONE` tester — Verified: `widget.add` added custom `W_HotkeyInfo-Image_C` as `NestedHotkeyRow` under `/Game/App/UI/Test/W_McpReviewTemp_20260429` and `blueprint.compile saveAfterCompile:false` returned `compiled:true`, `status:"UpToDate"`, `errors:[]`, `warnings:[]`; no designer-preview crash occurred.
