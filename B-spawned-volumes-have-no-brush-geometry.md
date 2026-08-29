---
id: B-spawned-volumes-have-no-brush-geometry
title: "foliage.create_procedural and actor.spawn spawn ABrush-derived volumes without ever initializing the brush, so the actor lands with bounds extent (0,0,0) and no collision while both verbs report success — the in-tree fix for B-blocking-volume-no-brush-geometry is scoped to VolumeHandler.cpp and neither verb reaches it"
status: OPEN
severity: High
category: bug
tags: [volume, brush, procedural-foliage, pcg, create_procedural, actor-spawn, silent-false-success, zero-extent, collision, spawn-path]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
---

# The brush fix exists, is correct, and sits in a file neither of these two spawn paths opens

`B-blocking-volume-no-brush-geometry` (IN-REVIEW, **Critical**, `encounters: 2`) diagnosed this
exact mechanism and its `#3-fix-brush-model-init` entry landed the correct fix:
`VolumeHelpers::BuildBoxBrushGeometry` at
`Plugins/PinWright/Source/PinWright/Private/Handlers/Volume/VolumeHandler.cpp:90`. **That fix's
scope is the three brush helpers in that one file, and it has exactly three production call
sites — all of them in `VolumeHandler.cpp`.** Two *other* verbs, in two *other* namespaces, spawn
`ABrush`-derived volume actors and never initialize the brush. Both were measured live this
session; both produce an actor with bounds extent `(0,0,0)` and no collision; both report success.

A fixer picking this up should read `B-blocking-volume-no-brush-geometry` first — the analysis and
the repair are already written there and this ticket asks for nothing new, only for two more
callers to be routed through it.

## The two spawn paths

### 1. `foliage.create_procedural` — scale is written onto geometry that does not exist

`Handlers/Environment/FoliageHandler.cpp:1489-1491`:

```cpp
  AProceduralFoliageVolume *Volume = Cast<AProceduralFoliageVolume>(
      SpawnActorInActiveWorld<AActor>(AProceduralFoliageVolume::StaticClass(),
                                      Location, FRotator::ZeroRotator, Name));
```

A raw `SpawnActor` on an `ABrush` subclass — no `Brush` UModel, no `Polys`, no
`BrushComponent->Brush`. The requested `size` is then applied as an **actor scale**
(`:1499`), under a comment (`:1497-1498`) that states the premise this ticket disproves:

```cpp
  // AProceduralFoliageVolume uses ABrush with default extent of 100 units (half-size)
  // Scale = desired_size / (default_brush_extent * 2) = desired_size / 200
  Volume->SetActorScale3D(Size / 200.0f);
```

There is no default extent of 100 units. A brush actor's extent comes from polys the builder
writes into an initialized `UModel`, and this actor never got one — so the divisor is scaling
nothing. Measured on a `size {15750, 18900, 3500}` request: bounds extent `(0, 0, 0)`, actor scale
`(78.75, 94.5, 17.5)`. The scale write lands; it has no geometry to multiply.

`FoliageHandler.cpp` mentions `Brush` exactly once in the whole file — in the comment at `:1497`
quoted above. There is no brush-init code anywhere in it.

### 2. `actor.spawn {className:"PCGVolume"}` — same, one level more generic

`REGISTER_RPC_HANDLER("actor.spawn", ...)` at `Handlers/Actor/SpawnHandler.cpp:57`; the spawn
itself is a plain `World->SpawnActor(ClassToSpawn, &Location, &Rotation, SpawnParams)` at `:52`.
The negative is the finding: **`grep -n "Brush" Handlers/Actor/SpawnHandler.cpp` returns nothing
across all 384 lines** — zero occurrences of `ABrush`, `BrushComponent`, `UModel` or `UPolys`. The
verb has no notion that some of the classes it will happily spawn are brush actors that need a
model built before they mean anything. `APCGVolume` is one of them.

Measured: `actor.spawn {className:"PCGVolume", ...}` returned a success payload;
`actor.get_bounding_box` on the returned actor returned `extent:[0,0,0]`. A `UPCGComponent`
driven off that volume's bounds samples an empty region.

## Why the existing fix does not reach either

