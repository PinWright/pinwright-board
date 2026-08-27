---
id: E-setup-volumetric-fog-single-param
title: "`lighting.setup_volumetric_fog` declares only `viewDistance`, so the fields that actually colour volumetric fog have no typed surface and the only route to them is the raw component write with a known silent-failure path"
status: OPEN
severity: Medium
category: ergonomic
tags: [lighting, setup-volumetric-fog, exponential-height-fog, volumetric-fog, typed-setters, missing-params, docs-gap]
encounters: 1
lastSeen: 2026-08-27T19:12:03+05:00
---

# One parameter for a five-field surface

`lighting.setup_volumetric_fog` registers with **exactly one** optional parameter.
`Plugins/PinWright/Source/PinWright/Private/Handlers/Environment/LightingHandler.cpp:654-657`:

```cpp
REGISTER_RPC_HANDLER("lighting.setup_volumetric_fog", "lighting", "Enable volumetric fog on existing or new ExponentialHeightFog",
    RPC_PARAMS(
        RPC_PARAM_OPT("viewDistance", "number", "Volumetric fog view distance")
    ))
```

The generated wiki page agrees — `Saved/PinWright/wiki/lighting.setup_volumetric_fog.md`
lists `viewDistance` and nothing else.

The handler body then writes `bEnableVolumetricFog = true` (`:690`) and, if given,
`VolumetricFogDistance` (`:694`). Those are the only two fields it can reach. Everything
that determines what volumetric fog *looks like* has no typed surface at all:

- `VolumetricFogAlbedo`
- `VolumetricFogEmissive`
- `VolumetricFogExtinctionScale`
- `VolumetricFogScatteringDistribution`

## Why the missing surface is worse here than in the usual "no typed setter" case

These four are not the long tail — inside `VolumetricFogDistance` they are the *only* live
colour controls. The volumetric integration replaces the height fog's analytic inscattering
within that distance, so `FogInscatteringLuminance` — the field whose name makes it the
obvious first thing an author reaches for — applies only *beyond* it. An author setting
"the fog colour" through the obvious field is writing to a field the renderer is not reading
in the region they are looking at.

That leaves raw `property.set` on `HeightFogComponent0` as the documented-by-elimination
route to the fields that are live. That call is the subject of
`B-property-set-container-empty-change-event` (OPEN, High), which now carries a measured
end-to-end demonstration that a `property.set` write on this exact component reports
`applied: true`, reads back correctly, and never reaches the renderer. So the escape hatch
this gap forces callers onto is itself defective — that is the difference between this
ticket and an ordinary missing-setter request, where the escape hatch merely costs calls.

A route that does work today is `actor.set_component_properties` on
`ExponentialHeightFog_0` / `HeightFogComponent0`, per
`B-set-component-properties-no-change-notification` (IN-REVIEW, High). It is untyped, it
requires knowing the engine field names, and it depends on that ticket's fix being in the
build under test — but it is a working route, which is why this is Medium and not High.

## Docs gap, verified

`grep -rl` for `VolumetricFogAlbedo`, `VolumetricFogEmissive`, `VolumetricFogExtinctionScale`,
`VolumetricFogScatteringDistribution` and `FogInscatteringLuminance` across the **entire**
generated wiki (`Saved/PinWright/wiki/`) returns **zero files**. Not one of the five names
appears anywhere in the wiki — not on `lighting.md`, not on
`lighting.setup_volumetric_fog.md`, not on `post_process.md`.

So the interaction above is undocumented on every surface an author would consult: a reader
cannot learn from the plugin's own docs that `FogInscatteringLuminance` is the wrong field
inside `VolumetricFogDistance`, nor that `VolumetricFogAlbedo` is the right one, nor that
these fields exist at all.

## What it should do

Grow the verb the same way the plugin has already grown two comparable surfaces: keep
`viewDistance`, add optional `albedo` (FLinearColor), `emissive` (FLinearColor),
`extinctionScale` (number) and `scatteringDistribution` (number), write them on the
component, and keep the existing `MarkComponentRenderStateDirty` push at `:698` covering
them. The handler is already open at exactly that point, already holds the component
pointer, and already knows raw writes here need the render-state push — the four writes go
between `:694` and `:698`.

