---
id: F-generic-volume-creator-with-brush-geometry
title: "volume.* has 22 typed volume-creating verbs and no generic one, so any AVolume subclass the table does not enumerate — APCGVolume above all — has no brush-geometry-bearing placement path; actor.spawn is not the answer because spawning an ABrush subclass produces a zero-extent, collisionless phantom"
status: OPEN
severity: Medium
category: feature
tags: [volume, brush, generic-creator, pcg, pcg-volume, avolume, class-parameter, hardcoded-class, missing-verb, brush-geometry]
encounters: 1
lastSeen: 2026-08-29
---

# Twenty-two hardcoded classes, and no way to name a twenty-third

The `volume` namespace registers 26 methods, all in
`Plugins/PinWright/Source/PinWright/Private/Handlers/Volume/VolumeHandler.cpp`. Counted from the
generated wiki directory (`Saved/PinWright/wiki/volume.*.md`, 26 pages) and cross-checked against
the `REGISTER_RPC_HANDLER("volume.…")` sites in that file:

- **16 typed `create_*` verbs** — `create_trigger_volume`, `create_trigger_sphere`,
  `create_trigger_capsule`, `create_blocking_volume`, `create_kill_z_volume`,
  `create_pain_causing_volume`, `create_physics_volume`, `create_audio_volume`,
  `create_reverb_volume`, `create_post_process_volume`, `create_cull_distance_volume`,
  `create_precomputed_visibility_volume`, `create_lightmass_importance_volume`,
  `create_nav_mesh_bounds_volume`, `create_nav_modifier_volume`, `create_camera_blocking_volume`.
- **6 typed `add_*` verbs** — the same idea anchored to an existing actor's location
  (`add_trigger_volume`, `add_blocking_volume`, `add_kill_z_volume`, `add_physics_volume`,
  `add_cull_distance_volume`, `add_post_process_volume`).
- 4 non-creating verbs — `get_volumes_info`, `set_volume_bounds`, `set_volume_extent`,
  `set_volume_properties`.

Twenty-two creation verbs, each with its volume class written into the handler. None takes a class
parameter. A volume class not on that list of sixteen has no creation path at all.

**`APCGVolume` is the case that makes this concrete.** It is the canonical host for a PCG graph — the
actor whose bounds define the generation domain — and it appears **nowhere** in
`Plugins/PinWright/Source`: zero occurrences of `APCGVolume` or `PCGVolume` across every module,
tests included. So the plugin ships an entire `pcg` namespace (17 methods) and no first-class way to
place the actor those graphs are normally hosted on.

## Rebutting `actor.spawn` as the generic path, because that is the board's stated position

`E-rpc-cull-151-record` (DONE) records the 151-method cull and its replacement table. Under
`**volume** (2)` at `:217`, the two entries are:

    - `volume.remove_volume` -> `actor.delete`
    - `volume.create_trigger_box` -> `actor.spawn`

`:218-219`. So the board has already ruled once that `actor.spawn` is the generic volume creator, and
this ticket will be culled on that precedent unless the rebuttal is concrete. It is:

**`actor.spawn` of an `ABrush`-derived class produces an actor with no brush geometry.** A raw
`World->SpawnActor` on an `ABrush` subclass leaves `Brush` (the `UModel`) null, `Polys` null and
`BrushComponent->Brush` unwired; `UEditorBrushBuilder::EndBrush` then early-returns success without
writing geometry. The result is bounds extent `(0,0,0)` and no collision, reported as success. Two
tickets carry this, both measured rather than reasoned:

- `B-blocking-volume-no-brush-geometry` (IN-REVIEW, **Critical**, `encounters: 2`) — the original
  diagnosis and the fix.
- `B-spawned-volumes-have-no-brush-geometry` (OPEN, **High**, filed this session) — measured live on
  two verbs, `foliage.create_procedural` and `actor.spawn`, both producing *"an actor with bounds
  extent `(0,0,0)` and no collision"* and both reporting success.

