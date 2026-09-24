---
id: B-widget-anim-json-unbounded-range-imports-empty
title: "widget animation JSON: an unbounded section range exports as \"range\": {} and imports back as the empty range [0,0), so round-tripping a default UMG animation silently kills every section"
status: OPEN
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
