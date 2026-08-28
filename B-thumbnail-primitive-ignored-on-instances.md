---
id: B-thumbnail-primitive-ignored-on-instances
title: "`asset.generate_thumbnail` silently substitutes a flat plane for the requested `primitive` on some material INSTANCES while their master renders the shape correctly — and the response echoes the requested primitive either way, so nothing reports the substitution"
status: DONE
severity: High
category: bug
tags: [asset, generate_thumbnail, thumbnail, primitive, material-instance, silent-noop, input-echo, force-plane, false-evidence, honest-response]
encounters: 1
lastSeen: 2026-08-28
---

# The response says `primitive: "sphere"` and the pixels are a flat quad

`primitive: "sphere"` (or `"cylinder"`) is honoured on a `UMaterial` and on some material
instances, and silently ignored on others, which render as a flat, uniformly lit camera-facing
quad. **The response echoes the requested primitive in both cases**, so nothing in the payload
reports the substitution.

A flat quad hides everything a shape exists to reveal: no lighting falloff, no silhouette, no
fresnel rim, no curvature. The fresnel sheen on `M_Fish` is invisible on every one of its four
instances and could only be verified on the master. Combined with
`E-material-capture-no-thumbnail-pointer` — `render.capture_asset_preview` refuses materials
outright — there is no way left to look at a material instance on a curved surface when this fires.

## Verbatim repro — same request, one asset apart

2026-08-27, UE 5.8, PinWright at this checkout's HEAD, during the Atlantis build.

```
asset.generate_thumbnail {assetPath:"/Game/Atlantis/Materials/M_Fish",
                          primitive:"sphere", width:320, height:320, outputPath:"a.png"}
  -> success, "primitive":"sphere".  scratchpad/M_Fish.png: a correctly lit SPHERE with a
     specular highlight and a visible fresnel rim.

asset.generate_thumbnail {assetPath:"/Game/Atlantis/Materials/MI_Fish_Silver",
                          primitive:"sphere", width:320, height:320, outputPath:"b.png"}
  -> success, "primitive":"sphere".  scratchpad/MI_Fish_Silver.png: a FLAT, uniformly lit
     square of the same grey.
```

`MI_Fish_Silver` is a plain instance of `M_Fish` whose `BodyColor` override equals the master's
default, so the two should differ only in shape — and they differ only in shape, in the wrong
direction. `scratchpad/MI_Fish_Blue_cyl.png` is the same flat square in blue with
`primitive:"cylinder"` requested, and reports `"primitive":"cylinder"`.

**The counter-example that rules out "all instances".** `scratchpad/MI_Coral_Pink.png`,
`MI_Coral_Orange.png` and `MI_Coral_Violet.png` — instances created the same way, in the same
session, of `M_Coral` — **do** render as spheres.

## Root cause — traced statically to engine source; the live control is NOT run

The reporter's hypothesis was explicitly unisolated: `M_Fish` carries
`bUsedWithNiagaraMeshParticles: true` **and** is one-sided, while `M_Coral` carries no particle
usage flags **and** is two-sided — two variables, neither controlled. Chasing that hypothesis into
UE 5.8 engine source at `C:/UE_5.8/Engine/Source/` resolves it as follows. Everything below is a
static read of code, matched against the three observed frames; **no A/B was run in this filing
pass**, and the control that would close it is named at the end.

**The two-sidedness variable is ruled out by source.** `UMaterial::ShouldForcePlanePreview()`
(`Runtime/Engine/Private/Materials/Material.cpp:7272-7281`) does not read two-sidedness at all:

```cpp
bool UMaterial::ShouldForcePlanePreview()
{
	const USceneThumbnailInfoWithPrimitive* MaterialThumbnailInfo = Cast<USceneThumbnailInfoWithPrimitive>(ThumbnailInfo);
	if (!MaterialThumbnailInfo)
	{
		MaterialThumbnailInfo = USceneThumbnailInfoWithPrimitive::StaticClass()->GetDefaultObject<USceneThumbnailInfoWithPrimitive>();
	}
	const bool bUsedWithNiagara = bUsedWithNiagaraSprites || bUsedWithNiagaraRibbons || bUsedWithNiagaraMeshParticles;   // :7279
	return Super::ShouldForcePlanePreview() || IsUIMaterial() || (bUsedWithParticleSprites && !MaterialThumbnailInfo->bUserModifiedShape) || (bUsedWithNiagara && !MaterialThumbnailInfo->bUserModifiedShape);   // :7280
}
```

