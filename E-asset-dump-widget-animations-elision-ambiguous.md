---
id: E-asset-dump-widget-animations-elision-ambiguous
title: "asset.dump elides widget_animations.json when no animations exist; missing sidecar is ambiguous"
status: DONE
severity: Low
category: ergonomic
tags: [asset-dump, widget, animation, sidecar, ergonomics]
---

# asset.dump elides widget_animations.json when no animations exist; missing sidecar is ambiguous

`AssetDumpHandler.cpp` only writes `widget_animations.json` when
`WBP->Animations.Num() > 0` (Widget Blueprint branch around line 481).
For widgets with no UMG animations, the file is silently omitted — no
empty stub, no diagnostic, no marker.

Consumers reading the dump cannot distinguish:
1. **Widget has zero animations** (the common case, expected).
2. **Dump failed silently** for this aspect on this asset.

Both look identical: just no `widget_animations.json` in the folder.
There is no positive signal anywhere ("animations: 0", aspect-completion
marker, etc.) that disambiguates them. The wiki text on
`asset.md` says "`widget_animations.json` when persisted animations
exist" but does not call out the ambiguity or describe what missing
means.

## Empirical confirmation (current corpus)

Sweep of `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/`:

- 548 widget BP folders (folders with `tree.xml`).
- 108 have `widget_animations.json`. All 108 have a non-empty
  `animations[]` array.
- 440 have no `widget_animations.json` at all.
- **0 widgets have a stub `{"animations":[]}` sidecar** anywhere in
  the corpus.

Spot-checked elided cases (e.g. `App/App/UI/BP_OverlayKey`,
`App/App/UI/KeysOverlay/BP_BuildMenuOverlay`,
`App/App/UI/KeysOverlay/BP_OverlayKey`) — all are legitimate widgets
with zero animations defined; the source-side guard
(`WBP->Animations.Num() > 0`) correctly skipped them. No silent
failures observed, but the dump shape gives the reader no way to
verify that without re-loading the asset live.

## Impact

Cosmetic / ergonomic. Tools that programmatically validate dumps
(e.g. "every widget folder has these N sidecars") either have to
special-case `widget_animations.json` as conditionally present, or
fall back to re-querying the editor when the file is absent — which
is exactly what the asset-dump cache is meant to avoid.

This is the same shape of ambiguity already filed as
`B-asset-dump-mgir-empty-graph-ambiguous` for materials (empty MGIR
indistinguishable from failed MGIR), but on the cheaper end of the
severity scale because no-animations is the common majority case.

## Fix (proposed — preferred)

In the `UWidgetBlueprint` branch of `AssetDumpHandler.cpp`
(around line 481), unconditionally emit `widget_animations.json` —
when `WBP->Animations.Num() == 0`, write the schema-compliant stub:

```json
{
  "schema": "editor-automation.widget-animations.v1",
  "widgetPath": "<asset path>",
  "animations": [],
  "warnings": []
}
```

so the file is always present and always parseable. Readers can then
treat "missing file" as a hard error (silent dump failure) instead of
an ambiguous condition. Also drop the `if (...Num() > 0)` branch and
let `ExportAnimations` handle the empty-list case directly if it
already does so cleanly.

Cost: a few extra empty files per dump folder
(~440 in the current corpus), each <200 bytes. Negligible vs the
clarity gain. Cheaper than threading "expected aspects" metadata
through `meta.json` or documenting the elision convention on every
consumer.

## Fix (alternative — docs only)

If always-emit-stub is rejected, update `docs/wiki/asset.md` and
`docs/wiki/asset.dump.md` to explicitly state: "missing
`widget_animations.json` means the widget defines zero animations;
silent dump failures for this aspect are instead recorded as
diagnostics in `meta.json` / `OutFileErrors`." This relies on every
consumer reading the wiki — weaker but zero-cost.

**Recommendation:** always-emit-stub. It eliminates the ambiguity at
the source and matches the schema invariant `widget.export_animations_json`
already satisfies (the exporter produces a valid document even for
no-animation widgets).

## History
- `#1-elision-ambiguity` `OPEN` reporter — `widget_animations.json` is elided when `WBP->Animations.Num() == 0` (`AssetDumpHandler.cpp:~481`); 440 of 548 widget folders in the current corpus have no sidecar, 108 do, and zero widgets carry an empty-animations stub. Missing-sidecar is ambiguous between "no animations" and "silent dump failure". Recommend unconditionally emitting a schema-compliant stub `{schema, widgetPath, animations:[], warnings:[]}` so absence becomes a hard error signal; falls back to a docs-only note as a cheaper alternative.
- `#2-always-emit-widget-anims` `IN-REVIEW` developer — Dropped the `if (WBP->Animations.Num() > 0)` guard in `AssetDumpHandler.cpp` UWidgetBlueprint branch so `WidgetAnimationJson::ExportAnimations` is always invoked. Exporter already produces a schema-compliant `{schema:"editor-automation.widget-animations.v1", widgetPath, animations:[]}` document for zero-animation widgets, so the sidecar is now unconditionally present and absence becomes an unambiguous hard-failure signal. Added regression test `FAssetDumpWidgetAnimationAspectEmptyTest` loading `BP_OverlayKey` (confirmed zero-anim widget in corpus) and asserting the sidecar's presence + empty `animations[]` array.
- `#3-verify-empty-stub-emitted` `DONE` tester — Verified: ran `asset.dump` on `/App/App/UI/KeysOverlay/BP_OverlayKey` (the ticket's named zero-anim repro asset); `writtenPaths` now includes `widget_animations.json`, and the file contents are `{"animations":[], "schema":"editor-automation.widget-animations.v1", "widgetPath":"/App/App/UI/KeysOverlay/BP_OverlayKey"}` — schema-compliant empty stub as proposed.
- `#4-reconfirm-and-fix-frontmatter` `DONE` tester — Re-ran `asset.dump` on `/App/App/UI/KeysOverlay/BP_OverlayKey`; `writtenPaths` includes `widget_animations.json` and on-disk file matches `{"animations":[],"schema":"editor-automation.widget-animations.v1","widgetPath":"/App/App/UI/KeysOverlay/BP_OverlayKey"}`. Frontmatter was still `IN-REVIEW` despite #3's `DONE` label — flipped frontmatter to `status: DONE` to match.
