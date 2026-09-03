---
id: E-add-variable-wiki-class-prefix-rejected
title: "blueprint.add_variable's wiki documents `class:/Script/X.Y` for object refs, which the handler rejects — the accepted form is `object<T>`, and only the error message says so"
status: OPEN
severity: Low
category: ergonomic
tags: [blueprint, add_variable, variableType, docs, wiki, type-grammar]
encounters: 1
lastSeen: 2026-09-02T19:41:00Z
---

# The documented `class:` prefix is not an accepted `variableType`

## Repro

```
blueprint.add_variable {path:"/Game/FPS/UI/WBP_HUD", variableName:"VigMID",
  variableType:"class:/Script/Engine.MaterialInstanceDynamic", category:"HUD"}

-> [TYPE_NOT_FOUND] Could not resolve variableType 'class:/Script/Engine.MaterialInstanceDynamic'.
   Accepted forms: primitives (bool, int, int64, float, double, byte, string, name, text);
   builtin structs (vector, rotator, transform, FLinearColor); struct short names
   (EditorReplay or FEditorReplay); class short names (MyWidget or UMyWidget or MyWidget*);
   full paths (/Script/Module.Type or /Game/Path/BP_Asset);
   wrappers (array<T>, set<T>, map<K,V>, object<T>, struct<T>, enum<T>, class<T>,
   softobject<T>, softclass<T>, interface<T>)
```

`variableType:"object<MaterialInstanceDynamic>"` works immediately.

## Where the wiki misleads

`Saved/PinWright/wiki/blueprint.add_variable.md`, `variableType` parameter description:

> Type token: built-in primitive (float, int, bool, string, name, text, vector, rotator,
> transform), or **'class:/Script/X.Y' for object/class refs**, 'struct:/Game/...' for struct refs.
> Container wrappers: array\<T\>, set\<T\>, map\<K,V\> …

Three problems with that sentence: the `class:` and `struct:` colon-prefix forms do not exist in the
grammar the handler actually uses; the wrapper list stops at the three containers and omits
`object<T>`, `class<T>`, `enum<T>`, `softobject<T>`, `softclass<T>` and `interface<T>`, which are
the forms that do work; and `object<T>` — the one a caller needs for the commonest case, an object
reference — is never mentioned on the page at all. The page's own **Notes** section repeats the
same list ("Beyond primitives and `class:`/`struct:` refs…").

The runtime error message is excellent and complete. The page it points away from is the problem.

## What should happen

Replace the `class:`/`struct:` prefixes on the page with the real grammar — the same list the
error emits — and add one `object<T>` example next to the existing `set<name>` /
`map<string,int>` ones. `interface<T>` deserves a mention too: it works and it is how a widget
holds a Blueprint-Interface reference, which is not obvious.

**Workaround:** ignore the page, read the error, use `object<T>`.

severity rationale: impact=docs only, and the error message recovers you in one round trip x
reach=`add_variable` runs in nearly every authoring session and object refs are the common case
-> Low.

## History
- `#1-filed` `OPEN` reporter — Hit on EAContentExamples58 (UE 5.8) adding `MaterialInstanceDynamic` variables to `/Game/FPS/UI/WBP_HUD`. Followed the wiki's `class:/Script/Engine.MaterialInstanceDynamic` form and got `TYPE_NOT_FOUND`; the error's own "Accepted forms" list named `object<T>`, which worked first try. Also confirmed in the same session that `interface<BPI_HUDSource_C>` works and is likewise undocumented on the page. Not filed as a bug because the handler behaves sensibly and reports well; the defect is entirely in the page.