Two things to read off that line. `bUsedWithNiagaraMeshParticles` — the flag `M_Fish` carries — is
one of the three that make `bUsedWithNiagara` true. And the particle/Niagara terms are **gated by
`bUserModifiedShape`**, i.e. they can be defeated, unlike `IsUIMaterial()`, which cannot. The
plugin knows this and says so in its own comment at
`Plugins/PinWright/Source/PinWright/Private/Handlers/Asset/ThumbnailPreviewOverride.cpp:277-288`,
which sets exactly that flag to defeat exactly that rule:

```cpp
            // A particle-sprite / Niagara material is force-planed by
            // UMaterial::ShouldForcePlanePreview unless the user has deliberately picked a
            // shape; an explicit `primitive` IS that deliberate pick. (A UI-domain material
            // is force-planed unconditionally and this flag does not rescue it.)
            ...
                UserModifiedShapeProp->SetPropertyValue_InContainer(Existing, true);   // :287
```

**And here is the asymmetry.** `Existing` is the ThumbnailInfo of *the asset the caller named*. The
engine does not read that object when deciding whether to force a plane. `FMaterialThumbnailScene::SetMaterialInterface`,
`Editor/UnrealEd/Private/ThumbnailHelpers.cpp:323-340`:

```cpp
		const USceneThumbnailInfoWithPrimitive* ThumbnailInfo = Cast<USceneThumbnailInfoWithPrimitive>(InMaterial->ThumbnailInfo);   // :323  <-- the INSTANCE's, which PinWright did set
		...
		UMaterial* BaseMaterial = InMaterial->GetBaseMaterial();
		if(BaseMaterial)
		{
			bForcePlaneThumbnail = BaseMaterial->ShouldForcePlanePreview();     // :333  <-- reads the MASTER's ThumbnailInfo
		}
		...
		EThumbnailPrimType PrimitiveType = bForcePlaneThumbnail ? TPT_Plane : ThumbnailInfo->PrimitiveType.GetValue();   // :340
```

`GetBaseMaterial()` returns the root `UMaterial` for an instance and the material itself for a
master, so `UMaterial::ShouldForcePlanePreview()` always consults **the master's own**
`ThumbnailInfo->bUserModifiedShape`. That accounts for all three observed frames:

| asset | where PinWright set `bUserModifiedShape` | where the engine reads it | result |
|---|---|---|---|
| `M_Fish` (master, Niagara usage) | `M_Fish`'s ThumbnailInfo | `M_Fish`'s ThumbnailInfo | rule defeated → sphere ✔ observed |
| `MI_Fish_Silver` (instance of it) | the **instance's** ThumbnailInfo | `M_Fish`'s ThumbnailInfo, still `false` | forced → `TPT_Plane` at `:340` ✔ observed |
| `MI_Coral_*` (instance of `M_Coral`, no particle flags) | the instance's ThumbnailInfo | `M_Coral`'s — irrelevant, no term is true | not forced → sphere ✔ observed |

Note the shape of the failure at `:340`: the requested `PrimitiveType` **is** read off the
instance's ThumbnailInfo at `:323` (PinWright's write landed) and is then discarded by the ternary.
The value is applied and overridden, not dropped — which is why nothing anywhere reports it.

**Still unverified, and the exact control that would settle it.** The trace above is a code read,
not an experiment. Run this and it is closed either way:

1. Duplicate `M_Fish` as `M_FishControl`, clear `bUsedWithNiagaraMeshParticles` (leaving
   one-sidedness alone), create one instance of each, and request `primitive:"sphere"` on all four.
   The prediction is: both masters render spheres; the control's instance renders a sphere; the
   unmodified `M_Fish`'s instance renders a flat quad.
2. Or, cheaper and equally decisive: set `bUserModifiedShape = true` on **`M_Fish`'s own**
   ThumbnailInfo and re-request `primitive:"sphere"` on `MI_Fish_Silver`. If it renders a sphere,
   the mechanism above is confirmed and the fix follows directly.

## What it should do

**Primary ask — report the primitive that was actually drawn, not the one that was requested.**
Today `Handlers/Asset/AssetWorkflowHandler.cpp:847-852` echoes the input:

```cpp
    if (!PrimitiveName.IsEmpty())
    {
        // The shape ASKED FOR. A UI-domain material is force-planed by the engine whatever is
        // requested, so this is not a promise about the pixels.
        Result->SetStringField(TEXT("primitive"), PrimitiveName);
    }
```

