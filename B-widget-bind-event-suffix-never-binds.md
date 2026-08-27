---
id: B-widget-bind-event-suffix-never-binds
title: "`widget.bind` cannot bind any real UMG delegate; the only name it accepts is the one that never resolves"
status: IN-REVIEW
severity: High
category: bug
tags: [umg, widget-bind, silent-false-success]
---

# `widget.bind` cannot bind any real UMG delegate

Reproduced live on a scratch `UBorder`, UE 5.8, plugin `d195a55d`:

```
propertyName="OnMouseButtonDownEvent"  -> BINDING_FAILED
                                          "Cannot derive return type for property"
propertyName="OnMouseButtonDown"       -> success:true, bindingType:"event", created:true
```

The **rejected** name is the real property (`UBorder::OnMouseButtonDownEvent`,
`Components/Border.h:96-97`). The **accepted** name does not exist on the class, so the binding
resolves to nothing at runtime.

`UWidgetBlueprintGeneratedClass::InitializeBindingsStatic`
(`Runtime/UMG/Private/WidgetBlueprintGeneratedClass.cpp:157-161`) looks up `PropertyName + "Delegate"`,
then `PropertyName` verbatim. It never appends `Event`. Neither candidate exists on `UBorder`.

No diagnostic: `FDelegateEditorBinding::IsBindingValid`
(`Editor/UMGEditor/Private/WidgetBlueprint.cpp:530-533`) falls into an empty
`else { // Bindable Property Removed }` and returns false with no MessageLog entry.
`SanitizeBindings` (`WidgetBlueprintCompiler.cpp:711-747`) prunes only on a missing *widget*, never a
missing property, so the dead entry persists.

**Read-back cannot detect it.** `widget.export_xml` emits `Bind.OnMouseButtonDown="PwProbeStripped"`,
indistinguishable from a real binding. Verified live.

## Mechanism

`Handlers/UI/WidgetBindHandler.cpp:71-72` decides property-vs-event with `bIsEvent = !Prop` after
`FindPropertyByName`. A delegate **is** a property, so that lookup always succeeds and `bIsEvent` is
never true for a real delegate. Control reaches `BuildReturnPinTypeForProperty`
(`WidgetBindingUtils.cpp:125-208`), which has no `FDelegateProperty` branch, leaving `TypeHint` empty —
producing PinWright's own error string at `:83-84`.

The event branch is therefore reachable **only** for names absent from the class, i.e. only for names
guaranteed not to bind. `UButton::OnClicked` fails identically, so the RPC's own documented example
(`propertyName` described as "Text, Visibility, **OnClicked**", `WidgetBindHandler.cpp:41`) cannot work.

`UpsertWidgetBinding` (`WidgetBindingUtils.cpp:358-427`) writes the three string fields with no
validation against the lookup UMG performs, and never sets `Kind`, so event bindings inherit the
struct default `EBindingKind::Property`.

## Suggested fix

Resolve the name the way UMG will before writing —
`FindFProperty<FDelegateProperty>(Class, Name + "Delegate")` then `(Class, Name)` — and reject when
neither resolves. Add an `FDelegateProperty` branch to `BuildReturnPinTypeForProperty` so the real
`...Event` name is accepted. The qualifying condition is `GetName().EndsWith(TEXT("Event"))` **or**
`meta=(IsBindableEvent)` (`UMGDetailCustomizations.cpp:113`) — `UMenuAnchor::OnGetUserMenuContentEvent`
has no meta; `UComboBoxKey::OnGenerateContentWidget` has the meta and no suffix.

## Coverage gap

`Tests/Media/TestUIHandlers.cpp:475-487` asserts only that the handler is registered. No test writes a
binding and asserts it resolves. There is no `widget.bind` wiki page; `wiki-src/widget.md:470-487`
documents only the different, verify-only `widget.bind_event`.

## History

- `#1-reported-with-live-repro` `OPEN` reporter — Confirmed at runtime on `d195a55d` / UE 5.8: real property name rejected, suffix-stripped name accepted and persisted, `export_xml` renders it as genuine. Engine lookup order and silent-drop path verified in engine source.
- `#2-resolve-names-the-way-umg-does` `IN-REVIEW` developer — `widget.bind` now resolves `propertyName` through UMG's own two lookups (`<Name>Delegate`, then the name verbatim when it is a bindable event per `IsBindableEvent`) instead of `bIsEvent = !FindPropertyByName`; the handler function graph is generated from the delegate's `SignatureFunction` (and flagged `FUNC_BlueprintPure` for property bindings), the record is written with `Kind=Function` plus a member guid, and the write is gated on the engine's own `FDelegateEditorBinding::IsBindingValid` so no dead binding is persisted. Unresolvable names are refused with `WIDGET_BINDING_NAME_UNRESOLVED` (payload carries `bindableProperties` / `bindableEvents`, message names the near-miss spelling) and multicast events with `WIDGET_BINDING_IS_MULTICAST_EVENT` steering to `blueprint.compile_bpir`. Files: `Handlers/UI/WidgetBindHandler.cpp`, `Handlers/UI/WidgetBindingUtils.{h,cpp}`. Tests added: `PinWright.widget.bind.BindableNamesResolve`, `PinWright.widget.bind.UnresolvableNamesRejected` (`Tests/Widget/TestWidgetBindResolution.cpp`).
