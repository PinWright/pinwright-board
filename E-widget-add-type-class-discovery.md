---
id: E-widget-add-type-class-discovery
title: "`widget.add` `type` — no wiki path to discover non-primitive / CommonUI container class names (e.g. `CommonActivatableWidgetStack`)"
status: OPEN
severity: Low
category: ergonomic
tags: [widget, add, docs, common-ui, class-discovery]
encounters: 1
costly: 1
lastSeen: 2026-06-30T22:53:48.7445084+03:00
---

# `widget.add` `type` — no wiki path to discover the addable class-name string for a CommonUI container

`widget.add`'s `type` parameter accepts any placeable widget class (engine short
name OR `_C` asset path — see `E-class-name-format-inconsistency`, DONE), but the
doc only enumerates **primitive** examples: `type` (string, required) "Widget
class name (e.g. TextBlock, Button, CanvasPanel, HorizontalBox)". No `widget*.md`
page lists the higher-level / plugin container classes — `CommonActivatableWidgetStack`,
`CommonActivatableWidgetQueue`, etc. — nor points to a discovery path for the
exact class-name string when an agent wants a CommonUI activatable stack.

Because the wiki dead-ends at primitives, the agent left the wiki for **UE engine
source** to confirm the addable type string: it ran a whole-engine ripgrep
(`Ripgrep search timed out after 20 seconds`), two narrower greps
(`No matches found` / `No files found`), a Glob, and finally **Read**
`C:/UE_5.7/.../CommonActivatableWidgetContainer.h` "to confirm the exact class
name and that it's a UWidget placeable" before concluding the `widget.add` type
string is `CommonActivatableWidgetStack`. The subsequent
`widget.add {type:"CommonActivatableWidgetStack"}` succeeded **first try**
(`insertIndex:0`), so the spelunking was pure discovery overhead — the only
missing piece was the class-name string's discoverability.

This is a discovery/process gap, not a tool bug — every MCP call in the task
succeeded first try (19 RPCs, zero retries, zero python fallback).

## The discovery RPC already exists — the doc just doesn't point to it

`system.inspect.search_classes` (shipped by `F-search-api-native-uclasses`, DONE)
was built **explicitly** for this use case — its filing names "What Slate / UMG
widget classes exist? — for `widget.import_xml` or `widget.add` authoring, caller
needs to know that `UTextBlock`, `UButton`, `UCanvasPanel` … exist." A call like
`system.inspect.search_classes {query:"activatable stack", parentClass:"Widget"}`
returns the native `/Script/…` container classes with no engine-source dive. The
agent did not use it — because nothing on the `widget.add` page links the two.

## What it should do (downstream, wiki only)

Improve `docs/wiki-src/widget.md` (the `widget.add` overlay material):
(1) beyond the primitive examples, list a few supported higher-level / CommonUI
container types (`CommonActivatableWidgetStack`, `CommonActivatableWidgetQueue`)
and note they derive from `UCommonActivatableWidgetContainerBase : UWidget` and are
therefore placeable; (2) point `type`-discovery at `system.inspect.search_classes`
(`parentClass:"Widget"` + a keyword) as the in-wiki way to enumerate addable
widget class-name strings, so authors don't leave the wiki for engine headers.

severity rationale: impact=docs/discoverability (correct guess, lost only
discovery time — one ripgrep timed out 20s + a header Read; no failed editor
action) × reach=rare (discovering a *non-primitive* `widget.add` type is a
sub-path; the primitives the doc already lists cover the common case) -> Low.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit (CallAnalyzer trace finding) of a CommonUI modal-layer-host task (`focus: ui.activatable_push`). To pick the `widget.add` type string for the activatable stack, the agent could not resolve it from the wiki and went to engine source: whole-engine ripgrep (timed out after 20s), two narrower greps (no matches), a Glob, then Read of `CommonActivatableWidgetContainer.h`, before `widget.add {type:"CommonActivatableWidgetStack"}` succeeded first try (`insertIndex:0`). `widget.add`'s `type` doc only lists primitive examples (TextBlock/Button/CanvasPanel/HorizontalBox); no `widget*.md` page mentions CommonUI containers or links to `system.inspect.search_classes` (DONE, `F-search-api-native-uclasses` — built for exactly this `widget.add`-authoring discovery use case). Distinct from `E-widget-add-then-rename-discoverability` (naming widgets at add time, not type discovery) and `E-class-name-format-inconsistency` (DONE — which *format*s `type` accepts, not how to *find* the class string). Fix is a `docs/wiki-src/widget.md` overlay only.
