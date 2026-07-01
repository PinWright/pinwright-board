---
id: B-asset-dump-widget-animation-aspect
title: "asset.dump should emit widget animation JSON for UWidgetBlueprint assets"
status: DONE
severity: High
category: bug
tags: [asset-dump, widget, animation, round-trip]
---

# asset.dump should emit widget animation JSON for UWidgetBlueprint assets

`asset.dump` / `asset.dump_folder` currently dump widget blueprints through:

```text
meta.json
properties.json
tree.xml
bpir.txt
```

That misses persisted `UWidgetAnimation` MovieScene data. This makes dumps incomplete for widget round-trip work: BPIR can reference animation variables, but the dump does not contain the animation tracks, playback range, bindings, or optional event metadata.

## Repro

Dump `/App` via `asset.dump_folder`, then inspect the start timer widgets:

```text
C:\Unity\unreal-fpv\.editor-automation\asset-dumps\App\App\UI\W_TrackStartTimer
C:\Unity\unreal-fpv\.editor-automation\asset-dumps\App\App\UI\LobbyAndMenu\HUD\W_HUD_TrackStartTimer
```

Observed files:

```text
bpir.txt
meta.json
properties.json
tree.xml
```

`W_HUD_TrackStartTimer` has an animation variable in `properties.json`:

```text
"Full": { "type": "UWidgetAnimation*", "value": null }
```

Its BPIR references that animation:

```text
call PlayAnimation(InAnimation: $Full, ...)
```

But the dump contains no widget animation JSON, no MovieScene bindings, and no track key data.

## Existing Capability

The plugin already has a live widget animation JSON surface:

```text
widget.export_animations_json
widget.import_animations_json
```

The exporter emits schema `editor-automation.widget-animations.v1` and serializes persisted `UWidgetAnimation` MovieScene data: display rate, tick resolution, playback range, widget-name bindings, float tracks, widget material tracks, 2D transform tracks, text tracks, and optional event metadata.

The asset dump pipeline does not call this exporter. In `AssetDumpHandler`, the `UWidgetBlueprint` branch only writes `properties.json`, `tree.xml`, and `bpir.txt`.

## Expected

For every dumped `UWidgetBlueprint`, write a widget animation aspect when animations exist, for example:

```text
widget_animations.json
```

The file should use the same document schema and serializer as `widget.export_animations_json`, so dump output and live export output stay consistent.

If the export returns warnings, preserve them in the JSON document or a companion diagnostic field/file. If animation export fails for one widget, the dump should still write the other widget aspects and report the animation failure without aborting the whole folder dump.

## Required Fix

1. Add a canonical dump filename constant, likely `widget_animations.json`, to the asset dump file-name set.
2. In the `UWidgetBlueprint` branch of `AssetDumpHandler`, call the same serializer used by `widget.export_animations_json`.
3. Call the serializer with includeEventMetadata=true for dumps; document that event metadata is inspection-only and not importable visual track data.
4. Add `widget_animations.json` to baseline/diff file loading so repeated dumps compare the animation aspect.
5. Add regression coverage using `W_HUD_TrackStartTimer`: the dump must contain the `Full` animation with MovieScene bindings/tracks, not just the `UWidgetAnimation*` property stub.
6. Update docs/wiki for `asset.dump` / `asset.dump_folder` widget aspect lists.

## History

- `#1-initial-report` `OPEN` reporter — A fresh `/App` dump contains start timer widget folders with only `meta.json`, `properties.json`, `tree.xml`, and `bpir.txt`. `W_HUD_TrackStartTimer` references `$Full` via `PlayAnimation`, but no widget animation JSON or MovieScene track data is dumped.
- `#2-widget-animation-aspect` `IN-REVIEW` developer — Added `widget_animations.json` for Widget Blueprint dumps using the existing widget animation JSON serializer with event metadata, registered the aspect for diff baseline loading, added TrackStartTimer regression coverage, and updated asset dump docs.
- `#3-verify-fix` `DONE` tester — `asset.dump` on `/App/App/UI/LobbyAndMenu/HUD/W_HUD_TrackStartTimer` now reports 5 files including `widget_animations.json`; the dumped file has schema `editor-automation.widget-animations.v1`, widgetPath matching the asset, the `Full` animation with bindings/tracks for FillShadow/RadialSB/FillGlow/Time/etc., plus event metadata — shape matches `widget.export_animations_json` live output.