The comment is honest and the field is not. A caller reading the response to answer "what shape am
I looking at?" sees `primitive: "sphere"` and is wrong. This is the house rule
`E-remesh-uniform-echoes-target-not-achieved` (IN-REVIEW, Medium) argues at length: a response that
carries a **field named after the answer** but holding the *request* is worse than one that carries
nothing, because there is nothing to prompt a readback. That ticket's remedy is the one to copy —
report the realised value, and keep the requested one alongside it only when they differ, the way
this same handler already does for size (`requestedWidth` / `requestedHeight`,
`AssetWorkflowHandler.cpp:841-846`).

Concretely: resolve force-planing before answering, and publish `primitive` = what was drawn plus
`requestedPrimitive` and a reason string when they differ (naming which rule forced it — UI domain,
particle sprite, or Niagara usage on the base material). `bForcePlaneThumbnail` is engine-internal,
but the predicate is reproducible from the plugin side: `GetBaseMaterial()->ShouldForcePlanePreview()`
is public.

**Secondary — make the override actually work on an instance.** `FScopedPreviewOverride` already
saves and restores every field it touches on every exit path; extend the same scoped write of
`bUserModifiedShape` to the **base material's** ThumbnailInfo when the named asset is a
`UMaterialInstance`, since `ThumbnailHelpers.cpp:333` is the object the gate reads. Restore it the
same way. That is a small change to code that is already built for it.

**And document the second forcing rule.** `E-generate-thumbnail-undocumented` `#3` put the
UI-domain force-plane caveat on the wiki page. The particle-sprite/Niagara rule, and the fact that
it applies to *instances via their base material*, is not there. An encounter note recording that
has been appended to that ticket.

## One observation this mechanism does NOT explain

From the same session: `M_Bubble` renders correctly on `primitive:"shaderBall"` and returns an
**empty frame** on `primitive:"sphere"`. Force-planing cannot account for that — it would flatten
`shaderBall` too, and it produces a flat quad rather than an empty frame. Recorded here so it is
not lost and not mistakenly folded into this fix; it looks like a separate defect (translucency,
framing, or zoom on a sphere) and has not been investigated.

## Distinct from related tickets, and one DONE verification that does not cover this

**`B-thumbnail-png-writes-jpeg` (DONE, Medium) cannot be cited as evidence that `primitive` is
honoured generally, and its `#4` non-vacuousness proof is narrowed by this ticket.** That entry
closed with an explicit demonstration that the `primitive` override really applies: on
`/Game/DotaBlockout/Materials/M_DotaRock`, "the cylinder and sphere renders differ in 97.3% of a
64x64 luma grid, mean abs diff 80/255". The proof is sound and the assertion it protects (the
scoped restore is not vacuous) still stands — but **`M_DotaRock` is a `UMaterial`**, which is
exactly the case this ticket says works. That verification never touched a material instance, so it
says nothing about the failing case. An encounter note recording that narrowing has been appended
to it; its status is untouched.

- `E-remesh-uniform-echoes-target-not-achieved` (IN-REVIEW, Medium) — the **house precedent for the
  primary ask**, not a duplicate: different verb, same response-shape defect (a field named after
  the result holding the input, on a verb whose whole purpose is to hit the requested value).
- `E-generate-thumbnail-undocumented` (DONE) — documented the UI-domain forcing rule. This is a
  second, undocumented one.
- `B-thumbnail-plane-elevation-edge-on` — the sibling primitive defect from the same session: there
  the caller gets the plane they asked for and the camera makes it useless; here they ask for a
  sphere and get a plane. Same two files (`ThumbnailPreviewOverride.cpp`,
  `AssetWorkflowHandler.cpp`); fix them together while the files are open.
- `E-material-capture-no-thumbnail-pointer` — why there is no fallback when this fires:
  `render.capture_asset_preview` refuses materials entirely.

## Workaround

Capture the **master** material for shape-dependent features and the instance only for colour. Or
try `primitive:"shaderBall"` — it worked where `sphere` did not on `M_Bubble`, though see the
unexplained observation above; on a force-planed instance it will not help, since the force applies
before the primitive is read.

severity rationale: impact=silent wrong data on a normal path — the requested shape is replaced by a flat quad and the response echoes the request, so a caller trusts a picture that cannot show the feature they were checking (a fresnel rim was declared unverifiable on four assets because of it) x reach=the only verb that renders a material at all, and material instances are the normal authoring unit -> High

