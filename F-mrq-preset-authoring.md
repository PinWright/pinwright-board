---
id: F-mrq-preset-authoring
title: "The only lever on an MRQ render is a preset asset, and no verb in the plugin can make one — mrq.list_presets enumerates, mrq.create_job consumes, nothing authors, so a project with no MPC asset can only render at engine CDO defaults or drop to python.execute"
status: OPEN
severity: Medium
category: feature
tags: [mrq, create_job, list_presets, movie-pipeline, preset, MoviePipelinePrimaryConfig, authoring, python.execute, workaround, reproducibility]
encounters: 1
lastSeen: 2026-09-02T00:00:00Z
---

# Three verbs: one enumerates presets, one consumes one, none makes one

The `mrq` namespace has exactly three registered verbs —
`mrq.create_job` (`Source/PinWright/Private/Handlers/MRQ/MRQHandler.cpp:205`),
`mrq.run_jobs` (`:344`), `mrq.list_presets` (`:466`) — and one line in all three files that
mutates a job's configuration:

    Job->SetConfiguration(Preset);
    -- Source/PinWright/Private/Handlers/MRQ/MRQHandler.cpp:304

`Preset` comes from exactly one place, a read-only load of an asset the caller names:

    Preset = LoadObject<UMoviePipelinePrimaryConfig>(nullptr, *PresetPath);
    -- MRQHandler.cpp:273

`mrq.create_job`'s complete parameter set is `sequencePath`, `levelPath`, `presetPath`, `jobName`
(`:219-222`). `mrq.list_presets` runs one asset-registry query and returns `assetPath` /
`packagePath` / `name` per hit (`:466-497`) — discovery only. A grep across all three files for
`SaveAsset|CreateAsset|SavePackage|NewObject|MarkPackageDirty` returns two hits: the comment at
`:149` and `SetConfiguration` at `:304`. **Nothing in the namespace creates, edits, or persists a
`UMoviePipelinePrimaryConfig`.**

Nor does anything outside it. Grepping every `REGISTER_RPC_HANDLER("<ns>.create...")` in `Source/`
turns up create verbs in 43 namespaces, and not one is a generic create-asset-of-class: `asset.*` offers only `asset.create_folder`
(`Handlers/Asset/AssetManageHandler.cpp:975`) alongside `asset.duplicate` and `asset.import`, both
of which need a source asset that in this case does not exist. So the reachable routes are:

1. **Author the preset by hand in the editor UI** — outside any agent's reach in a headless or
   shared-editor session, and unreproducible.
2. **`python.execute`** (`Handlers/System/PythonExecuteHandler.cpp:116`) — the escape hatch, with
   no typed surface, no readback contract, and none of the refusals the rest of the namespace has.
3. **Render at the engine's CDO defaults** — `Quality` / CRF 20, into a directory nobody chose.

`property.set` (`Handlers/Utility/UtilityPropertyHandler.cpp:1103`) reaches "arbitrary UObjects",
so it can change a property on a config that *already exists*. It cannot create the asset, and it
cannot add a setting object to one — that needs `FindOrAddSettingByClass` on the config, which no
verb exposes. A config with no video output setting has no property to set.

## What that cost, 2026-09-02

Both Atlantis flythrough renders took route 3. `Content/Atlantis/Cine/` holds
`LS_Atlantis_Flythrough.uasset` and no `MPC_*` asset beside it, so each job was queued with no
`presetPath` and every setting it rendered under was an engine default that no artifact on disk
records. Its settings could only be
*inferred* afterwards, and one of them — `EncodingRateControl = Quality`, CRF 20 — is exactly the
default that produced the 1.16 Mbps deliverable in `B-mrq-render-result-omits-bitrate-and-size`.

