---
id: F-set-vertex-color-no-channel-mask
title: "set_vertex_color has no channel mask — RGBA is always written as a unit, so a mesh whose RGB carries a per-part albedo ladder cannot be given a vertex-alpha mask without destroying the ladder"
status: OPEN
severity: Medium
category: feature
tags: [geometry, set_vertex_color, pwmodel, vertex-color, channel-mask, alpha, vertex-alpha, mask, missing-parameter]
encounters: 1
lastSeen: 2026-09-05T19:54:25Z
---

# Four channels, one write

`geometry.set_vertex_color` takes `r`, `g`, `b`, `a` as separate parameters but writes them as one
element — there is no parameter naming which channels the write applies to. The `.pwmodel` op is
narrower still: `set_vertex_color` accepts exactly `index`, `color`, `set_all`
(`Plugins/PinWright/Source/PinWrightGeometry/Private/Model/PwModelParser.cpp:1058-1062`, dispatched
at `PwModelCompiler.cpp:2920-2929`), and `color` is a `Vector4` written whole.

So vertex colour is a single-purpose channel in practice: whoever writes it first owns all four
components, and the second consumer cannot exist.

## The case that hits it

The idiomatic vertex-colour layout for a multi-part mesh is RGB for a per-part tint and A for a
mask — exactly the split the engine's own material nodes assume. On
`Content/FPS/Weapons/Meshes/SM_WPN_AR.pwmodel` and
`Content/FPS/Weapons/Meshes/SM_WPN_Pistol.pwmodel`, RGB already carries a per-part albedo ladder
consumed by a `VertexTintAmount` material parameter, written by `color=` on the generators. There
is no call, in either the RPC surface or the model DSL, that writes A and leaves RGB alone: any
`set_vertex_color` that sets alpha also overwrites the ladder with whatever RGB was passed
alongside it, and the ladder is per-part while the alpha write would be whole-mesh
(`set_all: true`), so it cannot even be reconstructed by passing the "same" RGB back in.

The consequence is not that alpha is hard to write — it is that the second channel group is
**unreachable**, so any feature that would produce one has nowhere to put its result. That is what
makes this worth a ticket of its own rather than a nice-to-have: it is the blocking half of
`F-no-ambient-occlusion-or-curvature-bake`, whose cheapest and preferred output is vertex alpha.

## What is asked for

A channel mask on both surfaces — for example `channels: "a"` / `channels: "rgb"` (default `"rgba"`,
so every existing caller is unaffected), or four independent optional components where an omitted
component is left untouched rather than defaulted to 1.

The RPC page's current defaults make the second form the more honest fix: `r`/`g`/`b`/`a` are each
documented "optional, default 1", so omitting them today silently writes white rather than leaving
them alone (`Saved/PinWright/wiki/geometry.set_vertex_color.md`). "Omitted means unchanged" is the
behaviour a caller reading that page would guess, and it is not what happens.

Whichever form, the same mask belongs on the `.pwmodel` op, since that is where a mesh's colour is
actually authored — a model source cannot reach the RPC mid-compile.

## Related, and deliberately not merged into this ticket

- `F-no-ambient-occlusion-or-curvature-bake` — the feature this blocks.
- `B-set-vertex-color-set-all-misses-appended` (IN-REVIEW) — `set_all` reaching appended geometry.
  Different defect: that one is about *which vertices* are painted, this one about *which channels*.
- `B-set-vertex-color-wrong-channel-not-persisted` (IN-REVIEW) — "channel" there means the legacy
  buffer vs the attribute overlay, i.e. *which storage*, not which of R/G/B/A.

## Workaround

None. The colour has to be treated as single-purpose: pick RGB or A for the whole mesh and give up
the other.

severity rationale: impact=the second channel group is structurally unreachable, so a standard
two-signal vertex-colour layout cannot be authored at all x reach=any mesh wanting both a tint and
a mask, and every feature that would write one -> Medium

## History
- `#1-filed` `OPEN` reporter — Filed from the FPS WEAPONS stream alongside `F-no-ambient-occlusion-or-curvature-bake`, which it blocks. `geometry.set_vertex_color` exposes `r`/`g`/`b`/`a` but no mask naming which of them the write applies to, and each is documented "optional, default 1" (`Saved/PinWright/wiki/geometry.set_vertex_color.md`), so omitting a component writes white rather than leaving it alone. The `.pwmodel` op is narrower: `index`, `color` (Vector4), `set_all` only (`PwModelParser.cpp:1058-1062`, `PwModelCompiler.cpp:2920-2929`). Measured consequence on `SM_WPN_AR.pwmodel` and `SM_WPN_Pistol.pwmodel`: RGB already carries a per-part albedo ladder consumed by a `VertexTintAmount` material parameter and written by generator `color=`, so there is no call that writes vertex alpha without overwriting that ladder — and since the ladder is per-part while an alpha write is whole-mesh (`set_all: true`), it cannot be reconstructed by passing the same RGB back in either. The standard RGB-tint-plus-alpha-mask layout is therefore not authorable, and any feature that would produce a second signal has nowhere to store it. Ask: `channels` mask (default `"rgba"`, so existing callers are unaffected) or per-component "omitted means unchanged" semantics, on both the RPC and the `.pwmodel` op — a model source cannot reach the RPC mid-compile. Distinct from `B-set-vertex-color-set-all-misses-appended` (which vertices) and `B-set-vertex-color-wrong-channel-not-persisted` (which storage buffer). No workaround: vertex colour has to be treated as single-purpose.
