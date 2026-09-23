---
id: B-create-widget-bp-abstract-parent
title: "widget.create_widget_blueprint refuses an Abstract C++ UUserWidget parent (CLASS_NOT_INSTANTIABLE), although the editor and blueprint.reparent accept it"
status: OPEN
severity: Medium
category: bug
tags: [widget, create-widget-blueprint, abstract, parent-class, bindwidget, layout-only-umg]
encounters: 1
lastSeen: 2026-09-23T18:30:00Z
---

# create_widget_blueprint refuses an Abstract C++ parent

`widget.create_widget_blueprint {name: W_AppSchoolNameLogin, folder: /App/App/UI/LobbyAndMenu/Popups, parentClass: /Script/App.AppSchoolNameLoginWidget}` returns `CLASS_NOT_INSTANTIABLE: ... cannot be used as a UUserWidget parent`. The parent is `UCLASS(Abstract, Blueprintable)`, the standard pattern for a C++ widget whose Blueprint subclass is layout-only (BindWidget). The UMG "Create Widget Blueprint" flow accepts such parents, and so does `blueprint.reparent`.

**Workaround:** create with the default `UserWidget` parent, `blueprint.reparent {newParentClass: /Script/App.AppSchoolNameLoginWidget, compile: false}`, then `widget.import_xml`.

**Fix:** allow `CLASS_Abstract` for Blueprintable parents (refuse only deprecated/superseded/non-UUserWidget), matching `FKismetEditorUtilities::CanCreateBlueprintOfClass`.

**Source (8748c637):** the refusal is deliberate code, not a resolution miss: `WidgetCreateHandler.cpp:131-141` rejects any `UUserWidget` subclass carrying `CLASS_Abstract | CLASS_Deprecated | CLASS_NewerVersionExists` (exempting only `UUserWidget` itself), and the generated page `Saved/PinWright/wiki/widget.create_widget_blueprint.md:26-27` documents that rule. It was introduced by the `B-widget-create-parent-fallback` (IN-REVIEW) fix, which added `CLASS_NOT_INSTANTIABLE`; that ticket's silent-fallback fix is sound, only the `CLASS_Abstract` term over-rejects. Fix the handler and the wiki paragraph together.

## History
- `#1-abstract-parent-refused` `OPEN` reporter — Hit twice (W_AppSchoolNameLogin, W_AppUserPanel), UE 5.8, host `X:\src\unreal\unreal-fpv-new`, plugin `8748c637`.
