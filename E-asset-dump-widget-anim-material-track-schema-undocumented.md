---
id: E-asset-dump-widget-anim-material-track-schema-undocumented
title: "widget_animations.json material-parameter track schema is undocumented"
status: DONE
severity: Low
category: ergonomic
tags: [asset-dump, widget, animation, docs, schema]
---

# widget_animations.json material-parameter track schema is undocumented

`widget_animations.json` (and the live `widget.export_animations_json` output) contains two structurally distinct track shapes under `bindings[].tracks[]`, but the wiki only describes one of them. Consumers that wrote a reader against the documented float/2dTransform/text shape silently skip every material track because the keys live under different field names.

## The two shapes

**Documented shape** — float / 2dTransform / text tracks. Carries `propertyName` + `propertyPath` and key arrays under either `sections[].keys[]` or `sections[].channels[]` (per-channel dictionaries like `translationX`, `rotation`, `scaleX`, ...). The widget wiki entry at `docs/wiki/widget.md` line 734 names the supported track families but only sketches the float-property layout.

**Undocumented shape** — `widgetMaterial` tracks emitted by `UMovieSceneMaterialTrack`-derived classes (the slate-brush material binding used for animated brushes/fonts/backgrounds). Sample from `App/App/UI/LobbyAndMenu/Editor/W_ProgressBarEditor/widget_animations.json`:

```json
{
    "brushPropertyNamePath": ["Font"],
    "sections": [
        {
            "colorParameters": [],
            "scalarParameters": [
                {
                    "keys": [
                        { "frame": 0, "interp": "cubic", "time": 0, "value": 0, ... }
                    ],
                    ...
                }
            ],
            "range": { "endFrame": 9000, "startFrame": -2 }
        }
    ],
    "type": "widgetMaterial"
}
```

Differences from the float shape:
- No `propertyName` / `propertyPath` — replaced by `brushPropertyNamePath` (array of brush-property hop names).
- No top-level `sections[].keys[]` or `sections[].channels{}` — keys are nested one extra level inside per-parameter entries under `sections[].scalarParameters[]` and `sections[].colorParameters[]`.
- A `widgetMaterial` track binds a slate brush slot, animates one or more material parameters on the material bound to that brush, and only carries data inside those two parameter arrays.

## Affected dumps

Grep for `brushPropertyNamePath` across `.editor-automation/asset-dumps/` finds the field in 20+ `widget_animations.json` files, including `W_ProgressBarEditor`, `W_HorizontalSelector`, `W_MeteoSelector`, `W_RatesPresetSelector`, `W_ObjBHorizontalSelector`, `W_LyraButtonSmallIcon`, `W_TopButton`, `W_SimpleButton`, `W_SelectModeBottomButton`, `W_MyExperienceTileButton`, `W_LobbyTransparentButton`, `W_LobbyButtomButton`, `W_ImageButton`, `W_AnimatedButton`, `W_HUD_TrackStartTimer`, `W_LyraButtonTab`, `W_SimpleProgressBar`, `W_SimpleProgressBarRounded`, `Radial_Wheel_Item_WB`, and `W_BoundActionButton`.

## Why this is ergonomic, not a bug

The data is fully populated — frames, times, values, tangents, interp/tangent modes, channel ranges — and round-trips through `widget.import_animations_json` per `F-widget-animation-full-fidelity`. The producer side is correct. The gap is purely in the docs: anyone writing a generic `tracks[]` reader against the wiki text will look only for `keys` / `channels` and silently miss every `widgetMaterial` (and any other material-track) entry.

## Suggested fix

Extend the "widget animation JSON" section of `docs/wiki/widget.md` (currently around line 730) with a short schema table or per-track-type subsection. At minimum:

1. Enumerate the track-type values that appear in `bindings[].tracks[].type` and which field each one uses for keys (`channels` vs `keys` vs `scalarParameters`/`colorParameters`).
2. Call out that `widgetMaterial` tracks use `brushPropertyNamePath` instead of `propertyName`/`propertyPath`, and that their key arrays live one level deeper under per-parameter entries.
3. Mention that the same shape is used for the asset-dump aspect at `widget_animations.json` (the dumper reuses `WidgetAnimationJsonSerializer`), so cache readers see the same variants.

If full schema docs are too heavy, even a single sentence in the existing paragraph naming the two shapes would close the gap.

## History
- `#1-initial-report` `OPEN` reporter — Audit of `.editor-automation/asset-dumps` found 20+ `widget_animations.json` files emitting `widgetMaterial` tracks with the `brushPropertyNamePath` + `sections[].scalarParameters[]` + `sections[].colorParameters[]` shape instead of the `propertyName`/`propertyPath` + `keys`/`channels` shape used by float/2dTransform/text tracks. `docs/wiki/widget.md` line 734 names `widgetMaterial` as supported but does not document the schema variant, so consumers reading `tracks[]` and looking only for `keys`/`channels` will silently skip every material track. Data is fully populated and round-trips correctly — this is a documentation gap, not a dumper bug.
- `#2-schema-table-added` `IN-REVIEW` developer — Added per-track-type schema table to docs/wiki/widget.md inside the widget animation JSON section, covering float / 2dTransform / widgetMaterial / text variants with identifier-field and key-location columns; noted that asset dumps reuse the same shape via WidgetAnimationJsonSerializer.
- `#3-verify-schema-docs` `DONE` tester — Verified: docs/wiki/widget.md lines 736-745 contain the per-track-type schema table for float / 2dTransform / widgetMaterial / text, with identifier field and key location columns; line 745 explicitly calls out widgetMaterial using brushPropertyNamePath and nested keys, and notes the asset-dump widget_animations.json shares the shape via WidgetAnimationJsonSerializer. Doc gap closed.
- `#4-reverify-and-fix-status` `DONE` tester — Re-verified docs/wiki/widget.md lines 736-745 still carry the schema table covering float / 2dTransform / widgetMaterial / text with identifier and key-location columns; widgetMaterial row at line 742 names brushPropertyNamePath plus the nested scalarParameters[].keys[] / colorParameters[].channels location, and line 745 reiterates the asset-dump shape via WidgetAnimationJsonSerializer. Frontmatter was still IN-REVIEW despite the prior DONE bullet; corrected to DONE.
