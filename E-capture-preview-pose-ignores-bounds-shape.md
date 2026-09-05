---
id: E-capture-preview-pose-ignores-bounds-shape
title: "render.capture_asset_preview's default pose ignores the bounds' aspect: on a long thin asset it lands on the bore axis (an 82 uu rifle framed end-on at subjectCoverage 0.0039), and framing.boundsInFrame still reads true"
status: OPEN
severity: Low
category: ergonomic
tags: [render, capture_asset_preview, default-camera, framing, boundsInFrame, subjectCoverage, offAxis, frameLimit, aspect-ratio, long-thin-asset, weapons]
encounters: 3
lastSeen: 2026-09-05T00:00:00Z
---

# The default camera azimuth is a world axis, so a long thin asset is shot down its own length

## What was called

```
render.capture_asset_preview {assetPath: "/Game/FPS/Weapons/Meshes/SM_WPN_AR",
                              filename: <…>, width: 1024, height: 1024}
```

No `location` / `rotation` / `angles` — the zero-argument framing path.

## What happened

The camera was placed at **`(-98.9, 0.46, 13.1)`, yaw 0** — directly behind the buttplate of an
82 uu rifle, looking down the barrel. The subject renders as a dot: **`subjectCoverage: 0.0039`**,
i.e. 0.39% of a 1024×1024 frame for an asset that fills the shot when viewed broadside.

Two things about the response, in the order they matter:

- **The coverage warning did fire.** `measureCoverage` did its job and the response said the
  subject was barely in frame. That is exactly the signal it is documented to provide, and it is
  the reason this ticket is Low rather than High — the verb did not lie about the outcome.
- **`framing.boundsInFrame` still read `true`.** Geometrically correct and, next to a 0.39%
  coverage number, actively unhelpful: an end-on rifle's bounds *are* in frame. A caller reading
  the framing verdict alone — or a pipeline gating on it — sees a healthy capture.

## Why the pose lands there

The default is a world-axis pose: camera on −X looking toward +X, offset up by a fraction of the
bounding-sphere radius, with the distance fitted to that sphere. The measured numbers are
consistent with the formula `B-capture-preview-default-camera-vs-light` records source-read at
`RenderHandler.cpp:313-317` (`Center + FVector(-Distance, 0, Radius*0.35)`): `Radius*0.35 = 13.1`
→ `Radius ≈ 37.4`, and an FOV-fit of that radius at a ~45° FOV gives `Distance ≈ 98`, against the
measured 98.9. *(That arithmetic is inference from the relayed numbers, not a source read taken
for this ticket — but it does mean this is the same code block, not a second one.)*

The consequence follows directly: the azimuth is chosen without reference to the **shape** of the
bounds. A roughly-isotropic asset (the rook, the crane, a cube) frames fine from any azimuth, so
the defect never showed. A rifle, a sword, a pipe, a railing, a ladder — anything whose bounds are
long on one axis — is a coin flip, and lands end-on whenever its long axis happens to be X.

## What is asked for

**Derive the default orbit azimuth from the bounds' longest axis**, so a long thin asset is framed
broadside by default: pick the azimuth perpendicular to the dominant horizontal extent (and, for a
vertically-dominant asset, back off and pitch rather than shooting from directly above). The
bounding box is already in hand — `bounds` is in the response — so this is a choice of azimuth,
not new measurement.

Second, smaller ask: **`framing.boundsInFrame` should not read `true` unqualified next to a
sub-percent `subjectCoverage`.** Either fold coverage into the framing verdict, or have the
verdict say *why* it is satisfied (bounds in frame, subject 0.4% of it), so the two fields stop
disagreeing in tone about the same picture.

## Severity

**Low** — pure friction, honestly reported. The subject really was in frame, `measureCoverage`
really did warn, and the workaround (pass an explicit `location`, or use `camera.orbit_shots` with
chosen `angles`) is available and documented. It earns a ticket because the *default* is the path
every first capture of an asset takes, and on a weapons kit — a whole folder of long thin assets —
it is wrong every time, so the warning becomes noise the caller learns to click past.

## Related — same code block, must be fixed together

- `B-capture-preview-default-camera-vs-light` (OPEN, Medium) — **the same zero-argument pose, at
  the same source lines, failing for a second independent reason.** That ticket's finding is that
  the pose is chosen with no reference to the fixed preview lighting rig, with measured luminance
  A/Bs and three false "inverted normals" reports; its proposed fix derives the pose from
  `UAssetViewerSettings::Get()->Profiles[i].DirectionalLightRotation`. This ticket's finding is that
  the same pose is chosen with no reference to the bounds' aspect. Filed separately because the
  evidence and the failure mode are different, but **whoever rewrites that block must satisfy both
  constraints at once** — a pose derived from the light rig alone will still shoot a rifle down the
  barrel, and a pose derived from the longest axis alone will re-open the lighting question. Two
  inputs, one azimuth decision.
- `B-orbit-shots-no-subject-coverage` (OPEN, Medium) — `camera.orbit_shots`, the verb you are
  pushed onto as the workaround here, is the one that **cannot ask for** `subjectCoverage`. So the
  documented escape from this defect is also the path on which the warning that caught it does not
  exist. Adds reach to both tickets.
- `B-capture-preview-ortho-drops-elevation` (OPEN) — a third defect in the same verb's default
  shot-plan handling; different parameter, same "the default quietly is not what you would expect"
  family.
- `B-capture-asset-preview-renders-empty` (IN-REVIEW) — the blank-frame bug; a different cause of a
  bad frame, explicitly not this.

