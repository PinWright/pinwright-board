---
id: E-set-image-brush-rejects-material
title: "widget.set_image_brush requires a UTexture2D and rejects a UMaterial, though FSlateBrush.ResourceObject accepts one — every material-backed UMG brush has to be hand-written as ExportText"
status: OPEN
severity: Low
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
