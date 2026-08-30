---
id: B-capture-docs-prescribe-retired-grass-wait
title: "render.md still prescribes editor.set_camera + a 60-150 s wait + repeat-until-stable captures for a grass amortization the capture's own settle path force-syncs past — the retired ritual sits 75 lines below the paragraph documenting the settle that retired it, and was re-propagated into a new page after the fix landed"
status: OPEN
severity: Medium
category: bug
tags: [render, capture_open_level, docs, wiki-src, landscape, grass, vegetation, stale-guidance, self-contradicting-page, set_camera, viewport-state, forcesync]
encounters: 1
lastSeen: 2026-08-30T19:00:00+05:00
---

# The page documents the settle at the top and prescribes the wait it removed at the bottom

`render.capture_open_level` force-syncs the landscape-grass build for its own eye position before
the first draw, so the engine's per-frame grass budget never applies to it. The `render` namespace
page says so at `:52-56` — and then at `:127-129` tells the caller that grass repopulates on a
tick budget of one component per frame and that the way to get a usable capture is to move the
camera with `editor.set_camera`, wait, and capture repeatedly until two frames agree.

Both paragraphs are on `Docs/wiki-src/render.md`. A caller who reads the page top to bottom is
told to spend 60-150 s per pose defeating something that costs 18-51 ms.

## The two paragraphs

**What is true now** — `Docs/wiki-src/render.md:54` (generated
`Saved/PinWright/wiki/render.check-the-view-mode-before-you-trust-a-capture.md:38`):

> Every capture now hands its own **measured** eye position to
> `ULandscapeSubsystem::RegenerateGrass(bFlushGrass: false, bForceSync: true, [cameraLocation])`
> before the first draw. No viewport camera is moved to do it and none is left moved: the
> location travels as data [...] `grass.cameraLocation` equals the response's top-level
> `cameraLocation` on every level capture; a mismatch, or `builtForPose: false`, means these
> pixels show somebody else's grass.

**What the same overlay still says 73 lines later** — `Docs/wiki-src/render.md:127` (generated
`Saved/PinWright/wiki/render.a-frame-can-be-contaminated-by-state-this-capture-does-not.md:16`).
Note that the generator splits the overlay by heading, so these two paragraphs land on **different**
generated pages: a client that reads only the frame-contamination page gets the retired ritual with
no sight of the settle paragraph at all.

> **View-driven geometry is built around the *persistent* viewport camera, on a tick budget, and
> `warmup.settled` does not wait for it.** Landscape grass is the common case: the engine builds
> it around the camera locations the streaming manager saw on tick, and `grass.MaxCreatePerFrame`
> defaults to **1**, so a large cull radius costs hundreds of frames to repopulate after the
> camera teleports.

and the instruction it hangs off, `Docs/wiki-src/render.md:129` (generated `:18`):

> Move the camera with `editor.set_camera`, **wait**, then capture repeatedly at the fixed pose
> until two consecutive frames agree in `imageStats.meanLuminance` and `sizeBytes` - and compare a
> pair only when both members are built.

The same ritual is repeated in full on a second page, `Docs/wiki-src/vegetation-authoring.md:272-281`
(generated `Saved/PinWright/wiki/vegetation-authoring.md:274-283`), under the heading *"Grass also
takes 60-150 s to rebuild after a camera move while every health signal reads clean."*

## The cvar is real and is the engine's — the wrong part is the consequence drawn from it

Recorded explicitly because the previous statement of this finding was mis-stated in exactly this
place, on `B-capture-open-level-pose-params-photograph-stale-grass` `#4`: *"`grass.MaxCreatePerFrame`
appears nowhere in plugin source [...] so nothing reads or reports it"*. The first clause is true
and the second is not. **The engine reads it, and it does throttle — just not this path.**

