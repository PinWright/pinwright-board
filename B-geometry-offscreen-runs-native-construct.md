---
id: B-geometry-offscreen-runs-native-construct
title: "`resolve_geometry` offscreen tier builds a transient instance of the asked Widget Blueprint, so a C++ parent's NativeConstruct runs with no game world and asserts, killing the editor"
status: IN-REVIEW
severity: Critical
category: bug
tags: [widget, export-xml, widget-describe, resolve-geometry, offscreen, native-construct, crash, commonui, gameplay-message-router]
encounters: 1
costly: 1
lastSeen: 2026-09-23T19:02:50Z
---

# The offscreen tier constructs the widget for real, and real widgets have constructors that need a game

`widget.export_xml {widgetPath: /App/App/UI/LobbyAndMenu/Elements/W_AppUserPanel, compact: true,
resolve_geometry: {instance_name: "W_AppUserPanel"}}` was sent while PIE was running on the front-end map
with that panel on screen. The editor died with `appError: Assertion failed: Router
[GameplayMessageSubsystem.cpp:47]`.

Stack (crash `UECC-Windows-B666C3AC4CE8C841536F70885B9616AD_0001`, host `X:\src\unreal\unreal-fpv-new`,
UE 5.8, plugin `8748c637`):

```
UGameplayMessageSubsystem::Get()                     GameplayMessageSubsystem.cpp:47
UAppUserPanel::NativeConstruct()                     (project C++ parent of the WBP)
UCommonUserWidget::OnWidgetRebuilt()
UWidget::TakeWidget()
ResolveViaOffscreen()                                WidgetGeometryResolver.cpp:654
FWidgetGeometryResolver::Resolve()                   WidgetGeometryResolver.cpp:773
AutoHandler_330_()                                   WidgetXmlExportHandler.cpp:211
```

Two defects stack:

1. **The live tier did not claim a widget that was on screen.** The panel is a child of `W_LyraFrontEnd`,
   not added to the viewport itself, so `IsInViewport()` is false and the live tier skips it even with an
   exact `instance_name`. The request fell through to offscreen.
2. **The offscreen tier calls `TakeWidget()` on a fresh `UUserWidget` of the asked class.** That runs
   `NativeConstruct` of any C++ parent. Project widgets routinely reach game-instance subsystems there
   (`UGameplayMessageSubsystem::Get(this)` asserts with no router). The existing guard
   (`ContainsExtensionPointWidget`, see `B-widget-describe-offscreen-crash-extension-point`) only skips
   `UUIExtensionPointWidget`; any widget whose native construct touches the world crashes the same way.

**Impact:** editor crash, unsaved work lost, PIE session lost. Any C++-backed UMG widget in a Lyra/CommonUI
project is a candidate.

**Workaround:** use `widget.export_xml {capture_source: "live", include_geometry: true, instance_name: <UMG
root>}` and read `<Geometry>` from the live Slate tree. Never pass `resolve_geometry` for a widget with a
C++ parent.

**Fix (proposed):** (a) let the live tier match nested instances (walk live `UUserWidget`s of the class in
PIE worlds, not only viewport roots); (b) refuse the offscreen tier with a typed error when the generated
class has a native (non-`UUserWidget`/`UCommonUserWidget`) ancestor that overrides `NativeConstruct`, or
build the transient instance with a PIE/game world context instead of none.

## History
- `#1-offscreen-native-construct-crash` `OPEN` reporter - Filed from a UMG layout pass on `W_AppUserPanel` (UE 5.8, host `X:\src\unreal\unreal-fpv-new`, plugin `8748c637`). `widget.export_xml` with `resolve_geometry: {instance_name}` during PIE crashed the editor in `UAppUserPanel::NativeConstruct -> UGameplayMessageSubsystem::Get` via `ResolveViaOffscreen` (crash UECC-...B666C3AC..._0001). Costly: editor restart and PIE session lost. Live snapshot (`capture_source: live`) gave the needed geometry safely.
- `#2-offscreen-builds-design-time-instance` `IN-REVIEW` developer — Fixed defect 2 at the root: `ResolveViaOffscreen` (`Handlers/UI/WidgetGeometryResolver.cpp`) no longer calls `CreateWidget`. It builds the transient instance the way the UMG Designer builds its preview (`FWidgetBlueprintEditorUtils::CreateUserWidgetFromBlueprint`): `NewObject(World, GenClass, RF_Transient)`, `SetDesignerFlags(EWidgetDesignFlags::Designing)` before and after `Initialize()`. With `IsDesignTime()` true the engine skips `NativeOnInitialized` (`UserWidget.cpp:173`) and `NativePreConstruct`/`NativeConstruct` in `OnWidgetRebuilt` (`UserWidget.cpp:1228`), so no C++ parent's runtime lifecycle runs without a game instance, whatever that parent reaches. Abstract/deprecated/superseded classes still refuse with `CREATE_WIDGET_FAILED` (the flags `CreateWidget` refused; `NewObject` would assert on them). All callers (`widget.export_xml`, `widget.describe`) route through this one function. Test: `PinWright.widget_geometry.offscreen.DoesNotRunNativeConstruct` (synthetic native parent `UTestWidgetConstructProbe` that counts its hooks; WBP parented to it, never opened or on screen, asserts offscreen tier served, NativeConstruct/NativeOnInitialized calls == 0, root `IsDesignTime()`, child geometry ok). Baseline with the one-hunk fix reverted: that test FAILED `Expected 'NativeConstruct never ran on the transient instance' to be 0, but it was 1` (4/5 pass). With the fix: `PinWright.widget_geometry+PinWright.widget.export_xml+PinWright.widget.describe` = 32/32, `check_suite_log` COMPLETED_CLEAN. Live, Linux host, no PIE: `widget.export_xml {widgetPath: <the C++-parented panel from #1>, resolve_geometry: {instance_name}}` returned `Geom.source=offscreen`, 84 ok / 4 error (the 4 are an inactive WidgetSwitcher slot, `GEOMETRY_NOT_ARRANGED`), no assert, editor alive. Not done: defect 1 (live tier matching nested, non-viewport-root instances in PIE) is untouched - a nested on-screen panel still falls through to offscreen, which now returns Designer-equivalent layout instead of crashing; split it into its own ticket if live geometry of nested instances is needed. The `ContainsExtensionPointWidget` guard is now probably redundant (design-time `UUIExtensionPointWidget::RebuildWidget` takes its preview branch) but was left in place, unverified. Docs: `docs/widget-geometry-resolver.md` Tier C, `docs/wiki-src/widget.md` resolver tier list. Plugin commit `9582f071`.