## History
- `#1-initial-repro` `OPEN` reporter — Found building the Atlantis level (map as forcing function; see host `CLAUDE.md` § "What this project is for"), 2026-08-27, UE 5.8, PinWright at this checkout's HEAD. `asset.generate_thumbnail {primitive:"sphere"}` on `/Game/Atlantis/Materials/M_Fish` rendered a correctly lit sphere with specular highlight and fresnel rim (`scratchpad/M_Fish.png`); the identical request on its plain instance `MI_Fish_Silver` rendered a flat, uniformly lit square (`scratchpad/MI_Fish_Silver.png`), as did `primitive:"cylinder"` on `MI_Fish_Blue` (`scratchpad/MI_Fish_Blue_cyl.png`). Both responses echoed the requested primitive. Counter-example ruling out "all instances": `MI_Coral_Pink` / `MI_Coral_Orange` / `MI_Coral_Violet`, instances made the same way in the same session, DO render as spheres. **The reporter's hypothesis was two uncontrolled variables** (`M_Fish` carries `bUsedWithNiagaraMeshParticles:true` and is one-sided; `M_Coral` neither). Traced statically into UE 5.8 engine source at `C:/UE_5.8/Engine/Source/` — a code read, NOT an experiment: `UMaterial::ShouldForcePlanePreview()` (`Material.cpp:7272-7281`) reads no two-sidedness at all (that variable is ruled out by source) and returns true for `bUsedWithNiagaraMeshParticles` unless `bUserModifiedShape` is set on **its own** `ThumbnailInfo`; `FMaterialThumbnailScene::SetMaterialInterface` (`ThumbnailHelpers.cpp:323-340`) reads `PrimitiveType` off the INSTANCE's ThumbnailInfo at `:323` but computes `bForcePlaneThumbnail` from `GetBaseMaterial()->ShouldForcePlanePreview()` at `:333`, then discards the primitive with `bForcePlaneThumbnail ? TPT_Plane : ...` at `:340`. PinWright sets `bUserModifiedShape` only on the asset it was handed (`ThumbnailPreviewOverride.cpp:277-288`), so for an instance the flag lands on the wrong object and the master's stays false. That accounts for all three observed frames, including the coral counter-example (`M_Coral` trips no term, so nothing is forced). **Still unverified — the control that would settle it** (not run here): duplicate `M_Fish`, clear `bUsedWithNiagaraMeshParticles` only, and request `sphere` on both masters and both instances; or cheaper, set `bUserModifiedShape` on `M_Fish`'s own ThumbnailInfo and re-request `sphere` on `MI_Fish_Silver`. **Load-bearing narrowing recorded on `B-thumbnail-png-writes-jpeg` (DONE)**: its `#4` non-vacuousness proof that the `primitive` override really applies used `/Game/DotaBlockout/Materials/M_DotaRock`, a `UMaterial` — exactly the case that works — so it does not cover the failing case and cannot be cited as evidence that `primitive` is honoured generally; an encounter note was appended there and its status left untouched. Primary ask: report the primitive actually drawn rather than echoing the request (`AssetWorkflowHandler.cpp:847-852` echoes the input under a comment that admits it is "not a promise about the pixels"), following `E-remesh-uniform-echoes-target-not-achieved` and this handler's own `requestedWidth`/`requestedHeight` precedent; secondary, extend the scoped `bUserModifiedShape` write to the base material for an instance. Also recorded and NOT explained by this mechanism: `M_Bubble` renders correctly on `shaderBall` and returns an empty frame on `sphere` — force-planing would flatten shaderBall too, so that is probably a separate defect. Worked around by capturing masters for shape-dependent features and instances for colour; defect untouched.
- `#2-honour-and-report-the-resolved-primitive` `IN-REVIEW` developer — Fixed both halves the ticket asked for. **Secondary (the mechanism):** `FScopedPreviewOverride` now extends its scoped `bUserModifiedShape` write to the BASE material via a new private `ApplyBaseMaterialShapeFlag` (`Handlers/Asset/ThumbnailPreviewOverride.cpp`), because `ThumbnailHelpers.cpp:333` computes `bForcePlaneThumbnail` from `GetBaseMaterial()->ShouldForcePlanePreview()` and `Material.cpp:7270-7281` reads the flag off that object's OWN ThumbnailInfo — so for an instance the plugin's write had been landing on the wrong object. The base is borrowed exactly like the named asset: an unedited master carries no ThumbnailInfo at all, so a transient `USceneThumbnailInfoWithPrimitive` is swapped in for the scope (never the class CDO, which is shared), the camera fields are carried over, and `Restore()` puts the flag and the pointer back on every exit path including the error returns. **Primary (the honest response):** new free function `PinWrightThumbnail::WillForcePlanePreview(Asset, OutReason)` reproduces the engine's gate from the plugin side; the guard resolves it INSIDE its own scope (after the base write, so the defeatable rules are already defeated) and publishes it through `WasForcedToPlane()` / `GetForcePlaneReason()`. `AssetWorkflowHandler.cpp` no longer echoes the request: `primitive` is now the shape actually drawn, with `requestedPrimitive` and `primitiveReason` beside it only when the two disagree — the `requestedWidth`/`requestedHeight` shape this handler already used, and the remedy `E-remesh-uniform-echoes-target-not-achieved` argues for. The `primitive` parameter doc now states the substitution and names the fields that report it. Tests in `Tests/Assets/TestGenerateThumbnail.cpp`: `PinWright.asset.generate_thumbnail.InstanceResolvesSameShapeAsMaster` builds a synthetic master (Niagara mesh-particle usage armed by reflection, since direct member access is `UE_DEPRECATED(5.8)`) plus a plain `UMaterialInstanceConstant` of it, proves the rule is armed on BOTH before any override (non-vacuity), then asserts the master and the instance both resolve to a non-plane shape under the same `primitive:"sphere"` request and that the borrowed master is armed again after the scope; `PinWright.asset.generate_thumbnail.ForcedPlaneIsReportedNotEchoed` drives the dispatcher on a fixture forced by the undefeatable `SetShouldForcePlanePreview` and asserts `primitive:"plane"` + `requestedPrimitive:"sphere"` + a non-empty `primitiveReason`. Counterfactuals: remove the `ApplyBaseMaterialShapeFlag` call and the instance reports forced-to-plane while the master does not (the ticket's exact one-asset-apart divergence); restore the old echo and the second test reads `"sphere"` for a flat quad. The ticket's unrun control was not executed live — this is a static fix against the cited engine source plus the two regression tests; the `M_Bubble` empty-frame-on-sphere observation was left untouched as the ticket asks. No aspect-version bump: `preview.png` in the asset dump is the widget Designer screenshot and never goes through this verb.