- Declared at `C:/UE_5.8/Engine/Source/Runtime/Landscape/Private/LandscapeGrass.cpp:181-185`:
  `static int32 GGrassMaxCreatePerFrame = 1;` bound to `TEXT("grass.MaxCreatePerFrame")` with the
  help text *"Maximum number of Grass components to create per frame"*. Default **1**, exactly as
  the docs say.
- The budget is taken at `:2873` (`int32 GrassMaxCreatePerFrame = GGrassMaxCreatePerFrame;`,
  multiplied by `GGrassCreationPrioritizedMultipler` at `:2877`) and spent at `:3121`:

```cpp
if (!bRebuildForBoxes && !bForceSync && (InOutNumCompsCreated >= GrassMaxCreatePerFrame || AsyncFoliageTasks.Num() >= MaxTasks))
{
    continue; // one per frame, but we still want to touch the existing ones and we must do the rebuilds because we changed the tag
}
```

`!bForceSync` is the second conjunct. **With `bForceSync` true the gate cannot fire.**

## PinWright's settle sets exactly that flag

`Source/PinWright/Private/Handlers/Render/LandscapeGrassSettle.cpp:115-116`, re-derived at plugin
HEAD `1a9e5778`:

```cpp
Subsystem->RegenerateGrass(/*bInFlushGrass=*/false, /*bInForceSync=*/true,
    TArrayView<FVector>(CaptureCameras));
```

with the reason already in the comment directly above it (`:110-112`): *"bInForceSync true so the
instances exist when this function returns instead of a few frames later."*

The flag reaches the gate unchanged: `ULandscapeSubsystem::RegenerateGrass`
(`C:/UE_5.8/Engine/Source/Runtime/Landscape/Private/LandscapeSubsystem.cpp:613`) calls
`Proxy->UpdateGrass(CameraLocations, bInForceSync)` at `:669`, into
`ALandscapeProxy::UpdateGrass` (`LandscapeGrass.cpp:2841`, forwarding to the counted overload at
`:2848`), whose `bForceSync` is the one tested at `:3121`. The engine force-syncs the same way for
its own full rebuild at `LandscapeGrass.cpp:1020`.

`SettleGrassForCapturePose` runs on both capture paths that can see a landscape —
`PreviewViewportCaptureUtils.cpp:2151` (the level capture, i.e. `render.capture_open_level`) and
`OrthoTileCaptureUtils.cpp:681` — unconditionally, with no opt-in parameter. So there is no
`render` verb reachable by a caller for which the documented wait is the right advice.

## Measured

**13:32 build, plugin commit `d8f1bc32`** — an ancestor of HEAD;
`Handlers/Render/LandscapeGrassSettle.cpp` and `Docs/wiki-src/render.md` are byte-identical at
`d8f1bc32` and `1a9e5778`. Source of record, read directly:
`X:/src/unreal/EAContentExamples58/Docs/map/vegetation-agent-brief.md:805-818` § *"Retracted: the
whole grass-capture ritual"*, project commit `7d629ad9`.

> a **28,000 uu teleport from zone A to the zone-E scree, with no `editor.set_camera` and no
> wait**, returned a fully built carpet on the FIRST capture — `builtForPose:true`,
> `pendingComponents:0`, components 137 -> 186

Independently, on the same build, `B-capture-open-level-pose-params-photograph-stale-grass` `#4`
took three captures at (0,0,0), (18000,18000,2000) and (-18000,-18000,2000) — up to ~50,900 uu
apart — and recorded `buildMs` **50.9 / 18.2 / 32.8**,
`builtForPose: true` on all three, with the persistent viewport camera read back through
`python.execute` and confirmed **not to have moved**. Milliseconds, not hundreds of frames.

## Why this is more than wasted minutes

1. **The prescribed step mutates shared state the fix deliberately stopped touching.**
   `editor.set_camera` moves the persistent Level Editor viewport camera and leaves it moved.
   `render.md:54` states the settle's design property in as many words — *"No viewport camera is
   moved to do it and none is left moved"* — so the docs are telling the caller to reintroduce, by
   hand, the side effect the verb was changed to avoid. In an editor shared by several agents that
   camera is another caller's state.
