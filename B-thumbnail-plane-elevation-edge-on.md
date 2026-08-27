---
id: B-thumbnail-plane-elevation-edge-on
title: "`asset.generate_thumbnail` `elevation` on `primitive:\"plane\"` renders the plane EDGE ON — a 2px sliver in an empty frame; the engine force-plane path zeroes orbit pitch for exactly this reason and the caller-requested plane path does not"
status: IN-REVIEW
severity: Medium
category: bug
tags: [asset, generate_thumbnail, thumbnail, primitive, plane, elevation, camera, empty-frame, false-evidence, clamp-reports-itself]
encounters: 1
lastSeen: 2026-08-27T18:58:26+05:00
---

# `elevation` is applied faithfully to a shape that cannot be looked at from above, and the result reads as a broken material

`asset.generate_thumbnail {primitive:"plane", elevation:89}` — which reads as "look at the plane
from almost straight above" — renders the plane **edge on**: a roughly 2-pixel-tall sliver across
the middle of an otherwise empty frame. Omitting `elevation` frames the same plane correctly.

The parameter is not dropped and not clamped. It is applied exactly as documented. The defect is
that the plane primitive's own orientation is **not in the vocabulary the parameter is documented
in**, so a correctly-applied value produces a picture of nothing — and the failure presents as a
blank frame, which reads as "this material is broken", not as "the camera is in the wrong place".

The parameter doc, `Handlers/Asset/AssetWorkflowHandler.cpp:650-651`:

```cpp
RPC_PARAM_OPT("azimuth", "number", "Horizontal camera angle in degrees (0 on +X, increasing toward +Y), matching camera.frame_actor. Defaults to the asset's own stored angle."),
RPC_PARAM_OPT("elevation", "number", "Camera angle in degrees above the horizon. Defaults to the asset's own stored angle."),
```

"Above the horizon" is a world-frame statement. The thumbnail plane has no world frame — it is
pinned to face the camera at one fixed attitude, and nothing tells the caller that.

## Root cause (guilty source lines)

Verified statically in this checkout's plugin source and in the UE 5.8 engine source at
`C:/UE_5.8/Engine/Source/`. Three lines, in order:

**1. PinWright maps `elevation` onto the orbit pitch.**
`Plugins/PinWright/Source/PinWright/Private/Handlers/Asset/ThumbnailPreviewOverride.cpp:299-316`:

```cpp
        if (Request.bHasElevation)
        {
            ...
            OrbitPitchProp->SetFloatingPointPropertyValue(
                ValuePtr, static_cast<double>(-Request.Elevation));
        }
```

This is correct for every solid primitive and its sign convention is derived and documented in the
comment above it (`:290-298`).

**2. The engine gives `TPT_Plane` a fixed rotation chosen for a pitch-0 camera.**
`Editor/UnrealEd/Private/ThumbnailHelpers.cpp:380-383`:

```cpp
		case TPT_Plane:
			// The plane needs to be rotated 90 degrees to face the camera
			Transform.SetRotation(FQuat(FRotator(0, -90, 0)));
			PreviewActor->GetStaticMeshComponent()->SetStaticMesh(GUnrealEd->GetThumbnailManager()->EditorPlane);
```

The plane's normal is horizontal and its attitude is a constant. It does not track the orbit.

**3. The engine already knows a plane must be viewed at pitch 0 — and only enforces it on the
path the caller did not ask for.** `FMaterialThumbnailScene::GetViewMatrixParameters`,
`ThumbnailHelpers.cpp:443-444`:

```cpp
	OutOrbitPitch = bForcePlaneThumbnail ? 0.0f : ThumbnailInfo->OrbitPitch;
	OutOrbitYaw = ThumbnailInfo->OrbitYaw;
```

`bForcePlaneThumbnail` is true only when the engine *forced* the plane (UI domain, particle-sprite
or Niagara usage — `Material.cpp:7272-7281`, reached via `ThumbnailHelpers.cpp:333`). When the
caller **asks** for `primitive:"plane"`, `bForcePlaneThumbnail` is **false**, so `OrbitPitch` is
passed straight through and the camera orbits off the plane's fixed face. A zero-thickness quad
seen ~89° off its normal is the observed sliver.

That is the whole mechanism: the engine's own guard against this exact frame exists, at the exact
line, and does not cover the requested-plane case.

**Note on `azimuth`, unmeasured.** `:444` gates nothing at all — `OrbitYaw` is passed through on
both paths. The plane's face is a constant world attitude, so a sufficiently off-axis `azimuth`
must produce the same sliver. Not measured this session; recorded so a fixer covers both parameters
rather than only the one that was hit.

