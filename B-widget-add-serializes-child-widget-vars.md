---
id: B-widget-add-serializes-child-widget-vars
title: "widget.add / widget.duplicate+replace_class of a Blueprint UserWidget child serialize the child's own widget-variable references (50 inner widgets) into the parent's designer template"
status: OPEN
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