2. **It teaches a heuristic that is now inverted.** *"capture repeatedly until two consecutive
   frames agree"* was correct advice against an amortised build. Against a force-synced one the
   repeat is pure cost, and the fields that answer the question directly — `grass.builtForPose`,
   `grass.pendingComponents`, `grass.settled`, `grass.buildMs` — are documented 75 lines up the
   same page and go unread.
3. **It was re-propagated after the fix, not merely left behind.** The
   `vegetation-authoring.md` copy was added by plugin commit `1e238f87` *"Add the vegetation
   authoring workflow"*, which is a **descendant** of `d8f1bc32`, the commit that added
   `Handlers/Render/LandscapeGrassSettle.{h,cpp}` (`git log --diff-filter=A`). So a new page
   written after the settle shipped copied the retired ritual onto itself. That is the argument
   for fixing the source paragraph rather than only the page a reader happened to hit.

## Fix

`Docs/wiki-src/render.md:127-129` and `Docs/wiki-src/vegetation-authoring.md:272-281`. Keep the
*mechanism* — it is true, and it is why `warmup.settled` cannot be read as scene readiness — and
replace the *prescription*:

- say that `grass.MaxCreatePerFrame` governs the editor's own world-tick grass update and that the
  capture verbs do not go through it, because `SettleGrassForCapturePose` passes
  `bForceSync: true` (`LandscapeGrassSettle.cpp:115`), which short-circuits the gate at
  `LandscapeGrass.cpp:3121`;
- replace `set_camera` + wait + repeat-until-stable with: read `grass.builtForPose`,
  `grass.pendingComponents` and `grass.settled` off the response, and treat a `grassWarning` as
  the only reason to re-shoot;
- keep the `warmup.settled` warning itself, which is a separate and still-true point about the
  *frame* settle, and is what the paragraph was originally written to make.

Everything else on `render.md:127` — the measured luminance ladder, the "two shots agreeing is not
evidence" observation — remains historically accurate for the pre-settle behaviour and should be
marked as such rather than deleted, so a reader on an older build is not misled the other way.

## Noted here, deliberately NOT filed

`Docs/wiki-src/render.md:56` tells the caller: *"`settled: true` with `instances: 0` is a
measurement — that ground really is bare."* `B-capture-grass-instances-always-zero` (OPEN, High)
establishes that `grass.instances` is **structurally always 0** for landscape grass, so that
sentence promotes a known-broken field to a prescribed test. That is that ticket's defect wearing
a documentation face, not a second grass-ritual defect, and its fixer will be editing this exact
sentence; folding it in here would put two tickets on one field. Recorded per the precedent on
`B-pcg-connect-pins-silently-replaces-edge` § *Noted here, deliberately NOT filed*. Whoever works
`B-capture-grass-instances-always-zero` should sweep `render.md:56` in the same change.

## Structure: why this is its own ticket

Filed alongside `B-layer-paint-doc-claims-blend-group-write` rather than merged with it. Both are
the same shape — a shipped change rewrote one part of a wiki page and left the older, now-false
part standing on the same page — but they share no verb, no namespace, no overlay file and no
fixer, and they rate differently. The board's one multi-defect precedent,
`B-bulk-rename-docs-behaviour-mismatch`, bundles three contradictions in **one verb, one function,
one contiguous block**; this set is the opposite, and bundling would force one severity onto items
the picker should order separately.

## Cross-links

- `B-capture-open-level-pose-params-photograph-stale-grass` (**DONE**, High, `encounters: 2`) —
  owns the defect the settle fixed and verified it on this build at `#4`. Not reopened: its
  subject is whether the pose drives the build, which is verified three ways there, and the code
  is correct. This ticket is the documentation that was never swept, which no status on that
  ticket covers. Its `#4` also contains the mis-statement corrected above; the correction is
  recorded here rather than appended there because that ticket is closed and the sentence is not
  what its close rests on.