- `#3-verified-behaviourally-on-the-built-binary` `DONE` verifier — 2026-08-28, against the
  `b79ba53e` build. `#2` closed with *"The ticket's unrun control was not executed live"*; it has now
  been run live, on the ticket's own assets, and **the pixels were looked at**.

  | request | response `primitive` | what the PNG shows |
  |---|---|---|
  | `M_Fish`, `primitive:"sphere"`, 320x320 | `"sphere"`, no `requestedPrimitive` | a lit SPHERE with a specular highlight and a bright silhouette rim |
  | `MI_Fish_Silver`, `primitive:"sphere"`, 320x320 | `"sphere"`, no `requestedPrimitive` | **a SPHERE** — round, shaded, highlight upper-right, fresnel rim visible at the edge |
  | `MI_Fish_Blue`, `primitive:"cylinder"`, 320x320 | `"cylinder"`, no `requestedPrimitive` | **a CYLINDER** — elliptical top cap, curved side shading, clearly a solid |

  Rows 2 and 3 are the defect. They were *"a FLAT, uniformly lit square of the same grey"* and
  *"the same flat square in blue"*; they are now the requested solids, and the fresnel sheen the
  ticket said was invisible on every instance of `M_Fish` is visible on `MI_Fish_Silver`. The
  substitution is gone, so `requestedPrimitive` / `primitiveReason` correctly do NOT appear.

  **The honest-response half was verified on a case that still forces the plane**, since the fix
  defeats the defeatable rules and the undefeatable one had to be found elsewhere:
  `/Game/ExampleContent/Blueprint_Communication/Materials/M_Crosshair` is UI-domain. Requesting
  `primitive:"sphere"` on it returns `primitive: "plane"`, `requestedPrimitive: "sphere"`, and
  `primitiveReason: "the base material 'M_Crosshair' is a UI-domain material, which the engine
  always draws on a flat camera-facing plane"`. So the response reports the shape drawn and names
  why it differs, instead of echoing the request — the exact behaviour the ticket's title asks for.
  Read-only throughout; no asset was modified. Closing.