Alongside it, document the `FogInscatteringLuminance` / `VolumetricFogAlbedo` split on
`lighting.setup_volumetric_fog`'s wiki source (`docs/wiki-src/lighting.md`): inside
`VolumetricFogDistance` the volumetric integration replaces the analytic inscattering, so
`FogInscatteringLuminance` and `DirectionalInscatteringLuminance` apply only beyond that
distance. That one paragraph is what turns an absurd-value probe into a lookup.

## Precedent

- `F-post-process-typed-setters` (**DONE**, Low) is the strongest structural precedent —
  the identical argument applied to `FPostProcessSettings`: ~150 UPROPERTYs reachable only
  through raw `property.set`, with a per-field foot-gun the caller had to know about. It was
  accepted and six typed setters shipped. This ticket is that argument on a five-field
  surface instead of a 150-field one, which makes it cheaper, not less valid.
- `F-sky-cloud-reflection-actors` (**DONE**, Medium) is the closest neighbour by subject:
  it added typed spawn verbs for `ASkyAtmosphere` / `AVolumetricCloud` / reflection captures
  with "a small set of the most-tuned UPROPERTYs" each. `AExponentialHeightFog` appears
  nowhere in it — the modern-atmospherics cluster got typed knobs and the fog actor was left
  out. This closes that omission.

## Provenance — read this before acting on it

This claim was **extracted from a defect-log entry (D16) whose other claims were retracted
by their own author** in the same session. That entry originally reported
`FogInscatteringLuminance` as inert under volumetric fog and proposed a cause and a
workaround; a controlled re-test disproved the cause — the field is fully live, and the
real symptom was the render-state staleness now recorded on
`B-property-set-container-empty-change-event`. Its "Cause", "Workaround" and "Suggested fix"
paragraphs are void and are deliberately **not** reproduced here.

What is filed here is only the paragraph that never depended on the retracted cause: the
verb's parameter list, which is a fact about registered source and is verified above at
`LightingHandler.cpp:654-657` independently of any runtime behaviour. It stands whether or
not the retraction had happened. Recorded explicitly so a fixer who finds the retracted
entry does not conclude this ticket went with it.

severity rationale: impact=soft blocker — the goal is reachable, but only by dropping to untyped raw component writes with engine field names, and the route the plugin's own docs lead you to (`property.set`) has a measured silent-failure path, so the workaround is the *other* untyped verb (Medium, not Low as the analogous `F-post-process-typed-setters` was, because there the escape hatch worked) x reach=not every session, but every author doing outdoor or underwater atmospherics, and the docs gap is total — zero of the five field names appear anywhere in the generated wiki -> Medium

## History
- `#1-single-param-and-total-docs-gap` `OPEN` reporter — Found while building the Atlantis example level on host project EAContentExamples58 (map as forcing function; see that project's `CLAUDE.md` § "What this project is for"), 2026-08-27, UE 5.8, PinWright at this checkout's HEAD. Source-verified: `lighting.setup_volumetric_fog` registers exactly one parameter, `RPC_PARAM_OPT("viewDistance", "number", "Volumetric fog view distance")`, at `Handlers/Environment/LightingHandler.cpp:654-657`; the body reaches only `bEnableVolumetricFog` (`:690`) and `VolumetricFogDistance` (`:694`). No typed surface exists for `VolumetricFogAlbedo`, `VolumetricFogEmissive`, `VolumetricFogExtinctionScale` or `VolumetricFogScatteringDistribution`, which inside `VolumetricFogDistance` are the only live colour controls — the volumetric integration replaces the height fog's analytic inscattering there, so `FogInscatteringLuminance` applies only beyond that distance. Docs gap verified by grep: **none** of those five field names appears in any file under `Saved/PinWright/wiki/` (zero matching files), so `lighting.md`, `lighting.setup_volumetric_fog.md` and `post_process.md` all omit both the fields and the interaction. That leaves raw `property.set` on the component as the route the surface implies — the same call `B-property-set-container-empty-change-event` (OPEN, High) now shows measured, on this exact component, to report success and never reach the renderer; the working route is the untyped `actor.set_component_properties`, which is why this is Medium rather than High. Proposed: add optional `albedo`/`emissive`/`extinctionScale`/`scatteringDistribution` between `:694` and the existing `MarkComponentRenderStateDirty` push at `:698`, and document the inscattering split in `docs/wiki-src/lighting.md`. **Provenance:** extracted from defect-log entry D16, whose other claims (cause, workaround, suggested fix) were retracted by their own author after a controlled test disproved them; only the parameter-list claim is filed here, and it is verified directly against registered source independently of that retraction.