- `B-capture-grass-instances-always-zero` (OPEN, High) — see *Noted here* above.
- `B-ortho-capture-renders-no-landscape-grass` — the ortho path, which runs the same settle from
  `OrthoTileCaptureUtils.cpp:681`; whatever that ticket concludes, the wait ritual is no more
  applicable there than here.

## Same shape as

Not the session's recurring class from `B-foliage-paint-does-no-ground-projection` § *Same shape
as* — no number here is wrong, and the response carries `builtForPose` / `pendingComponents` /
`buildMs`, which are exactly the deciding numbers that class complains are missing. The defect is
that the documentation tells the caller not to read them. Said explicitly so it is not filed under
that class by habit.

## Severity

**Medium**, reached as Low impact bumped once for reach — both halves argued rather than asserted.

*Impact: Low.* The rubric's Low band is *"pure friction. Docs, discoverability, naming [...] or
cosmetic"*, and no call returns wrong data because of this. A caller who ignores the page gets a
correct capture on the first shot.

*Reach: bump up one.* The rubric: *"if the affected method runs in almost every session, bump up
one level"*. `render.capture_open_level` is the plugin's only level-capture verb and the surface
every visual claim is checked on, and `render.md` is the namespace page a caller reads **before**
capturing — the ritual is at the point of first contact, not in a corner. The cost when it is
followed is 60-150 s per pose plus a mutation of the shared viewport camera, repeated per capture.
Low + reach bump = **Medium**.

**High argued and declined.** High is silent wrong data on a normal path. Nothing here returns
wrong data; the response is honest and carries the fields that settle the question. What is wrong
is guidance, which is the Low band's subject — the bump comes from reach, not from re-classing the
impact.

**`encounters` is 1** and is not a severity input: one observation, on the 13:32 build, plus the
independent three-pose measurement recorded on the DONE ticket's `#4` from the same build and
session.