The eight per-tile presets that fixed it (`/Game/Atlantis/Cine/Video/MPC_Vid_*.uasset`, all created
2026-09-02) were authored outside the namespace, carrying the settings recorded at
`Docs/map/atlantis-video-plan.md:126-129` — deferred pass, PNG, TSR, 4 temporal / 1 spatial sample,
engine warm-up 300, plus five game overrides. Note for whoever picks this up: `Docs/map/OWNERSHIP.md:43`
describes the equivalent Dota2 presets (`/Game/Dota2/Cine/MPC_Dota2_*`) as *"authored over the
PinWright loopback (`sequencer.*` + `mrq.*`)"*. That is accurate for the sequence half and
**inaccurate for the config half** — `mrq.*` has no verb that could have produced them — which is
itself a small sign of the gap: the namespace looks like it authors because it is the only namespace
whose name appears near the asset. (That sentence is in a project doc, not a plugin doc; correcting
it is not this ticket's job, but a fixer should not take it as evidence the capability exists.)

## Why this is not the scoping decision the namespace already made

The `mrq` wiki records a deliberate refusal that looks adjacent and is not:

    **Not implemented: the `minBitsPerPixel` / `minBitrateMbps` param** the ticket also
    proposed. It was judged out of scope, and deliberately so: rewriting a caller's resolved
    config to `VariableBitRate` at a bitrate PinWright derived from a floor would make the
    plugin the one applying settings nobody asked for — this ticket's own defect, inverted.
    Disclosure plus the warning gives a caller everything needed to change one property on
    the preset asset themselves.
    -- Saved/PinWright/wiki/mrq.md:47

That decision is right and this ticket does not reopen it. It refuses *implicit* settings — the
plugin silently overriding what the caller resolved. An **explicit** authoring verb is the
opposite: the caller states every value and the plugin writes exactly that, nothing derived and
nothing inferred. The handler's own framing supports the distinction — *"the encoder settings are
read off it rather than accepted from the caller, since `mrq.create_job` takes no encoder params
at all"* (`MRQHandler.cpp:126-127`) — describes a consequence of the missing verb, not an argument
against one.

The wiki sentence's last clause is where the gap shows: *"everything needed to change one property
on the preset asset themselves"* presupposes a preset asset. When there is none, the disclosure and
the warning name a problem the namespace offers no way to fix.

## Ask

One verb, or two, in the shape the rest of the plugin uses — typed parameters, a readback, and
refusals rather than silent partial success:

- **`mrq.create_preset`** — `assetPath`, plus the settings to install, each named explicitly:
  output (resolution, directory, filename format, frame-rate override), the output writer
  class(es), the video output's rate control / CRF / average / max bitrate, and the anti-aliasing
  sampling block (`B-mrq-config-readback-omits-sampling` asks for the read side of the same
  properties, so the two should share one vocabulary). Saves to disk and verifies against the file
  the way the rest of the plugin's asset writers do; returns the same `preflight` block
  `mrq.create_job` already builds (`MRQHandler.cpp:155`, `MRQArtifactReport.cpp` `BuildPreflightReport`),
  so what was written is stated in the same words a later `create_job` will use.
- **`mrq.set_preset_settings`** — the same fields against an existing asset, for the "change one
  property" case the wiki sentence assumes is easy. Refuse a setting class the config does not
  carry unless asked to add it, rather than silently creating one.

Both are strictly additive: `presetPath` keeps meaning what it means, an omitted `presetPath` keeps
queueing deliberately unconfigured, and no existing call shape changes.

Minimum viable alternative if a full authoring surface is judged too large: `mrq.create_preset`
that only creates an asset with an output setting and one named video output class, leaving the
rest to `property.set` on the resulting subobjects. That alone removes route 3 as the default for
a project that has never made a preset.

## Related

- `B-mrq-render-result-omits-bitrate-and-size` (High, IN-REVIEW) — the defect a missing preset
  caused, and the source of the scoping note above. Its landed disclosure tells a caller the
  encode is unbounded below; this ticket is about having somewhere to put the fix.
- `B-mrq-config-readback-omits-sampling` — the read side. An authoring verb and a readback should
  name the same properties.
- `F-mrq-render-queue` (Low, DONE) — the origin ticket that designed `presetPath` + `list_presets`.
  The authoring gap starts here and was never filed as one.

## History
- `#1-no-verb-authors-a-config` `OPEN` reporter — "Filed 2026-09-02 from the Atlantis showcase video. The `mrq` namespace registers three verbs (`MRQHandler.cpp:205`, `:344`, `:466`); its only config mutation is `Job->SetConfiguration(Preset)` (`:304`) from a read-only `LoadObject` (`:273`); `mrq.create_job` takes only sequencePath/levelPath/presetPath/jobName (`:219-222`); `mrq.list_presets` enumerates (`:466-497`). Grep of all three files for `SaveAsset|CreateAsset|SavePackage|NewObject|MarkPackageDirty` returns one comment and `SetConfiguration`. No generic create-asset-of-class verb exists anywhere in the plugin (`asset.*` offers `create_folder` only, `AssetManageHandler.cpp:975`), so the routes are the editor UI, `python.execute` (`PythonExecuteHandler.cpp:116`), or engine CDO defaults. Both flythrough renders took the third: `Content/Atlantis/Cine/` holds the sequence and no `MPC_*`, so their settings exist nowhere on disk. Distinct from the namespace's deliberate refusal at `wiki/mrq.md:47`, which rejects PinWright applying settings nobody asked for; an explicit authoring verb is the opposite and the same sentence's 'change one property on the preset asset themselves' presupposes an asset that here did not exist."
