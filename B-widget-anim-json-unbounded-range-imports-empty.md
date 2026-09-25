---
id: B-widget-anim-json-unbounded-range-imports-empty
title: "widget animation JSON: an unbounded section range exports as \"range\": {} and imports back as the empty range [0,0), so round-tripping a default UMG animation silently kills every section"
status: IN-REVIEW
severity: High
category: bug
tags: [widget, animation, umg, json, round-trip, silent-false-success, section-range]
encounters: 1
lastSeen: 2026-09-24T07:25:00Z
---

# Unbounded section range does not round-trip through widget animation JSON

`WidgetAnimationJsonSerializer.cpp`:

- `MakeRangeObject` (~`:139-151`) writes `startFrame` only if the range has a
  lower bound and `endFrame` only if it has an upper bound. An open-ended
  section (`TRange<FFrameNumber>::All()`, which is what UMG creates for every new
  widget animation section) therefore exports as `"range": {}`; a half-open one
  exports with a single field.
- `ReadRangeObject` (~`:153-171`) is called with default `TRange::All()`. When the
  `range` object is present, it seeds `StartFrame = 0` (default has no lower
  bound) and `EndFrame = StartFrame` (default has no upper bound), and then
  builds `TRange(Start, End)`. For `{}` that is `[0, 0)`, an empty range; for
  `{startFrame: S}` it is `[S, S)`.

So `widget.export_animations_json` -> `widget.import_animations_json` (and the
same shape in asset-dump `widget_animations.json`) turns every default UMG
section into an empty section. The import reports success; the animation then
evaluates nothing. The export side is also ambiguous: a reader cannot tell
"unbounded" from "field omitted".

Observed: `/…/W_AppUserPanel` `OnHovered` after its sections were made
open-ended now dumps `"range": {}` for both 2D-transform sections (export was
not imported back, so no asset was damaged here).

**Fix:** emit explicit bound types (e.g. `"start": {"frame": 0}` /
`"start": "unbounded"`, or `startBounded:false`), and in `ReadRangeObject` treat a
missing field as unbounded (use `TRangeBound::Open()`), never as the other bound.
Add a round-trip test on a freshly created widget animation section. Bump the
`widget_animations.json` aspect version if the serialized shape changes.

## History
- `#1-unbounded-range-round-trip` `OPEN` reporter — Found by source read after exporting an animation whose sections were set to (-inf,+inf): export shows `"range": {}`; `ReadRangeObject` maps `{}` to `[0,0)`. Not reproduced by an actual import (would have damaged a shipped asset); the code path is unambiguous.
- `#2-explicit-open-bounds` `IN-REVIEW` developer — Fixed both sides in `Source/PinWright/Private/Handlers/UI/WidgetAnimationJsonSerializer.cpp`. `MakeRangeObject` now writes an open side as `startBounded: false` / `endBounded: false` instead of omitting it, so `TRange::All()` exports as `{"startBounded": false, "endBounded": false}`. `ReadRangeObject` resolves each side independently: `xBounded: false` gives `TRangeBound::Open()`, a frame gives `Inclusive(start)` / `Exclusive(end)`, and an omitted side keeps the `DefaultRange` bound (open for sections, current bound for `playbackRange`), never the other side's value. Legacy `"range": {}` in existing dumps therefore imports as unbounded too. Bumped the `widget_animations.json` aspect to 3 in `AssetDumpCache.cpp`; documented the range shape in `docs/wiki-src/widget.md`. Baseline was worse than the ticket said: `{startFrame: 10}` read as `[10, 0)`, which fired the engine ensure in `UMovieSceneSection::SetRange` (`MovieSceneSection.h:340`) and got rejected, while the import still reported success. New test `PinWright.widget.animation_json.UnboundedSectionRangeRoundTrip` round-trips (-inf,+inf), [10,+inf), (-inf,20), plus the legacy `{}` shape through export and replace-mode import. It asserts the markers and exact range equality. On the unmodified serializer it fails 12 assertions (the markers are missing, the imported ranges come back as `[0,0)` and `[0,20)`, and the ensure fires). Evidence: host `Saved/Logs/PDS-backup-2026.09.24-16.31.59.log`. With the fix, it and the other 17 `WidgetAnimationJson/` tests pass (18/18, 0 fail). Plugin commit `f28feaea`.