## History
- `#1-docs-prescribe-a-wait-the-settle-force-syncs-past` `OPEN` reporter — `Docs/wiki-src/render.md:127` (generated `render.a-frame-can-be-contaminated-by-state-this-capture-does-not.md:16`) still tells the caller that landscape grass is built around the persistent viewport camera on a tick budget and that `grass.MaxCreatePerFrame` defaults to 1 "so a large cull radius costs hundreds of frames to repopulate after the camera teleports", and `:129` (generated `:18`) prescribes `editor.set_camera`, a wait, and repeat-until-stable captures on that basis; `Docs/wiki-src/vegetation-authoring.md:272-281` (generated `vegetation-authoring.md:274-283`) repeats it under "Grass also takes 60-150 s to rebuild after a camera move". The **same page** documents the settle that removed it at `:52-56`, 73 lines above. **The premise as first stated was wrong and is corrected here**: `B-capture-open-level-pose-params-photograph-stale-grass` `#4` recorded that `grass.MaxCreatePerFrame` "appears nowhere in plugin source [...] so nothing reads or reports it". The first clause is true — a whole-repo grep of the plugin (excluding `.git`) returns exactly the two doc lines above and nothing else — but the second is false: **the cvar is the engine's and the engine reads it.** Re-derived at UE 5.8: declared `static int32 GGrassMaxCreatePerFrame = 1;` bound to `grass.MaxCreatePerFrame` at `LandscapeGrass.cpp:181-185`, taken as the frame budget at `:2873` (scaled at `:2877`) and spent at the gate `:3121` — `if (!bRebuildForBoxes && !bForceSync && (InOutNumCompsCreated >= GrassMaxCreatePerFrame || AsyncFoliageTasks.Num() >= MaxTasks)) continue;`. The correct statement is therefore **not** "a documented mechanism that does not exist" but "the docs prescribe a wait for an amortization this plugin's own settle path does not go through": `!bForceSync` is the second conjunct of that gate, and `SettleGrassForCapturePose` passes `bForceSync` true — `Subsystem->RegenerateGrass(/*bInFlushGrass=*/false, /*bInForceSync=*/true, TArrayView<FVector>(CaptureCameras))` at `LandscapeGrassSettle.cpp:115-116` at HEAD `1a9e5778`, its own comment at `:110-112` saying why. The flag reaches the gate unchanged: `ULandscapeSubsystem::RegenerateGrass` (`LandscapeSubsystem.cpp:613`) -> `Proxy->UpdateGrass(CameraLocations, bInForceSync)` (`:669`) -> `ALandscapeProxy::UpdateGrass` (`LandscapeGrass.cpp:2841`, counted overload `:2848`) -> the test at `:3121`; the engine force-syncs the same way at `LandscapeGrass.cpp:1020`. The settle runs unconditionally on both capture paths that can see a landscape (`PreviewViewportCaptureUtils.cpp:2151`, `OrthoTileCaptureUtils.cpp:681`), with no opt-in, so no reachable `render` verb makes the documented wait correct. **Measured on the 13:32 build (`d8f1bc32`, ancestor of HEAD; `LandscapeGrassSettle.cpp` and `Docs/wiki-src/render.md` byte-identical at both commits), source of record `X:/src/unreal/EAContentExamples58/Docs/map/vegetation-agent-brief.md:805-818` at project commit `7d629ad9`, read directly:** a 28,000 uu teleport with no `editor.set_camera` and no wait returned a fully built carpet on the FIRST capture — `builtForPose:true`, `pendingComponents:0`, components 137 -> 186. Corroborated independently on the same build by that DONE ticket's `#4`: three poses, `buildMs` 50.9 / 18.2 / 32.8, `builtForPose` true on all three, with the persistent viewport camera read back through `python.execute` and confirmed unmoved. Milliseconds against the documented hundreds of frames. Three costs beyond the wasted time, argued rather than asserted: the prescribed `editor.set_camera` moves and leaves moved the shared persistent viewport camera, which `render.md:54` says in as many words the settle exists to avoid; the repeat-until-stable heuristic displaces the fields that answer the question directly (`builtForPose` / `pendingComponents` / `settled` / `buildMs`), documented 75 lines up the same page; and the ritual was **re-propagated after the fix** — `Docs/wiki-src/vegetation-authoring.md` was added by plugin commit `1e238f87`, a descendant of `d8f1bc32` (the commit that added `LandscapeGrassSettle.{h,cpp}`, per `git log --diff-filter=A`), so a page written after the settle shipped copied the retired advice onto itself. Dedup: board grep for `MaxCreatePerFrame` returns only `B-capture-open-level-pose-params-photograph-stale-grass` (DONE) — not reopened, because its subject is whether the pose drives the build and the code is right; grep for `settle` / `builtForPose` / `capture_open_level` + grass additionally returns `B-capture-grass-instances-always-zero` (OPEN, High) and `B-ortho-capture-renders-no-landscape-grass`, both checked and distinct. **One adjacent finding recorded and deliberately NOT filed**, per the `B-pcg-connect-pins-silently-replaces-edge` precedent: `Docs/wiki-src/render.md:56` instructs the caller to read `settled: true` with `instances: 0` as evidence the ground is bare, and `B-capture-grass-instances-always-zero` proves that field is structurally always 0 — that is that ticket's defect in documentation form and its fixer will be editing the same sentence, so it is routed there rather than absorbed here. Rated **Medium** = Low impact (pure documentation friction; no call returns wrong data) bumped one for reach under the README's every-session rule, since `render.capture_open_level` is the only level-capture verb and `render.md` is read before capturing; High declined because nothing on the wire is wrong. Filed as its own ticket rather than merged with `B-layer-paint-doc-claims-blend-group-write`, whose shape it shares but whose verb, namespace, overlay file, fixer and rating it does not.