`BuildBoxBrushGeometry` (`VolumeHandler.cpp:90`) does exactly the right thing and is not in
question — `PinWright::MarkLevelActorModified` (`:102`), `PreEditChange` (`:104`),
`Volume->Brush = NewObject<UModel>` (`:110`), `Brush->Initialize(nullptr, true)` (`:111`),
`Brush->Polys = NewObject<UPolys>` (`:112`), `GetBrushComponent()->Brush = Volume->Brush`
(`:113`), `UCubeBuilder` (`:115`), `CubeBuilder->Build(Volume->GetWorld(), Volume)` (`:121`),
`FBSPOps::csgPrepMovingBrush(Volume)` (`:124`), `PostEditChange` (`:126`).

Its three production call sites are all in the same file, all reached through
`CreateBoxBrushForVolume` (`:131`):

- `:298` — the `SpawnVolumeActor` brush overload (`volume.create_*`)
- `:1268` — `volume.set_volume_extent`
- `:1467` — `volume.set_volume_bounds`

(`CreateSphereBrushForVolume` `:136` and `CreateCapsuleBrushForVolume` `:141` are thin wrappers
onto the same helper.) A repo-wide grep for `BuildBoxBrushGeometry` / `Create*BrushForVolume`
across `Plugins/PinWright/Source/` finds no other caller — only those, and two comments in
`Private/Tests/World/TestVolumeHandlers.cpp`. Neither `FoliageHandler.cpp` nor `SpawnHandler.cpp`
appears.

Engine ground truth for why the omission is silent rather than an error:
`UEditorBrushBuilder::EndBrush` (`C:/UE_5.8/Engine/Source/Editor/UnrealEd/Private/EditorBrushBuilder.cpp:52`)
early-returns **success** on a null model —
`UModel* Brush = BuilderBrush->Brush; if (Brush == nullptr) { return true; }` (`:82-86`). Nothing
downstream of these two spawns has any way to notice.

## Verbatim repro

1. `foliage.create_procedural` with a bounds `size` of `{15750, 18900, 3500}` over terrain →
   success, `resimulated:true`, `instances_spawned: 0`.
2. `actor.get_bounding_box {actorName:"<the volume>"}` → `extent:[0,0,0]`.
   `actor.describe` → actor scale `(78.75, 94.5, 17.5)`, i.e. `size / 200` applied to nothing.
3. Repeat 1–2 three more times with different bounds and seeds: **four calls, four
   `instances_spawned: 0`, four `extent:[0,0,0]`.**
4. `actor.spawn {className:"PCGVolume", ...}` → success payload.
5. `actor.get_bounding_box` on it → `extent:[0,0,0]`.

## Workaround

`volume.set_volume_extent` repairs both after the fact: it takes the `Cast<ABrush>` branch and
calls `CreateBoxBrushForVolume` (`VolumeHandler.cpp:1268`), which builds the model the spawn
skipped. So the sequence is spawn → `volume.set_volume_extent` → only then use the volume.

That workaround is harder to reach than it looks, and the obstacle is a parameter name.
`volume.set_volume_extent` declares its actor slot as `RPC_PARAM_REQ("volumeName", ...)`
(`VolumeHandler.cpp:1239-1243`) with no alias. A caller arriving here has just used
`actor.get_bounding_box` / `actor.describe` to *observe* the zero extent, and those declare
`actorName` — canonical head of `ActorNameKeys()` in
`Handlers/Actor/ActorNameParamUtils.h:40-48`, aliased to `objectPath`, `actorPath` and
`actor_name`, but **not** to `volumeName`. So the natural next call fails with `UNKNOWN_PARAMS`
at the exact moment the caller is trying to repair a volume they just proved is broken.
`E-volume-create-name-vs-volumename` (OPEN, Low) covers this slot, but it enumerates the colliding
guess as the generic `name` (taught by `actor.spawn`, `geometry.create_*`,
`material.create_material`) and mentions `actorName` only in passing, as *another* namespace's
create/operate drift. The guess this repro produces is `actorName`, taught by the very readback
that reveals the defect — that path is not in that ticket.

## Fix

Route both spawns through the brush init that already exists rather than re-deriving it:

- `foliage.create_procedural` — after the spawn at `FoliageHandler.cpp:1489-1491`, build the
  brush at the requested `Size` instead of writing `SetActorScale3D(Size / 200.0f)` (`:1499`).
  Delete the `:1497-1498` comment with it; it asserts a default extent that does not exist and
  will otherwise mislead the next reader the same way.
- `actor.spawn` — detect `ClassToSpawn->IsChildOf(ABrush::StaticClass())` at
  `SpawnHandler.cpp:52` and build a default brush, or refuse the class with a message naming
  `volume.create_*` as the right verb. Silently spawning an inert brush actor is the one option
  that should not survive.

`BuildBoxBrushGeometry` is `static` inside an anonymous/`VolumeHelpers` namespace in
`VolumeHandler.cpp`, so routing a second file through it means lifting it into a shared header
first — the same lift `B-foliage-paint-does-no-ground-projection` `#3` did for
`GroundPlacement::MakeProvenanceJson`. Lift it, do not copy it: a second copy of brush-model init
is exactly the kind of duplicate that drifts.

## Same shape as

`B-foliage-paint-does-no-ground-projection` (IN-REVIEW, High) carries the fullest statement of the
class: the call succeeds, every number it reports is correct, and the output is wrong because the
deciding number was never reported. Here the deciding number is the volume's own bounds, and no
response on either path carries it.

Nearest members:

- `B-blocking-volume-no-brush-geometry` (IN-REVIEW, Critical) — same root mechanism, fix already
  written, scope covers neither of these two verbs. The most important link on this ticket.
- `B-foliage-create-procedural-empty-callback-noop` (IN-REVIEW, High) — same verb. Its `#2` fix is
  present and working at HEAD; this ticket supplies the reason its `instances_spawned` is still
  zero. Encounter appended there this session.
- `B-create-procedural-ignores-scale-and-normal-fields` (OPEN, High) — same verb, and its `#3`
  entry already cross-links to this ticket by this exact id, so **this id must not change**.

## Related

- `E-volume-get-info-zero-extent-ambiguous` (OPEN, Low) — asks for a `brushValid` / `hasGeometry`
  flag so an inventory scan can tell a broken volume from a listing that cannot surface bounds.
  This finding is a second and larger population of `{0,0,0}` volumes that flag would catch, and
  one that `volume.get_volumes_info` would not obviously be scanned for: a procedural-foliage or
  PCG volume is not what a caller auditing "volumes" has in mind.
- `E-volume-set-extent-units-class-dependent-docs` (OPEN, Low) — directly load-bearing for the
  workaround above. `set_volume_extent`'s `extent` means absolute world half-extent on the brush
  branch and scale×100 on the non-brush branch; the workaround here only works because both of
  these actors are on the brush branch. A caller who generalises it to a trigger volume gets the
  other meaning and no repair.
- `F-pcg-generate-readback` (IN-REVIEW, Medium) — names `APCGVolume` as a target and checks
  nothing about whether it has geometry. A `pcg.generate` that reads back `0` points off a
  zero-extent volume reports an honest number about a broken input, which is the same trap
  `instances_spawned: 0` set here.

## What was NOT done

- **No fix was attempted.** No source was modified for this ticket.
- **It was not verified that routing these spawns through `BuildBoxBrushGeometry` makes the
  procedural simulation place instances.** What is established is that the volume currently has no
  geometry for the simulation to place *into*. Whether a correctly built brush is sufficient to
  make `instances_spawned` go positive is untested, and there is at least one other known reason
  it might not — `B-create-procedural-ignores-scale-and-normal-fields` `#3` documents a second
  independent defect in the same verb's placement path. A fixer should expect to have to check.
- The `actor.spawn` half is one measured class (`PCGVolume`). Every other `ABrush` subclass
  `actor.spawn` accepts is affected by the same source read, but was not called.

