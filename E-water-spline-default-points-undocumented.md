---
id: E-water-spline-default-points-undocumented
title: "water/spline wiki never states a fresh river's default WaterSpline point count, forcing a discovery probe before authoring points"
status: OPEN
severity: Low
category: ergonomic
tags: [docs, water, spline, set_spline_point_position, add_spline_point, water-spline, discovery]
encounters: 1
lastSeen: 2026-06-23T10:08:23Z
---

# A fresh river's default WaterSpline point count is undocumented, so authoring N points needs a discovery probe first

The `water` wiki overlay's canonical workflow (`docs/wiki-src/water.md`,
step 3) tells callers to author a river/lake/custom body's shape by
"repeat[ing] `spline.set_spline_point_position` against the spawned actor's
`WaterSpline` component." What it never states is that a freshly spawned
`AWaterBodyRiver` already comes with a **non-zero number of default
WaterSpline points** (the attempt observed **3**). A caller who reads the
workflow literally and wants to lay out 4 points cannot know whether to issue
four `set_spline_point_position` calls or to `set` the existing points and
`add_spline_point` the remainder — the answer depends on the default count,
which is nowhere in the docs. So the only way to author the spline correctly
is to spend a `spline.get_splines_info` call first purely to learn the count.

Two adjacent doc gaps compound this:

1. **Default-point count not stated.** Neither `water.md` nor `spline.md`
   says how many points a fresh river/lake/custom WaterSpline has. The caller
   must probe to find out, then split the work into `set` (for the existing
   default points) + `add_spline_point` (for the extras) accordingly.

2. **"Target the WaterSpline component" overstates the API.** The workflow
   says to author points "against the spawned actor's `WaterSpline`
   component," but `spline.set_spline_point_position` and
   `spline.add_spline_point` expose **no component selector at all** — their
   only params are `actorName` / `pointIndex` / `position` (see
   `SplineHandler.cpp:410` and `:288`). Both resolve the target via
   `FindSplineComponent(Actor)` with no name, which returns
   `SplineComponents[0]` (`SplineHandler.cpp:103`). It happens to hit the
   WaterSpline only because that is the body's sole/first spline component.
   The wiki implies a targeting step that does not exist, which also costs
   the caller a moment of "how do I name the WaterSpline?" uncertainty.

This is a docs/discovery ergonomic gap, not a tool bug — every call in the
task succeeded. But the prescribed workflow is not self-contained: it cannot
be executed in one pass from the documentation alone, forcing an extra
round-trip just to discover state the docs could have stated.

**What it should do:** The `water.md` (and/or `spline.md`) overlay should
state, for each spline-based body type, how many default WaterSpline points a
freshly spawned body has (e.g. "a new `AWaterBodyRiver` starts with 3
WaterSpline points"), and spell out the author pattern: overwrite the default
points with `set_spline_point_position` (indices `0..defaultCount-1`) and use
`add_spline_point` only for points beyond the default count. It should also
correct step 3 to say the spline RPCs operate on the actor's first/only spline
component (the WaterSpline) and take no component selector, rather than
implying a `WaterSpline`-targeting parameter.

This is a downstream wiki-overlay edit on
`docs/wiki-src/water.md` (canonical workflow, step 3) and optionally
`docs/wiki-src/spline.md`; no handler code change is required.

## Evidence (this task — focus `water.spawn_water_body`)

Verbatim friction note (item 1): "the wiki/spline pages never state how
`spline.set_spline_point_position` targets the river's WaterSpline
specifically nor how many default points a fresh river has — I had to do a
pre-edit `spline.get_splines_info` to discover 3 default points and realize I
needed `add_spline_point` for the 4th rather than 4 sets."

Call-log shows the cost concretely: the river-valley task spent one
`spline.get_splines_info` (args_summary "ValleyRiver (pre-edit, 3 default
pts)") purely as a discovery probe before any edit, then authored the 4 points
as 3 × `set_spline_point_position` (idx0/1/2) + 1 × `add_spline_point` (idx3)
— a split that is only knowable after the probe. All calls returned
`ok:true`; the friction is the mandatory pre-probe, not a failure.

## History
- `#1-initial-audit` `OPEN` reporter — Process audit of the river-valley water task (focus `water.spawn_water_body`). The `water.md` canonical workflow tells callers to author a spawned river's shape by repeating `spline.set_spline_point_position` against its WaterSpline, but never states a fresh `AWaterBodyRiver` ships with N default WaterSpline points (observed 3), so the caller cannot know whether to `set` or `add` each point without first spending a `spline.get_splines_info` discovery probe. Also notes the workflow overstates the API: `set_spline_point_position`/`add_spline_point` take no component selector (params are actorName/pointIndex/position; `FindSplineComponent` returns `SplineComponents[0]`), so "target the WaterSpline component" implies a parameter that does not exist. Docs-tagged ergonomic gap; proposes stating the default-point count and the set-then-add pattern in the `docs/wiki-src/water.md` overlay (and clarifying the no-selector spline API). Distinct PROCESS angle from the judge's `E-water-underwater-settings-silent-drop` (that ticket covers the underwater-settings silent-drop; this one covers the spline-authoring discovery probe).
