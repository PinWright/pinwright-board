---
id: E-set-image-brush-rejects-material
title: "widget.set_image_brush requires a UTexture2D and rejects a UMaterial, though FSlateBrush.ResourceObject accepts one — every material-backed UMG brush has to be hand-written as ExportText"
status: DONE
severity: Medium
category: enhancement
tags: [widget, set_image_brush, umg, material, slate-brush, ui-material]
encounters: 1
lastSeen: 2026-09-03T01:20:00Z
---

# `widget.set_image_brush` cannot assign a material

## Symptom

```
call("widget.set_image_brush", {widgetPath:"/Game/FPS/UI/WBP_HUD", widgetName:"RadarBack",
                                texturePath:"/Game/FPS/UI/Materials/M_HUD_RadarBack.M_HUD_RadarBack",
                                imageSize:{X:148,Y:148}, drawAs:"Image"})
-> [ASSET_NOT_FOUND] UTexture2D not found at '/Game/FPS/UI/Materials/M_HUD_RadarBack.M_HUD_RadarBack'
```

`FSlateBrush::ResourceObject` is a `UObject*` and Slate renders a `UMaterialInterface`
there perfectly well — it is how every procedural, resolution-independent UMG element is
built, and `widget.export_xml` round-trips it happily:

```
ResourceObject=/Script/Engine.Material'/Game/FPS/UI/Materials/M_HUD_Radar.M_HUD_Radar'
```

The verb is the obvious tool for the job and the only one with `imageSize`, `tint`,
`drawAs` and `margin` handled for you, but it type-checks against `UTexture2D` alone.

## Workaround

Write the whole `FSlateBrush` as an ExportText literal through `widget.set`, which does
work but means hand-authoring the entire struct — every field, including the ones the
verb would have defaulted — for what should be a one-line call:

```
call("widget.set", {widgetPath:"...", widgetName:"RadarBack", properties:{Brush:
  "(TintColor=(SpecifiedColor=(R=1.0,G=1.0,B=1.0,A=1.0),ColorUseRule=UseColor_Specified),"
  "DrawAs=Image,Tiling=NoTile,Mirroring=NoMirror,ImageSize=(X=148.0,Y=148.0),"
  "ResourceObject=/Script/Engine.Material'\"/Game/FPS/UI/Materials/M_HUD_RadarBack.M_HUD_RadarBack\"')"}})
```

Getting the nested-quote form of the object reference right is the whole difficulty, and
it is exactly the part a typed verb should own.

## Suggested change

Accept any `UObject` that Slate can draw — `UTexture2D`, `UMaterialInterface`,
`USlateBrushAsset`, `USlateVectorArtData` — and rename the parameter to `resourcePath`
with `texturePath` kept as an alias. A material has no imported size, so `imageSize`
should stay optional-but-recommended there rather than defaulting from the asset.

## Why it matters beyond this project

A HUD built from materials instead of textures is resolution-independent and carries no
texture memory; on this project it is what let a crosshair, compass tape, radar and
weapon silhouette stay crisp from 1080p to 4K. That technique is general, and this verb
is the first thing an author reaches for while using it.

## History
- `#1-re-rated` `OPEN` triage — Severity Low -> Medium. The verb rejects valid input (a `UMaterialInterface` brush resource) and the only route is hand-authoring the whole `FSlateBrush` ExportText through `widget.set`: Medium soft blocker via workaround, on a common UMG HUD authoring path (no reach modifier).
- `#2-any-drawable-resource` `IN-REVIEW` developer — Reproduced by reading the handler: it loaded `LoadObject<UTexture2D>` only, so any material path fell to `ASSET_NOT_FOUND`. `widget.set_image_brush` now declares `resourcePath` (required, `RPC_PARAM_REQ_ALIAS` with `texturePath` as the dispatcher-registered alias), loads it as `UObject`, and accepts exactly what `FSlateRHIRenderer::CanRenderResource` accepts: any `UTexture`, any `UMaterialInterface`, or an `ISlateTextureAtlasInterface` object (Paper2D sprite). Anything else (`USlateBrushAsset`, `USlateVectorArtData`, a mesh, ...) is refused with `INVALID_ASSET_TYPE` naming the class, before `FSlateBrush::SetResourceObject` (whose `ensure` would otherwise fire and silently drop the resource). Default `ImageSize`: `UTexture2D` imported size (unchanged), other textures surface size, atlas objects source dimensions; a material keeps the `FSlateBrush` default (32x32) and the response carries a `warnings` entry telling the caller to pass `imageSize`. Response adds `resourceClass` and the final `imageSize`. Behaviour change: an existing-but-undrawable object now returns `INVALID_ASSET_TYPE` instead of `ASSET_NOT_FOUND`. Files: `Source/PinWright/Private/Handlers/UI/WidgetSetImageBrushHandler.cpp`, `Source/PinWright/Private/Tests/Widget/TestWidgetSetImageBrushHandler.cpp`, `docs/wiki-src/widget.md` (new `### widget.set_image_brush`), `CHANGELOG.md`. Tests (driven through the real `FRpcDispatcher`, so the alias gate is exercised): `PinWright.widget.set_image_brush.AppliesMaterial` (material + imageSize lands, no warning), `PinWright.widget.set_image_brush.MaterialDefaultSizeWarns` (legacy `texturePath` spelling passes the gate, default size kept, warning emitted), `PinWright.widget.set_image_brush.RejectsUndrawableResource` (StaticMesh -> `INVALID_ASSET_TYPE`, brush untouched); existing `AppliesTexture` / `AssetNotFound` still cover the texture path. Filter `PinWright.widget.set_image_brush`. Fastcheck (clang syntax-only) OK on both .cpp files; not built or run yet.
- `#3-verified-linux` `DONE` tester — Passed non-skipped in run3/full: `PinWright.widget.set_image_brush.AppliesMaterial`, `MaterialDefaultSizeWarns` (legacy `texturePath` spelling through the real dispatcher), `RejectsUndrawableResource`, `AppliesTexture`, `AssetNotFound`. Acceptance met: a UMaterialInterface brush resource is accepted with `imageSize` applied; the parameter is `resourcePath` with `texturePath` as an alias; a material without `imageSize` keeps the default and warns rather than guessing. Note: the suggested `USlateBrushAsset` / `USlateVectorArtData` are refused with `INVALID_ASSET_TYPE`. That follows the ticket's own criterion ("any UObject Slate can draw") as `FSlateRHIRenderer::CanRenderResource` defines it, so neither class can render as a ResourceObject.