severity rationale: impact=High — the README's High band verbatim, "silent false-success ... the caller trusts a result that is a lie and builds on it": `actor.spawn` reports a spawned volume, `foliage.create_procedural` reports `resimulated:true` over a region it has already been told the size of, and `instances_spawned: 0` reads as an empty region rather than as a phantom volume, which is what sent the previous investigation to the wrong cause (see `B-foliage-create-procedural-empty-callback-noop` `#2`). NOT Critical, and this is a deliberate divergence from `B-blocking-volume-no-brush-geometry`'s Critical despite the identical root mechanism: the Critical band names two things, an editor crash and a write that corrupts or loses asset data, and neither is present — the volumes are written to disk exactly as constructed, with a valid transform and an empty brush; nothing pre-existing is damaged, nothing is unreadable, and deleting the actor loses nothing. Inert is not corrupt. (For the record on the divergence rather than to re-rate another ticket: that ticket's own body justifies its Critical as "the capability appears to work and is silently broken," which is the High band's definition, not the Critical band's — so the gap is between that rating and the rubric, not between the two defects.) The one real asymmetry runs the other way and still does not reach Critical: a collisionless blocking volume fails visibly the first time something walks through the wall, whereas an empty scatter region looks like a design choice, so this is harder to detect and no worse for the data. × reach: BOTH modifiers declined. Not a bump up — `actor.spawn` is an every-session verb but the defect is confined to the `ABrush`-derived slice of the classes it accepts, which is a narrow path through an every-session verb, not an every-session path; `foliage.create_procedural` is one of six verbs in its namespace. Not a bump down either — procedural foliage and PCG volumes are the two standard ways to populate a region with content, not a "rare edge path" in the rubric's sense, and a single session reached for both. High stands unmodified.

## History
- `#1-two-spawn-paths-skip-brush-init` `OPEN` reporter — Measured live against a running editor, five RPC calls total; no fix attempted. **`foliage.create_procedural` (four calls):** each returned `success:true`, `resimulated:true`, `instances_spawned: 0` over terrain; `actor.get_bounding_box` on the spawned `AProceduralFoliageVolume` returned `extent:[0,0,0]` every time, and `actor.describe` showed the requested size written as actor scale `(78.75, 94.5, 17.5)` for a `size {15750, 18900, 3500}` request. **`actor.spawn {className:"PCGVolume"}` (one call + readback):** success payload, `actor.get_bounding_box` → `extent:[0,0,0]`. Mechanism, all line numbers re-derived this session: `FoliageHandler.cpp:1489-1491` raw-`SpawnActor`s the volume with no brush init, then `:1499` writes `SetActorScale3D(Size / 200.0f)` under a `:1497-1498` comment asserting a "default extent of 100 units" that a never-built brush does not have; `SpawnHandler.cpp:52` (registration `:57`) spawns through a plain `World->SpawnActor` and the file contains **zero** occurrences of `Brush` in 384 lines. Engine reason it is silent: `UEditorBrushBuilder::EndBrush` (`C:/UE_5.8/Engine/Source/Editor/UnrealEd/Private/EditorBrushBuilder.cpp:52`) returns `true` on a null model (`:82-86`). The correct repair already exists in-tree — `VolumeHelpers::BuildBoxBrushGeometry` `VolumeHandler.cpp:90` (`PreEditChange` `:104`, `NewObject<UModel>` `:110`, `Initialize` `:111`, `Polys` `:112`, `BrushComponent->Brush` `:113`, `UCubeBuilder` `:115`, `Build` `:121`, `csgPrepMovingBrush` `:124`, `PostEditChange` `:126`) — landed by `B-blocking-volume-no-brush-geometry` `#3-fix-brush-model-init`, but a repo-wide grep finds its only production call sites are `VolumeHandler.cpp:298` (`SpawnVolumeActor`), `:1268` (`set_volume_extent`) and `:1467` (`set_volume_bounds`), so neither verb here reaches it. Workaround `volume.set_volume_extent` (which routes through `:1268`) is obstructed by its `volumeName` slot (`:1239-1243`, no alias) rejecting the `actorName` the readback verbs teach (`ActorNameParamUtils.h:40-48`) — cross-linked to `E-volume-create-name-vs-volumename`, which enumerates `name` but not that path. NOT DONE: no source modified, and it was **not** verified that building the brush makes the simulation place instances — only that there is currently no geometry for it to place into; `B-create-procedural-ignores-scale-and-normal-fields` `#3` documents a second independent placement defect in the same verb, so a fixer should expect to check. `PCGVolume` is the one `actor.spawn` class actually called; other `ABrush` subclasses are implicated by source read only. Dedup: this is not `B-blocking-volume-no-brush-geometry` re-filed — that ticket is `volume.create_blocking_volume` / `set_volume_extent` / `set_volume_bounds`, its fix is written and IN-REVIEW, and its scope is `VolumeHandler.cpp`; the two verbs here are in `foliage.*` and `actor.*`, are untouched by that fix, and would still fail after it is verified DONE. No umbrella filed — the class statement lives in `B-foliage-paint-does-no-ground-projection`'s `## Same shape as` and is referenced, not restated.
- `#2-the-recommended-workaround-overshoots-by-100x` `OPEN` reporter — **Correction to `#1`'s workaround, from a second zone agent measuring the same session.** `#1` recommends `spawn -> volume.set_volume_extent -> use the volume` as the repair for a zero-extent volume. That sequence produces a volume **100x too large on every axis**. Measured: `volume.set_volume_extent` asked for `(8000, 12000, 3000)` left the actor at scale `(80, 120, 30)` with bounds extent `(640000, 1440000, 90000)`. The brush the verb builds is the right size; the actor scale is a redundant second application of the extent against an assumed 100-uu cube, and the two multiply. So a fixer following `#1` literally trades a volume that samples nothing for one that samples the whole level, which on a PCG or procedural-foliage host is the worse of the two. **The workaround as written is: spawn -> `volume.set_volume_extent` -> reset actor scale to `(1,1,1)` -> use the volume.** That third step is inferred from the measurement, not tested — nobody has run the corrected sequence end to end, and this entry does not claim they have. Filed separately as `B-set-volume-extent-applies-scale-on-top-of-brush`, which owns the defect and the fix; this entry exists so the workaround here is not read on its own. The two are cross-linked in both directions. Note the interaction a fixer must hold in mind: if `volume.set_volume_extent` is fixed to stop writing the scale, the correction above becomes wrong in turn, so whichever lands second must update the other.
- `#3-correcting-my-own-2-the-scale-is-this-tickets-own-defect` `OPEN` reporter — **Two corrections to `#2`, both owed to the ticket it forward-referenced, which landed under a different id and with a better diagnosis than `#2` gave it.** **(1) The id is `B-set-volume-extent-ignores-existing-actor-scale`**, not the `B-set-volume-extent-applies-scale-on-top-of-brush` that `#2` names; the name changed because the mechanism did. **(2) `#2`'s mechanism is wrong.** `#2` says `volume.set_volume_extent` applies the extent a second time as a scale against an assumed 100-uu cube. It does not, and that ticket disproves it two independent ways. The `SetActorScale3D(NewExtent / 100)` line is real (`Handlers/Volume/VolumeHandler.cpp:1272`) but sits in the **`else`** arm of `if (Cast<ABrush>(VolumeActor))` (`:1265-1273`), and `AProceduralFoliageVolume` is an `ABrush`, so `:1268` runs and `:1272` cannot. The arithmetic confirms it independently: at scale 80 a bounds half-extent of 640000 requires a base half-extent of 8000, which is exactly what `CreateBoxBrushForVolume` builds; a non-brush base would have given about 3200. **What that means for THIS ticket is worse, not better, which is why it is recorded here and not only there.** The scale was already on the actor before `set_volume_extent` was ever called, and it was put there by **this ticket's own defect** - `FoliageHandler.cpp:1499` writing the requested size as `SetActorScale3D(Size / 200)` onto geometry that did not exist. So the `(78.75, 94.5, 17.5)` that `#1` records as a harmless symptom of a phantom volume is not harmless: it is live state that survives the repair and multiplies it. Build the brush correctly on a volume still carrying that scale and you get a volume about 100x too large - which, for a PCG or procedural-foliage host, is a worse failure than the zero-extent one it replaced. **Consequences for the fix, which a fixer must not miss.** Routing these two spawn paths through `BuildBoxBrushGeometry` is necessary and **not sufficient**: the same change has to stop writing the scale at `FoliageHandler.cpp:1499`, or every volume it creates comes out at extent x scale. And any migration for volumes already spawned by the broken path must normalize scale to `(1,1,1)`, because all of them are carrying one. The workaround in `#1` and `#2` therefore reads, corrected: spawn -> `volume.set_volume_extent` -> reset actor scale to `(1,1,1)`. **Still untested end to end** - nobody has run the corrected sequence, and `#2` should not have stated the correction as confidently as it did.
