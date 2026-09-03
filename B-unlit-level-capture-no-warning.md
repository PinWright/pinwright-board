---
id: B-unlit-level-capture-no-warning
title: "A capture of a level with no lighting actors comes back dark with blank:false and no warning — nothing anywhere reports that the level contains no lights"
status: OPEN
severity: Medium
category: bug
tags: [render, capture_open_level, lighting, imagestats, blank, dark-frame, diagnostics, level]
encounters: 1
lastSeen: 2026-08-18T00:00:00Z
---

# Dark because the level has no lights, and nothing says so

Measured this session: capturing `L_FootballBootstrap` — a map that contains **no SkyLight and no
DirectionalLight** — returned a clean success at **mean luminance 0.20**, against **0.59** for the
lit arena map. The response carried `blank: false`, because the classifier is

```cpp
Stats.bBlank = Stats.MeanLuminance <= 0.01 && Stats.LuminanceVariance <= 0.0001;
```

(`Handlers/Render/PreviewViewportCaptureUtils.cpp:219`) — and 0.20 is twenty times the first gate.
So the one field a caller reads to ask "is this frame usable" says yes, for a frame rendered in a
level with nothing lighting it.

## What the response does and does not carry

The world-identity half is **already covered** and is explicitly not what this ticket claims.
`render.capture_open_level`'s success payload reports `levelPath` (`RenderHandler.cpp:505`),
`activeWorld` / `activeWorldPackage` / `viewportWorld` / `viewportWorldPackage` / `worldMatches`
(`AddWorldFields`, `:156-168`), the viewport block (type, viewMode, gameView, realtime) and all
four luminance stats (`AddCaptureFields`, `:139-144`). That came in with
`B-open-level-blank-success` and works.

What no verb reports is the actionable fact: **this level contains zero lighting actors.**
`grep -rn "DirectionalLight\|SkyLight" Source/PinWright/Private/Handlers/Render/` returns nothing —
the render path has no lighting awareness at all — and neither `level.get_info` nor the capture
response inventories lights. A dark frame has several plausible causes (no lights, an exposure
pin, a wrong view mode, a non-realtime viewport) and the caller is left to guess between them
from a single luminance number with nothing to compare it against.

## Why this one stings

The session was fixing exactly this defect **in the game** — `L_FootballBootstrap` ships with no
lighting actors, which is why the plan adds a SkyLight, a DirectionalLight and an `HDRIBackdrop`
to it. The tool reproduced the defect rather than diagnosing it: an agent capturing that map to
check whether the fix was needed got a dark PNG and a success response, and had to reason its way
back to "the level has no lights" by comparing against a capture of a different map.

## Neither open sibling covers it

- `B-open-level-blank-success` (IN-REVIEW, High) built `bBlank` for a **uniform black readback** —
  a dead capture, mean ≈ 0 — and its scoping comment says so. 0.20 with real variance is not that.
- `B-exposure-pin-black-frame` (OPEN, High) is the crushed-by-exposure case: correct pixels, wrong
  exposure, `pinned: true` / `blank: false` / no warning. It argues, correctly, **against**
  widening `bBlank`'s conjunction to absorb other dark-frame classes.

Both leave the same hole from opposite sides: a frame that is dark for a *scene* reason, with
every reported field individually true and the aggregate misleading.

**Fix:** take the shape `B-exposure-pin-black-frame` proposes for its own case — a warning string
beside the existing `imageStats`, not a new failure mode, since a deliberately dark scene must
stay capturable. Add the diagnosis this case needs and the other two do not: on a level-viewport
capture, count the world's lighting actors (`ADirectionalLight`, `ASkyLight`, and any
`ULightComponent` with `bAffectsWorld`) and, at zero, say so — *"level '<path>' contains no
lighting actors"*. That is a fact about the level, not a heuristic about the frame, so it can be
stated unconditionally and costs nothing when lights are present. Reporting the same count from
`level.get_info` would let a caller check before capturing rather than after.

## Related

- `B-open-level-blank-success` (IN-REVIEW) — the uniform-black classifier this frame clears, and
  the origin of the world-identity fields this ticket explicitly does **not** re-file.
- `B-exposure-pin-black-frame` (OPEN) — the sibling dark-frame case, and the proposed warning
  shape reused here.
- `E-level-spawn-light-no-sky-type`, `E-get-graph-details-light-inventory-undiscoverable` — the
  lighting-inventory surface a light count would sit on.

## History
- `#1-dark-level-reads-as-a-good-capture` `OPEN` reporter — Capturing `L_FootballBootstrap`, a map with no SkyLight and no DirectionalLight, returned success at **mean luminance 0.20** against **0.59** for the lit arena map, with `blank: false` — the classifier at `PreviewViewportCaptureUtils.cpp:219` is `MeanLuminance <= 0.01 && LuminanceVariance <= 0.0001`, and 0.20 is twenty times the first gate. Deliberately narrowed at filing: the "nothing reports the active map" half is **already covered** — `render.capture_open_level` reports `levelPath` (`RenderHandler.cpp:505`), `activeWorld`/`viewportWorld`/`worldMatches` (`AddWorldFields`, `:156-168`), the viewport block and all four luminance stats (`AddCaptureFields`, `:139-144`), all shipped with `B-open-level-blank-success`. What is missing is the lighting diagnosis: `grep DirectionalLight|SkyLight` over `Handlers/Render/` returns nothing, and no verb (capture response or `level.get_info`) reports that a level contains zero lighting actors, so a dark frame is indistinguishable from a wrong exposure, a wrong view mode or a non-realtime viewport. The session was fixing this exact defect in the game — the bootstrap map ships unlit, which is why the plan adds SkyLight + DirectionalLight + HDRIBackdrop — so the tool reproduced the bug instead of diagnosing it. Neither open sibling covers it: `B-open-level-blank-success` (IN-REVIEW) built `bBlank` for a uniform-black **dead readback** (mean ≈ 0, and its scoping comment says so), and `B-exposure-pin-black-frame` (OPEN) is the crushed-by-exposure case and argues correctly against widening `bBlank`'s conjunction. Fix: reuse that ticket's proposed shape — a warning beside `imageStats`, not a new failure mode, since a deliberately dark scene must stay capturable — plus the fact this case needs: count `ADirectionalLight` / `ASkyLight` / `ULightComponent` with `bAffectsWorld` and, at zero, state "level '<path>' contains no lighting actors". A fact about the level, not a heuristic about the frame. Same count exposed from `level.get_info` would let a caller check before capturing.
