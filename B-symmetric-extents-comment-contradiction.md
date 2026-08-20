---
id: B-symmetric-extents-comment-contradiction
title: "GeometryOps_Modeling.h contradicts itself six lines apart on bSymmetricExtents — the section header still says the warps set it, the paragraph below correctly says it was hardcoded and is now caller-supplied"
status: OPEN
severity: Low
category: bug
tags: [geometry, comments, stale, bSymmetricExtents, twist, taper, bend, warp, docs]
encounters: 1
lastSeen: 2026-08-20T00:00:00Z
---

# Two adjacent comments, opposite claims

`Source/PinWrightGeometry/Private/Handlers/Geometry/GeometryOps_Modeling.h:638-640`:

```
// Extent on all three warps is a SYMMETRIC HALF-EXTENT in world units, measured about the
// origin and spanning [-Extent, +Extent] (all three set bSymmetricExtents). A mesh of
// height H is therefore only fully covered at Extent ~= H/2; ...
```

`GeometryOps_Modeling.h:644-648`, six lines later:

```
// The two extent fields all three warps share, and the reason the header comment above
// says "symmetric". bSymmetricExtents was hardcoded true on all three - which is also the
// engine default, so publishing it changes nothing at the default - and LowerExtent was
// therefore dead. Together they are what turns the warp from [-Extent, +Extent] into the
// ASYMMETRIC [-LowerExtent, +Extent] a mesh that does not straddle the origin needs.
```

The parenthetical at `:639` is now false. The flag is caller-supplied on every path:
`GeometryOps_Modeling.cpp:1463`, `:1490` and `:1521` each read `Params.Extents.bSymmetricExtents`;
it is published as `symmetricExtents` at `MeshOpsHandler.cpp:180` and as `symmetric_extents` at
`PwModelCompiler.cpp:277`.

## Consequence

A reader who stops at the section header concludes the asymmetric mode does not exist, and sizes
`extent` against `H/2` when they did not have to — which is exactly the case the paragraph below was
written to describe.

## Only the header lags

Two other docs already carry the corrected past tense: `Docs/pwmodel-format.md:801` ("was dead while
`bSymmetricExtents` was pinned true") and `Docs/wiki-src/geometry.md:47` ("`bSymmetricExtents` was
hardcoded true"). The file was last touched by `365db75e`, which predates the wave, so nothing in the
wave addressed it.

**Fix:** replace the `:639` parenthetical with the caller-supplied fact and a pointer to the
asymmetric mode. Sweep the same sentence out of `E-geometry-warp-extent-semantics` while doing it.

## Related

- `E-geometry-warp-extent-semantics` (IN-REVIEW, Low) — the *generated public wiki* param text for
  `geometry.twist/taper/bend` lacking units and symmetry semantics. Adjacent but distinct: that is
  the published page, this is the private header contradicting itself. Note the ticket body itself
  repeats the identical present-tense error ("all three handlers set `bSymmetricExtents = true`"), so
  closing it should sweep this too.

## History
- `#1-header-says-set-paragraph-says-was-hardcoded` `OPEN` reporter — `GeometryOps_Modeling.h:639` states in the present tense that the extent is symmetric because "all three set bSymmetricExtents"; `:645`, six lines below, states the opposite history correctly — "bSymmetricExtents was hardcoded true on all three ... and LowerExtent was therefore dead" — and goes on to describe the asymmetric `[-LowerExtent, +Extent]` mode the field now enables. The flag is caller-supplied on every path: `GeometryOps_Modeling.cpp:1463`, `:1490`, `:1521` copy it from `Params.Extents`, `MeshOpsHandler.cpp:180` publishes it as `symmetricExtents`, and `PwModelCompiler.cpp:277` as `symmetric_extents`. A reader who stops at the section header concludes the asymmetric mode does not exist and sizes `extent` against `H/2` unnecessarily — the exact case the paragraph below documents. `Docs/pwmodel-format.md:801` and `Docs/wiki-src/geometry.md:47` already carry the corrected wording, so only the header lags; the file was last touched by `365db75e`, predating the wave. The board ticket `E-geometry-warp-extent-semantics` (IN-REVIEW) repeats the same stale sentence in its own body and should be swept at the same time.