## History
- `#1-filed` `OPEN` WEAPONS-critic — Measured during a WEAPONS critic review. `render.capture_asset_preview {assetPath:"/Game/FPS/Weapons/Meshes/SM_WPN_AR", filename:…, width:1024, height:1024}` with no camera arguments placed the camera at `(-98.9, 0.46, 13.1)` yaw 0 — directly behind the buttplate looking down the barrel — so an 82 uu rifle rendered at `subjectCoverage: 0.0039`, 0.39% of a 1024×1024 frame. Credit where due: the verb's own coverage warning **did** fire, which is why this is Low and not a silent-wrong-data ticket. But `framing.boundsInFrame` still read `true` — geometrically correct and unhelpful beside a sub-percent coverage number, since an end-on rifle's bounds are indeed in frame; a caller or pipeline reading the framing verdict alone sees a healthy capture. Root cause is the fixed world-axis default azimuth: the measured pose is consistent with the `Center + FVector(-Distance, 0, Radius*0.35)` formula that `B-capture-preview-default-camera-vs-light` records source-read at `RenderHandler.cpp:313-317` (`Radius*0.35 = 13.1` → `Radius ≈ 37.4`; an FOV-fit of that radius at ~45° gives `Distance ≈ 98` vs the measured 98.9) — that arithmetic is inference from the relayed numbers, not a source read taken here, but it places this on the same code block. Ask: derive the default orbit azimuth from the bounds' longest axis so a long thin asset is framed broadside, and stop `framing.boundsInFrame` from reading an unqualified `true` next to a sub-percent coverage. Must be fixed together with `B-capture-preview-default-camera-vs-light`: that ticket wants the same azimuth derived from the preview light rig, and satisfying either constraint alone leaves the other broken.
- `#2-reproduced-round-2` `OPEN` WEAPONS-critic — Reproduced in a WEAPONS critic review round 2, same asset (`SM_WPN_AR`), same defect, independent capture. Default framing placed the camera at **(−64.85, −0.11, 13.07) yaw 0** — again straight down the bore from behind — for **`subjectCoverage: 0.0228`**, with **`framing.offAxis −31.6`** against a **`frameLimit` of 52.9**. Two things this adds to `#1`. **First, the coverage warning fired again**, which is worth stating plainly: the signal exists, it works, and it is why this ticket stays Low — the verb is not lying, its default is just useless for a long thin asset, every single time, on a folder made entirely of them. **Second, the pose is not identical to `#1`'s and the difference is unexplained.** The up-offset matches to two decimals (13.07 here vs 13.1 in `#1` → the same `Radius*0.35`, `Radius ≈ 37.3`), and the azimuth is identical (yaw 0, −X looking +X), but the standoff distance is **64.85 here vs 98.9 in `#1`** for the same asset. Something in the framing inputs differed between the two captures (`#1` was 1024×1024 at the default FOV; this round's width/height/fov were not recorded alongside the pose), so the fit distance moved while the azimuth — the thing this ticket is about — did not. Recorded rather than diagnosed: the standoff is a red herring for this ticket and possibly a real question for whoever reads the fit code, since coverage rose only from 0.0039 to 0.0228 despite the camera sitting 34% closer, which is what an end-on subject does. Net: the default azimuth is wrong for this asset class regardless of how the distance is fitted, which is the same conclusion `#1` reached and the reason the ask in this ticket is about azimuth, not distance. No source was read for this entry. Two further defects in the same verb were filed the same round: `B-preview-key-azimuth-moves-backdrop` (an azimuth A/B silently moves the environment too) and, on the mesh-review side that prompted these captures, `F-static-mesh-section-material-map`.
- `#3-reproduced-round-3-and-the-ev100-is-unusable-too` `OPEN` WEAPONS-critic — Reproduced a third time in a WEAPONS critic review round 3, same asset (`SM_WPN_AR`), same defect: the default pose frames the 82-uu rifle **end-on** for **`subjectCoverage 0.0154`** — between `#1`'s 0.0039 and `#2`'s 0.0228, i.e. the coverage varies with the unexplained standoff `#2` recorded while the azimuth, the thing this ticket is about, does not vary at all. Three captures, three rounds, one azimuth. **New this round — the default shot's exposure is unusable as a pin, which makes the bad framing cost more than one wasted capture.** The workflow the plugin steers a caller into is: take a default shot, read its `ev100Equivalent`, then pin that value across a set of comparison shots so the frames are photometrically comparable (that pinning is what `B-preview-key-azimuth-moves-backdrop`'s A/B depended on, and it is the standard remedy for `B-preview-capture-lighting-cycles-per-shot`). The default shot here reported **`ev100Equivalent −5.022`** — an exposure fitted to a frame that is 98.5% backdrop and 1.5% subject. Pinning it for a properly framed profile shot **blew the frame out to mean 0.805**. So the default capture yields neither a usable image nor a usable exposure reference, and a caller following the documented pin-then-compare recipe gets a white frame on the *second* call, one step removed from the cause. Ask, added to the azimuth ask already in the body: derive `ev100Equivalent` from the **subject** region rather than the whole frame, or at minimum do not present a whole-frame exposure fitted over a sub-2%-coverage subject as a pinnable reference — a `subjectEv100` alongside the frame-wide value would satisfy the recipe without changing the metering. **Severity stays Low** on the same reasoning `#2` gave: the coverage warning fired again, so the verb is not lying, and every failure here is visible in the response. Recorded as reach and as one added ask, not as an escalation. No source was read for this entry. Filed the same round from these captures: `B-material-readiness-false-notcompiled-warning` (the readiness block warned "may have used the engine Default Material" on all 19 of this round's captures, including these). The sibling `B-preview-key-azimuth-moves-backdrop` was **confirmed fixed and closed DONE** this round, so the environment no longer moves under an azimuth A/B — which leaves this ticket's fixed default azimuth as the remaining framing defect on the verb.
