---
id: B-widget-resourceobject-path-prefix-inconsistent
title: "tree.xml Image ResourceObject paths use mixed prefix formats"
status: DONE
severity: Low
category: bug
tags: [tree-xml, widget, schema]
---

# tree.xml Image ResourceObject paths use mixed formats

Some Image widgets emit `ResourceObject="/Script/Engine.Texture2D'/Game/...'"` (fully qualified with class prefix); others emit a bare `/Game/Foo.Foo` path or a `Texture2D'/Game/...'` form without the `/Script/Engine.` prefix. Mixed formats within a single asset.

## Sample

- `App/Blueprints/UI/W_Drone/tree.xml` — mixed formats across multiple Image widgets

## Fix sketch

Normalize to one canonical form in `WidgetXmlExporter.cpp` for asset references. Either always emit `/Script/Engine.ClassName'/AssetPath'` (full UE export-text format) or always emit bare `/Game/Path.Asset`. Document the chosen convention.

## History
- `#1-resourceobject-mixed` `OPEN` reporter — schema drift makes asset-reference parsing fragile.
- `2026-05-22` implementation — compact widget XML normalizes `FSlateBrush.ResourceObject` attributes to full UE export-text syntax such as `/Script/Engine.Texture2D'/Game/Path.Asset'`; regression coverage added under `Tests/WidgetXml`.
- `#2-verify-fix` `DONE` tester — Verified: `asset.dump` on `/Game/Blueprints/UI/W_Drone` produced tree.xml with all 7 `ResourceObject` references in canonical `/Script/Engine.Texture2D'/Game/Textures/UI/T_*.T_*'` form; no bare `/Game/...` paths or prefix-less `Texture2D'...'` forms remain.
