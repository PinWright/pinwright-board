---
id: B-unlit-level-capture-no-warning
title: "A capture of a level with no lighting actors comes back dark with blank:false and no warning — nothing anywhere reports that the level contains no lights"
status: DONE
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
- `#2-level-lighting-count-reported` `IN-REVIEW` developer — Fixed in the shape the ticket asked for: a fact about the level, beside `imageStats`, never a refusal. New `Utils/LightingSurvey.h/.cpp` (`PinWrightLightingSurvey::SurveyWorld` / `SurveyLevel` / `AddLightingFields`) counts every `ULightComponentBase` with `bAffectsWorld` that is visible, split into directional, sky (`USkyLightComponent`, which is not a `ULightComponent`) and local; `SurveyWorld` walks only VISIBLE levels, since a hidden streaming level lights nothing in the frame. `render.capture_open_level` (`PinWrightOpenLevelCapture::Handle`, so `effect.step_and_capture` gets it too) now publishes `lighting {directionalLights, skyLights, localLights, lightComponents}` on every success and on the `BLANK_CAPTURE` details, plus `lightingWarning` — "level '<path>' contains no lighting actors ..." — when the count is zero. `level.get_info` publishes the same block for its target level, so a caller can check before capturing. Two targeted insertions in `RenderHandler.cpp` (after `AddWorldFields` on the success and blank paths) plus one include; no other line of the open-level path moved. Docs: `docs/wiki-src/render.md` (capture_open_level important fields), `docs/wiki-src/level.md` (get_info). Tests: `Tests/Render/TestLevelLightingSurvey.cpp` — `PinWright.render.lighting_survey.WarnsOnlyWhenTheLevelHasNoLights` (pure: zero count warns and names the level, one sky light silences it), `PinWright.render.lighting_survey.CountsLightsThatAffectTheWorld` (spawns a DirectionalLight, a PointLight and a PointLight with `bAffectsWorld=false`; asserts the DELTA is 1 directional + 1 local, so the disabled light is the failure direction; and that `level.get_info` reports the survey's count with no warning), `PinWright.render.lighting_survey.OpenLevelCaptureReportsLighting` (live, RHI-guarded with skip markers: the capture carries the block and `lightingWarning` iff the count is zero). Verification for a tester: capture an unlit map with `render.capture_open_level` and read `lighting.lightComponents: 0` plus `lightingWarning`; add a SkyLight and recapture — the warning is gone.
- `#3-review-fixes` `IN-REVIEW` developer — Review fixes. (1) `level.get_info` warned falsely for a persistent level lit from a visible sublevel, because it decided `lightingWarning` from `SurveyLevel(target)` while `render.capture_open_level` surveys every visible level. `PinWrightLightingSurvey::AddLightingFields` takes an optional world survey (`Utils/LightingSurvey.h/.cpp`); `level.get_info` (`Handlers/Level/LevelHandler.cpp`) keeps the level's own `lighting` counts, adds `lighting.worldLightComponents`, and warns only when the world count is zero. Warning text now says "renders with no lighting ... in any visible level of its world". Docs `level.md`, `render.md`, CHANGELOG. (2) `PinWright.render.lighting_survey.OpenLevelCaptureReportsLighting` skipped on ANY error; it now skips only on typed viewport-unavailable codes (NO_ACTIVE_LEVEL_VIEWPORT, NO_EDITOR_WORLD, EDITOR_NOT_AVAILABLE, VIEWPORT_WORLD_MISMATCH, CAPTURE_NOT_READY, CAPTURE_FAILED) and `AddError`s otherwise; `allowBlank` dropped, so a `BLANK_CAPTURE` refusal is accepted and its details must carry the lighting block (that path was never exercised). Tests: `PinWright.render.lighting_survey.WarnsOnlyWhenTheLevelHasNoLights` (sublevel-lit and world-unlit cases), `.CountsLightsThatAffectTheWorld` (get_info `worldLightComponents` equals `SurveyWorld`), `.OpenLevelCaptureReportsLighting`.
- `#4-verified-linux` `DONE` tester — Verified on the committed tree (PinWright 8de8a5a2, pushed as 7230b41d). run3/full, all non-skipped: `PinWright.render.lighting_survey.WarnsOnlyWhenTheLevelHasNoLights` (zero lights warns and names the level; a sublevel-lit persistent level does not warn), `.CountsLightsThatAffectTheWorld` (a spawned DirectionalLight plus PointLight count as 1 directional + 1 local, a `bAffectsWorld=false` light is excluded, and `level.get_info` reports `lighting` plus `worldLightComponents` with no false warning) and `.OpenLevelCaptureReportsLighting` (a live `render.capture_open_level` carries the `lighting` block, and `lightingWarning` iff the count is zero). That covers the ask: a lighting-actor count beside imageStats, the "contains no lighting" warning that never refuses, and the same count on `level.get_info`. Coverage limit: the live capture ran on the test map's own lighting, so the zero-light warning is proven by the pure test, not by a capture of a real unlit map such as `L_FootballBootstrap`.
