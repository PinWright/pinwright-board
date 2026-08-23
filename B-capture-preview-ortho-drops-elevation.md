---
id: B-capture-preview-ortho-drops-elevation
title: "render.capture_asset_preview silently discards `elevation` when projectionMode is orthographic — the parameter doc promises an ignored elevation 'says so in shotDistribution rather than dropping it silently', and that promise is kept only for distribution:'sphere'"
status: OPEN
severity: Medium
category: bug
tags: [render, capture_asset_preview, orthographic, elevation, silent-drop, shot-plan, honest-response]
encounters: 1
lastSeen: 2026-08-23T00:00:00Z
---

# An orthographic shot set is always at elevation 0, and nothing says so

`render.capture_asset_preview` publishes `elevation` as "Elevation in degrees above the horizon
for `count` shots (default 30)", and the same sentence continues: *"Ignored by
distribution:'sphere', which says so in shotDistribution rather than dropping it silently."*

That promise is kept for `distribution: 'sphere'` and broken for `projectionMode: 'orthographic'`,
which discards the value with no signal anywhere in the response.

## Repro — one A/B, same asset, same shot plan, one field changed

Orthographic:

```js
call({method: "render.capture_asset_preview", args: {
  assetPath: "<any static mesh>", filename: "e.png",
  count: 4, elevation: 12, projectionMode: "orthographic" }})
```

Every shot comes back `pitch: 0`, the written filenames read `..._az0_el0.png` …
`..._az270_el0.png`, `shotDistribution` reads `{distribution: "ring", seeded: false,
appliesToPlan: true}` with no mention of elevation, and the response's top-level `elevation`
field is **`null`**. The `orthoView` field on each shot reads `front` / `left` / `back` /
`right`.

Perspective, identical otherwise:

```js
call({method: "render.capture_asset_preview", args: {
  assetPath: "<same mesh>", filename: "p.png",
  count: 4, elevation: 12, projectionMode: "perspective", fov: 30 }})
```

Every shot comes back `pitch: -12.000000215117`, filenames read `..._el12.png`.

## Why it happens, and why it is still a defect

The mechanism is not mysterious: a UE orthographic viewport has six fixed axis-aligned view
types, so an *elevated* orthographic camera is not expressible — the capture snaps to
`OrthoFront` / `OrthoLeft` / `OrthoBack` / `OrthoRight` and the pitch is discarded on the way.
That is a legitimate constraint. Silently dropping a caller's value is not, and this surface has
a house rule about exactly this: a clamp reports itself, a parameter that is accepted and
discarded is worse than one that does not exist.

Three consequences, in the order a caller meets them:

1. **The default is affected too.** `elevation` defaults to 30, so an orthographic `count` set
   taken with no `elevation` at all is silently a 0-elevation ring rather than the documented
   30. Nothing in the response distinguishes "you asked for 0" from "0 is all you can have".
2. **The filenames assert the wrong thing.** `_el0` is written as though `el0` were the request.
   A caller diffing two shot sets by filename cannot see that one lost its elevation.
3. **`elevation` echoes as `null` on BOTH projections**, so the response never reports the
   elevation actually used — even on perspective, where it was honoured. The per-shot
   `cameraRotation.pitch` is the only place the truth appears.

## Fix

Report it the way `distribution: 'sphere'` already reports it: when the resolved projection is
orthographic and a shot plan would need a non-zero elevation, say so in `shotDistribution` (an
`elevationApplied: false` plus a reason naming the six fixed ortho view types), and echo the
elevation actually used rather than `null`. Refusing would be wrong — the six ortho views are
genuinely useful and are what `views: 'sides'` is for — so this is a reporting fix, not a
behaviour change. Consider also naming the constraint in the `elevation` parameter doc beside
the existing `distribution:'sphere'` clause, since the clause reads as an exhaustive list of the
cases where elevation does not apply.

## History
- `#1-ortho-silently-flattens-the-ring` `OPEN` reporter — found while capturing multi-angle
  verification shots of a static mesh. `count: 4, elevation: 12, projectionMode: "orthographic"`
  returned four shots at `pitch: 0` with filenames `_el0` and `shotDistribution` reporting a
  plain `ring`; the identical call at `projectionMode: "perspective"` returned `pitch:
  -12.000000215117` and filenames `_el12`. Top-level `elevation` echoed `null` in both. The
  mechanism is the engine's six fixed orthographic view types, which is a real constraint — the
  defect is that the constraint is invisible, that it also silently rewrites the documented
  default of 30, and that the parameter's own doc promises an ignored elevation is reported in
  `shotDistribution` rather than dropped. Reporting fix, not a behaviour change.