## Verbatim repro

Two calls, identical but for `elevation`, same material, same size, same session (2026-08-27,
UE 5.8, this checkout):

```
asset.generate_thumbnail {assetPath:"<any non-UI material>", primitive:"plane",
                          width:768, height:768, elevation:89, outputPath:"a.png"}
  -> success. Frame is empty except a ~2px horizontal sliver across the middle.

asset.generate_thumbnail {assetPath:"<same>", primitive:"plane",
                          width:768, height:768, outputPath:"b.png"}
  -> success. Plane framed correctly, material legible.
```

Captured as `scratchpad/seamprobe2.png` (elevation 89) against `scratchpad/seamprobe3.png`
(omitted). Both responses report `success` with the same `width`, `height`, `format` and
`primitive: "plane"` — nothing in either payload distinguishes the useless frame from the good one.

## What it should do

Follow the house rule this surface already has, quoted from
`B-capture-preview-ortho-drops-elevation`: *"a clamp reports itself; a parameter accepted and
discarded is worse than one that does not exist."* Here the value is neither clamped nor
discarded — it is honoured into uselessness, which the rule's spirit covers and its letter does
not. Either behaviour is defensible as long as the response says which one happened:

- **Mirror the engine's own rule.** When the resolved primitive is `plane` (`TPT_Plane`, and the
  `TPT_None`-with-unresolvable-`PreviewMesh` fallback at `ThumbnailHelpers.cpp:357-362`, which uses
  the same rotation), force the pitch to 0 exactly as `ThumbnailHelpers.cpp:443` does on the forced path,
  and report it — an `elevationApplied: false` plus a reason naming the plane's fixed attitude,
  in the shape `B-capture-preview-ortho-drops-elevation` asks for on `shotDistribution`.
- **Or refuse**, with `INVALID_ARGUMENT` naming `cube` / `cylinder` / `sphere` / `shaderBall` as
  the primitives an angle is meaningful on. Cheaper, and it cannot produce a picture of nothing.

Either way, **document the plane's fixed attitude on the `primitive` and `elevation` parameter
docs**: the `camera.frame_actor` vocabulary the `azimuth` doc invokes does not describe this shape.

## Distinct from related tickets

- `B-capture-preview-ortho-drops-elevation` (OPEN, Medium) is the **precedent, not a duplicate**.
  Same parameter name and the same "elevation doesn't do what it says" complaint, different verb
  and different mechanism: there, `render.capture_asset_preview` **discards** `elevation` under
  `projectionMode:"orthographic"` because UE's six fixed ortho view types cannot express an
  elevated camera, and the response never says so. Here the value **is applied**, to a shape whose
  orientation was never in the documented vocabulary. A fix for either does not touch the other —
  they are different handlers, and this one has no `shotDistribution` block to report into.
- `E-generate-thumbnail-undocumented` (DONE) documented the parameter surface including
  `elevation`. The parameters are documented; the documented *behaviour* is wrong for this
  primitive. An encounter note recording that has been appended to it as a re-open candidate.
- `B-thumbnail-primitive-ignored-on-instances` is the sibling primitive defect found in the same
  session — that one is about the primitive being silently substituted, this one about the camera
  applied to a primitive the caller did get. Both live in the same two files
  (`ThumbnailPreviewOverride.cpp`, `AssetWorkflowHandler.cpp`); fix them together while the files
  are open.
- `B-thumbnail-png-writes-jpeg` (DONE) is the encoder. Orthogonal.

## Workaround

Omit `azimuth` and `elevation` on `primitive:"plane"` and accept the default framing. Use `cube` or
`cylinder` when an angle is actually needed.

severity rationale: impact=a documented parameter, correctly applied, produces an empty frame with a success payload that cannot be told from a good one, and the failure misattributes to the material (soft blocker, undocumented workaround) x reach=the plane is the primitive for flat/tiling material checks, one of two shapes a material author reaches for -> Medium

