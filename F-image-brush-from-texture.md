---
id: F-image-brush-from-texture
title: "Ergonomic `widget.set_image_brush` helper"
status: DONE
severity: Low
category: feature
tags: [widget-set, image, slate-brush, ergonomics]
---

# Ergonomic `widget.set_image_brush` helper

Setting the `Brush` property on an `Image` (or any `FSlateBrush` UPROPERTY) via
`widget.set` today requires the caller to compose a fully-spelled
`FSlateBrush` ExportText literal, e.g.:

```
"Brush": "(ResourceObject=Texture2D'/Game/UI/T_Icon.T_Icon',ImageSize=(X=64,Y=64),DrawAs=Image,TintColor=(SpecifiedColor=(R=1,G=1,B=1,A=1)))"
```

That string is hostile to author: the texture path must be quoted with the
class prefix, `ImageSize` must be redundantly stated even though it's
trivially derivable from the texture, and `TintColor` is a nested
`FSlateColor.SpecifiedColor` struct rather than a plain `FLinearColor`. Easy
to mistype, hard to remember, and shows up every time you author an icon.

## Proposal

A one-call shortcut:

```
widget.set_image_brush
  widgetPath: string
  widgetName: string
  texturePath: string                  # /Game/... path; class prefix auto-derived
  imageSize?: { X: number, Y: number } # optional; defaults to texture's imported size
  tint?: { R, G, B, A } | "#RRGGBBAA"  # optional; defaults to white
  drawAs?: "Image" | "Box" | "Border" | "RoundedBox" | "NoDrawType"
  margin?: { Left, Top, Right, Bottom }   # only meaningful for Box/Border
  brushProperty?: string               # defaults to "Brush"; e.g. "BackgroundImage" for some templates
```

Semantics:

- Load the texture at `texturePath`; error `ASSET_NOT_FOUND` if missing.
- Build an `FSlateBrush` in C++ with `ResourceObject = Texture`,
  `ImageSize = texture-imported-size` (overridden by `imageSize` if supplied),
  `DrawAs = drawAs` (default `Image`), `TintColor = tint` (default white),
  `Margin = margin` (default zero).
- Apply via the same `widget.set` plumbing (transaction + mark dirty +
  `requiresCompile: true`).

## Why it matters

Image brushes are one of the most frequent properties touched in HUD
authoring (icons, backgrounds, button states). The ExportText literal is the
single most verbose common-case argument in `widget.set` and a clear
ergonomic gap. Comparable to `F-widget-set-ftext-nsloctext-parse`, which
added a one-string shortcut for the same kind of "spell out the whole struct"
pain on FText.

## Alternatives considered

- **Extend `widget.set` to detect a JSON-object shape for `FSlateBrush`
  properties** (similar to the structured form mentioned in
  `F-widget-set-ftext-nsloctext-parse`). Strictly more general but spreads
  the convenience across every brush-typed property and every brush field;
  a dedicated handler is easier to discover and to document.
- **Leave the ExportText form as the only option.** Workable for tools, but
  every agent reinvents the same brush string and gets it wrong the first
  time.

## Workaround

Hand-build the ExportText literal and pass it as a string value to
`widget.set` under the brush property. Get the texture's imported size from
`asset.dump` or `texture.describe` first if you need it baked into
`ImageSize`.

## History
- `#1-reported` `OPEN` reporter — Filed during a board-grooming pass after authoring HUD icons via `widget.set` repeatedly required spelling out the full `FSlateBrush` ExportText struct. No existing handler covers the brush-from-texture case (`widget.create_style` only adds style member variables; `widget.set` takes raw ExportText). Proposing a low-priority ergonomic shortcut analogous to `F-widget-set-ftext-nsloctext-parse`.
- `#2-implemented-image-brush-handler` `IN-REVIEW` developer — Added `widget.set_image_brush` handler in `Handlers/UI/WidgetSetImageBrushHandler.cpp` that loads a UTexture2D, constructs an FSlateBrush with optional ImageSize/Tint/DrawAs/Margin/brushProperty, and writes it to the named widget instance through a scoped transaction. Regression test `FWidgetSetImageBrushAppliesTextureTest` in `Tests/Widget/TestWidgetSetImageBrushHandler.cpp` verifies the texture resource and ImageSize round-trip; counterfactual: reverting the brush write makes `Brush.GetResourceObject()` nullptr.
- `#3-verify-fix` `DONE` tester — Verified: created temp `/Game/App/UI/Test/W_McpVerifyTemp_F_image_brush` with an Image child, called `widget.set_image_brush` with `texturePath=/Engine/EditorResources/T_EditorHelp.T_EditorHelp`, `imageSize={32,32}`, `tint="#FF8040FF"`, `drawAs=Box`, `margin={0.1,0.2,0.3,0.4}`; `widget.describe` returned `Brush=(TintColor=(SpecifiedColor=(R=1.000000,G=0.215861,B=0.051269,A=1.000000)),DrawAs=Box,ImageSize=(X=32,Y=32),Margin=(0.1,0.2,0.3,0.4),ResourceObject="/Script/Engine.Texture2D'/Engine/EditorResources/T_EditorHelp.T_EditorHelp'",...)` — every field round-tripped; temp asset deleted.