**So the ask is not "there is no generic creator" — it is "there is no brush-geometry-bearing
generic creator", and a fix that merely aliases `actor.spawn` would reproduce a Critical defect.**
That distinction is the whole ticket. `actor.spawn` genuinely is the right generic path for an
ordinary `AActor`; it is wrong for `ABrush` precisely because a brush volume's shape is not its
transform, and nothing in the spawn path builds it.

For `APCGVolume` the consequence is not cosmetic. `pcg.generate`
(`Source/PinWrightPCG/Private/Handlers/PCG/PCGGenerateHandler.cpp:109`) takes any placed actor and
adds a `UPCGComponent` if absent, so a caller *can* reach PCG without a volume — but a PCG graph's
sampling domain is its host actor's bounds, and a host spawned through `actor.spawn` has bounds
extent `(0,0,0)`. The generic path therefore does not fail loudly; it hands PCG a degenerate domain.

## The function a generic creator routes through already exists

`VolumeHelpers::BuildBoxBrushGeometry` — `VolumeHandler.cpp:90`. Its own header comment states the
mechanism this ticket depends on:

> A freshly `World->SpawnActor<ABrush-derived>()` volume has a null Brush UModel, so
> `UEditorBrushBuilder::EndBrush` early-returns success without writing any geometry … The result is
> a phantom volume that reports success but has no geometry and no collision. This mirrors the
> engine's own `UActorFactory::CreateBrushForVolumeActor` …

`:82-89`. It does `PreEditChange` → `Brush = NewObject<UModel>` + `Initialize` +
`Polys = NewObject<UPolys>` → wire `BrushComponent->Brush` → `UCubeBuilder` → `FBSPOps::csgPrepMovingBrush`
→ `PostEditChange` (`:98-127`).

**It has exactly three production call sites, all in that same file**, verified by grepping the whole
`Plugins/PinWright/Source` tree: `CreateBoxBrushForVolume` (`:133`), `CreateSphereBrushForVolume`
(`:138`), `CreateCapsuleBrushForVolume` (`:143`). The only other hits anywhere are two comment
references in `Private/Tests/World/TestVolumeHandlers.cpp` (`:554`, `:808`). Nothing outside
`VolumeHandler.cpp` calls it — which is exactly the finding
`B-spawned-volumes-have-no-brush-geometry` is built on, and it is also why a generic creator is
cheap: the hard part is written, tested and confined to one file.

## Proposed verb shape

**`volume.create`** — `{volumeClass, name?, location?, rotation?, extent?, shape?, properties?}` →
the same response the sixteen typed verbs return.

- `volumeClass` resolved by the shared class-resolution path and gated on
  `IsChildOf(AVolume::StaticClass())` — the way `actor.add_component` gates on `UActorComponent`
  (`Handlers/Actor/ComponentHandler.cpp:75-77`) and the way `F-pcg-create-graph-class-parameter`
  proposes gating a `graphClass`.
- Routes through `CreateBoxBrushForVolume` / `CreateSphereBrushForVolume` /
  `CreateCapsuleBrushForVolume` per `shape`, so the brush is built by the one function that knows
  how, and a class the table never enumerated gets the same geometry a `BlockingVolume` gets.
- `properties` applied post-construction with per-key warnings, matching
  `actor.add_component`'s contract (`ComponentHandler.cpp:128-140`) — that is what makes a generic
  verb usable for a class with typed settings nobody wrote a verb for.

This does **not** propose deleting the sixteen typed verbs. They carry class-specific defaults and
echoes that a generic verb cannot; the ask is a floor under the classes nobody enumerated, not a
replacement for the ones somebody did.