## History
- `#1-initial-repro` `OPEN` reporter — Found building the Atlantis level (map as forcing function; see host `CLAUDE.md` § "What this project is for"), 2026-08-27, UE 5.8, PinWright at this checkout's HEAD. `asset.generate_thumbnail {primitive:"plane", elevation:89, width:768, height:768}` returned success and a ~2px sliver in an otherwise empty 768x768 frame (`scratchpad/seamprobe2.png`); the identical call with `elevation` omitted framed the plane correctly (`scratchpad/seamprobe3.png`). Mechanism traced statically — plugin source at HEAD plus UE 5.8 engine source at `C:/UE_5.8/Engine/Source/`, not run live in this filing pass: (1) `ThumbnailPreviewOverride.cpp:299-316` maps `elevation` to `OrbitPitch = -elevation`, correct for solid primitives; (2) `ThumbnailHelpers.cpp:380-383` gives `TPT_Plane` the constant rotation `FQuat(FRotator(0, -90, 0))` with the engine comment "The plane needs to be rotated 90 degrees to face the camera", so its normal is horizontal and fixed; (3) `ThumbnailHelpers.cpp:443` reads `OutOrbitPitch = bForcePlaneThumbnail ? 0.0f : ThumbnailInfo->OrbitPitch` — the engine zeroes the pitch on the FORCED plane path (UI / particle-sprite / Niagara, `Material.cpp:7272-7281` via `ThumbnailHelpers.cpp:333`) and passes the caller's pitch through unguarded when the plane was REQUESTED. So the engine's own protection against this exact frame exists and does not cover the requested-plane case. `OrbitYaw` (`:444`) is gated on neither path, so `azimuth` has the same structural exposure — not measured this session, recorded so a fixer covers both. Asked for: mirror the engine rule (pitch 0 on a plane) and report it as an applied-false with a reason, or refuse with `INVALID_ARGUMENT` naming the primitives an angle is meaningful on; plus document the plane's fixed attitude, since the `camera.frame_actor` vocabulary the `azimuth` doc invokes does not describe it. `B-capture-preview-ortho-drops-elevation` cited as precedent for the house rule ("a clamp reports itself; a parameter accepted and discarded is worse than one that does not exist") and explicitly NOT as a duplicate — different verb, different mechanism, and there the value is discarded rather than honoured. Worked around by omitting the angle on planes; defect untouched.
- `#2-plane-drops-camera-angles-and-says-so` `IN-REVIEW` developer — Took the ticket's first option and covered both parameters, not just the one that was hit. `FScopedPreviewOverride` (`Handlers/Asset/ThumbnailPreviewOverride.cpp`) now resolves the effective preview shape before touching the camera — a new file-local `ResolvedShapeIsPlane` reads the ThumbnailInfo's `PrimitiveType` after the request has been applied and covers `TPT_Plane` plus the `TPT_None`-with-unresolvable-`PreviewMesh` fallback the ticket names (`ThumbnailHelpers.cpp:357-362` uses the same mesh and rotation), OR'd with `WillForcePlanePreview` so the engine-forced plane counts too. When that resolves true, the `elevation` and `azimuth` writes are skipped and the asset's own stored angles stand — which is the framing the ticket's control frame (`seamprobe3.png`, elevation omitted) proves correct. Azimuth is suppressed alongside elevation deliberately: `ThumbnailHelpers.cpp:444` gates `OrbitYaw` on neither path, and the plane's normal works out roughly to +Y in world, so `azimuth:0` is edge on for the same reason `elevation:89` is — leaving it would have left a second way to produce the same empty frame. **The drop reports itself**, since there was nothing existing to report into (the response carried only `success` / `assetPath` / `width` / `height` / `primitive` / `outputPath` / `format`, none of which distinguishes the useless frame from the good one): `AssetWorkflowHandler.cpp` now emits `elevationApplied:false` and/or `azimuthApplied:false` for whichever angle was actually passed, plus one `cameraReason` naming the plane's fixed attitude and pointing at the solid primitives an angle is meaningful on — all three only when something was really dropped, so the fields stay a signal rather than boilerplate. The `elevation` and `azimuth` parameter docs now state the plane's fixed attitude, which the `camera.frame_actor` vocabulary the `azimuth` doc invokes does not describe. Test `PinWright.asset.generate_thumbnail.PlaneKeepsFramingUnderElevation` in `Tests/Assets/TestGenerateThumbnail.cpp` renders `/Engine/EngineMaterials/DefaultMaterial` on `primitive:"plane"` twice — control with no angle, probe with `elevation:89` — asserts the probe response carries `elevationApplied:false` and a reason while the control carries no `cameraReason` at all, then measures PIXELS: coverage is the fraction of the frame differing from its own corner pixel, the control must land in a (0.15, 0.98) band or the run skips with the `PINWRIGHT_ASSERTIONS_SKIPPED` marker rather than asserting against a measurement that proved nothing, and the probe must exceed 0.15 and stay within 0.05 of the control. Counterfactual: drop the two `&& !bResolvedShapeIsPlane` conditions and the probe's coverage collapses to the ticket's ~2px sliver (~0.02) while the control keeps its framing. Rejection with `INVALID_ARGUMENT` was considered and not taken: the caller gets a usable picture plus a note, which is strictly more than a refusal. Not measured live in this pass — the assertions are the evidence.
