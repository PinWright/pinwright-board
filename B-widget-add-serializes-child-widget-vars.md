---
id: B-widget-add-serializes-child-widget-vars
title: "widget.add / widget.duplicate+replace_class of a Blueprint UserWidget child serialize the child's own widget-variable references (50 inner widgets) into the parent's designer template"
status: DONE
severity: Medium
category: bug
tags: [widget, widget.add, widget.replace_class, widget.duplicate, userwidget, template, serialization, asset-bloat, silent-wrong-data]
encounters: 1
lastSeen: 2026-09-25T09:00:00Z
---

# A placed child UserWidget carries its inner widget references in the parent asset

## Symptom

Adding `W_LobbyLoginButton_C` (a Blueprint UserWidget with ~50 widget variables) to `W_LyraFrontEnd` with
`widget.add` (and, the same, `widget.duplicate` of another child then `widget.replace_class` to that class, then
`blueprint.compile` + `asset.save`) wrote every one of the child's widget-variable properties onto the template
instance in the parent: the dump's `tree.xml` lists `SBW_MyRecords="{...,_kind=/Script/UMG.SizeBox}"`,
`LoginButton="{ButtonText=...}"`, `img_HeaderBG="{Brush=...}"` and 47 more on the `<W_LobbyLoginButton>` element,
and `strings` on the saved `.uasset` finds those names (absent from the same placement made in the UMG designer:
`origin/dev`'s asset has none of them). The asset grew by ~3.6 KB. Runtime rebinds those variables from the
class's own tree, so behaviour is unchanged, but the parent now carries a stale copy of the child's layout and
its dump no longer matches a designer-made placement.

## Repro

UE 5.8 Linux, host `/sdb-disk/src/unreal/unreal-fpv`, plugin `ba115afb`: `widget.add {widgetPath:<parent>,
type:"<BP UserWidget>_C", name:..., parentName:<canvas>}`, compile, save, `asset.dump`; look at the new
element in `tree.xml`.

## Workaround

`widget.set properties:{<each listed variable>: null}` on the placed child, then compile and save: the dump and
the asset lose the copies.

## History

- `#1-child-vars-serialized` `OPEN` reporter - Filed while restoring a legacy login button next to a newer panel
  in a school-computer compatibility fix (plugin `ba115afb`).
- `#2-clear-template-bindings` `IN-REVIEW` developer - Root cause: `UWidgetTree::ConstructWidget` on a
  `UUserWidget` runs `Initialize()`, which binds every widget property of the class to the instance's own
  transient runtime tree; those values sat on the template in the parent's designer tree and got saved.
  Every authoring path constructs through `WidgetAuthoringHelpers::ConstructWidgetForAuthoring`
  (`widget.add`, `widget.duplicate`, `widget.replace_class`, `widget.import_xml`), which now nulls any
  object property of the placed UserWidget whose value lives in a UserWidget-owned `UWidgetTree`;
  `CopyMatchingProperties` (duplicate / replace_class `preserveProperties`) does the same after copying,
  so a source template bound in-session cannot hand the copy references into its tree. Files:
  `Source/PinWright/Private/Handlers/UI/WidgetAuthoringUtils.cpp`,
  `Source/PinWright/Private/Tests/WidgetXml/TestWidgetAddUserWidgetHandler.cpp`, `CHANGELOG.md`. Tests
  (scratch parent + compiled row WBP with a variable `RowRoot`):
  `PinWright.widget.add.UserWidgetOmitsChildWidgetVars` (also re-checks after compiling the parent),
  `PinWright.widget.duplicate.UserWidgetOmitsChildWidgetVars` (source placed through the engine path, so
  it is bound), `PinWright.widget.replace_class.UserWidgetOmitsChildWidgetVars`.
- `#3-fixture-guid` `IN-REVIEW` developer - Full suite: duplicate and replace_class tests passed; the add test
  failed only on the engine's `ValidateAndFixUpVariableGuids` ensure ("Widget [RootCanvas] was added but did
  not get a GUID") because the fixture built the parent's root with raw `ConstructWidget`, and once `widget.add`
  registered its widget the GUID map was no longer empty. Fixture now registers the root with
  `OnVariableAdded` (5.6+ guard) as the editor does; product code unchanged.
- `#4-fixture-parent-class` `IN-REVIEW` developer - Fix round 2: the add test then failed on "[Compiler]
  Blueprint ... has missing or NULL parent class": the fixture's target came from the on-disk-shaped stub
  (`MakeOnDiskShapedWidgetBlueprint`, no ParentClass), which cannot be compiled. Target is now made through
  `FKismetEditorUtilities::CreateBlueprint` with a `UUserWidget` parent (shared `MakeUserWidgetBlueprint`,
  also used by the row fixture); product code unchanged.
- `#5-fixture-loaded-shape` `IN-REVIEW` developer - Fix round 3: with a factory-made target all three tests lost
  the bug's precondition. Finding: `UBaseWidgetBlueprint`'s constructor flags its `WidgetTree`
  `RF_ArchetypeObject`, so a UserWidget placed in a freshly created WBP `IsTemplate()` and
  `InitializeWidgetStatic` returns before duplicating its tree or binding variables; `UWidgetBlueprint::PostLoad`
  clears that flag, so the bloat happens on every WBP loaded from disk (the reporter's case), not on one
  created in the same session. Fixture now: factory-made target with the flag cleared (loaded shape), GUID
  registration for every raw-constructed widget (RootCanvas, SourceRow, SwapMe), and explicit preconditions
  (row class property and class widget-tree archetype hold RowRoot; the engine-constructed source binds it).
- `#6-verified-linux` `DONE` tester — PinWright `10212ee4` (on origin/master `6283b63b`), UE 5.8 Linux Vulkan. Runs: w23-final = offscreen full suite, no DISPLAY, 5568/5568 passed; w23-xfinal = DISPLAY=:0 offscreen, drive.os_input+click_occlusion+os_gesture+input, 31/31; w23-vis = DISPLAY=:0 windowed drive.input.ModifierChord, 2/2; Python = Content/Python/tests, 427 OK / 5 skipped (all skips Windows-only or an absent local script). Passed in w23-final: `PinWright.widget.add.UserWidgetOmitsChildWidgetVars` (also re-checked after compiling the parent), `PinWright.widget.duplicate.UserWidgetOmitsChildWidgetVars` and `PinWright.widget.replace_class.UserWidgetOmitsChildWidgetVars`. After #5 the fixture has the loaded-from-disk shape and asserts the bug's precondition (the child class binds `RowRoot`) before checking that the placed template holds no reference into the child's widget tree. Fixed for `widget.add`, `duplicate` and `replace_class`; `import_xml` shares the helper but has no test. Limit: the #1 PDS asset dump (`W_LobbyLoginButton` in `W_LyraFrontEnd`) was not regenerated.