**Severity: Medium, argued.** Impact class is the rubric's *"High or Medium: hard blocker with no
workaround (a stub, a missing verb, or rejecting valid input)"* — a missing verb, and the nominal
workaround is a Critical defect rather than a workaround. It lands on **Medium** rather than High
for one reason, stated so a reviewer can disagree with it directly: `python.execute` reaches
`UActorFactory` / the brush-build sequence and is a real, if unpleasant, route, so a caller is not
absolutely stuck — the rubric's Medium is *"Doable, but only via a documented workaround, a source
dive, or many extra calls"*, and this is a source dive into the engine's own factory. It is
explicitly **not** High-as-silent-wrong-data: that half of the problem is already owned, correctly
and at Critical/High, by `B-blocking-volume-no-brush-geometry` and
`B-spawned-volumes-have-no-brush-geometry`. Filing this at High would double-count their severity on
a ticket that asks for a *capability*, not for a lie to stop. **Reach modifier declined in both
directions, and named:** placing a volume is common enough that `volume.*` carries 22 verbs for it,
but no single one of them — and certainly not the generic case — runs in almost every session; nor
is it a rare edge path, since the PCG workflow the plugin ships 17 verbs for normally begins here.
Medium stands unmodified.

## Related

- `F-pcg-create-graph-class-parameter` (IN-REVIEW, High) — the nearest sibling and the same shape
  exactly: *"One hardcoded `NewObject` is the whole distance between PinWright and native tree
  authoring"* — `pcg.create_graph` constructs one `UPCGGraph` and cannot be told otherwise
  (`Source/PinWrightPCG/Private/Handlers/PCG/PCGGraphCreate.cpp:73-76`), so a `UPCGGraph` subclass is
  unreachable for want of one optional class parameter. This ticket is that argument one level up:
  the graph has no class parameter and neither does its host. A fixer landing both gives the PCG
  workflow a reachable start and a reachable graph; landing one leaves the other as the blocker.
- `B-spawned-volumes-have-no-brush-geometry` (OPEN, High) — the measurement this ticket's rebuttal
  rests on, and the ticket that establishes `BuildBoxBrushGeometry`'s three-call-site scope.
- `B-blocking-volume-no-brush-geometry` (IN-REVIEW, Critical) — the original diagnosis;
  `#3-fix-brush-model-init` is where `BuildBoxBrushGeometry` came from and why it is safe to route a
  new verb through it.
- `E-rpc-cull-151-record` (DONE) — the `volume.create_trigger_box -> actor.spawn` precedent at
  `:219` that this ticket must and does answer.

## History
- `#1-no-brush-bearing-generic-creator` `OPEN` reporter — Counted the namespace myself rather than
  quoting a figure: 26 `volume.*.md` pages in `Saved/PinWright/wiki/`, matching 26
  `REGISTER_RPC_HANDLER("volume.…")` sites in `VolumeHandler.cpp`, of which 16 are `create_*` and 6
  are `add_*` — 22 volume-creating verbs, every one with its class hardcoded, none taking a class
  parameter. Confirmed `APCGVolume` / `PCGVolume` appear **nowhere** in `Plugins/PinWright/Source`.
  Rebutted the `actor.spawn` precedent (`E-rpc-cull-151-record:217-219`) against two measured
  tickets: an `ABrush` subclass spawned that way lands with bounds extent `(0,0,0)` and no collision
  while reporting success (`B-spawned-volumes-have-no-brush-geometry`, High, measured live on two
  verbs; `B-blocking-volume-no-brush-geometry`, Critical). Scoped the ask to
  brush-geometry-bearing generic creation and stated that an alias to `actor.spawn` would reproduce
  the Critical defect. Verified `VolumeHelpers::BuildBoxBrushGeometry` at `VolumeHandler.cpp:90`
  with exactly three production call sites — `:133`, `:138`, `:143`, all in that file — by grepping
  the whole Source tree; the only other occurrences are two comments in
  `Private/Tests/World/TestVolumeHandlers.cpp` (`:554`, `:808`). Noted that `pcg.generate`
  (`PCGGenerateHandler.cpp:109`) will attach a `UPCGComponent` to any placed actor, so the failure
  mode for PCG is a degenerate zero-extent sampling domain rather than a refusal.
