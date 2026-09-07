---
id: F-spatial-swept-shape-query
title: "The whole plugin issues exactly two kinds of collision query — a line trace and one STATIC box overlap — so there is no swept capsule, box or sphere anywhere on the RPC surface, and \"can a character capsule get from here to there\" has to be hand-rolled in `python.execute`; a line trace is not a substitute, because a capsule is stopped by geometry no ray through its axis ever touches"
status: OPEN
severity: Medium
category: feature
tags: [spatial, raycast, sweep, capsule, shape-trace, overlap, collision, pawn, traversal, walkability, missing-verb, level-building, vegetation]
encounters: 1
costly: 1
lastSeen: 2026-08-29T22:05:00+03:00
---

# Two collision queries in the entire tree, and neither one is swept

Grepped across all of `Plugins/PinWright/Source/` at HEAD `962275fa`, the plugin makes exactly three
collision queries:

    Private/Handlers/Spatial/SpatialTraceUtils.cpp:306    World->LineTraceSingleByChannel(Hit, Start, End, Channel, QueryParams)
    Private/Handlers/Spatial/SpatialTraceUtils.cpp:407-408 World->OverlapMultiByChannel(Overlaps, Probe.Center, Probe.Rotation, Probe.Channel,
                                                                                        FCollisionShape::MakeBox(Half), QueryParams)
    Private/Handlers/Spatial/GroundPlacementUtils.cpp:93   Component->LineTraceComponent(Hit, Start, End, Params)

`SweepSingleByChannel`, `SweepMultiByChannel`, `SweepTestByChannel` and `ComponentSweepMulti` have
**zero occurrences**. (Four `Sweep` tokens do exist in the tree and all four are audio automation
test names — `Tests/Media/TestAudioAnalysisHandler.cpp:1008`, `:1012`,
`Tests/Media/TestAudioSynthGenerate.cpp:837`, `:841`. Worth pre-empting because a reviewer's own
grep will find them.)

Two further hits look like sweeps and are inert:

    Private/Compiler/CodeFunctionResolver.cpp:65   SphereTraceByChannel  -> SphereTraceSingle
    Private/Compiler/CodeFunctionResolver.cpp:66   BoxTraceByChannel     -> BoxTraceSingle
    Private/Compiler/CodeFunctionResolver.cpp:67   CapsuleTraceByChannel -> CapsuleTraceSingle

Those are BPIR display-name to C++-name aliases. They *author a K2 node in a Blueprint graph*; they
never execute a query in the editor.

**The one shape query is a static overlap, not a sweep**, and it is reachable through exactly one
verb: `ProbeFootprintOccupancy` (`SpatialTraceUtils.cpp:385`) -> `MeasureFootprintClearance`
(`:490`) -> `spatial.find_clear_placement` (`Private/Handlers/Spatial/MeasureHandler.cpp:638`, call
at `:1112`). It is box-only, it tests discrete poses rather than a path, and its channel is
hardcoded (`MeasureHandler.cpp:1061`, `Probe.Channel = ECC_Visibility;`).

## Why a line trace is not a substitute

A walking character is a capsule. A ray down its axis passes through a doorway a capsule cannot fit
through, misses the rock that stops it, and misses every overhang between knee and head height. The
question is not "is there something at this point" but "does this volume fit along this path", and
the two have different answers on any real level.

Concretely, from a collision fix pass on `/Game/Maps/PW_VegetationTest` (editor build **10:44**;
receipts `X:/src/unreal/EAContentExamples58/dev/grasscollide/`): verifying that 5527 groundcover
instances no longer blocked a player needed a capsule of r 34 / half-height 88 swept 600 cm
horizontally at ground+90, at 8 frozen sites plus 120 segments over 8 ground-following traverses
across four zones. It was written as `dev/grasscollide/gc_sweep.py` and `gc_traverse.py` on
`SystemLibrary.capsule_trace_multi_by_profile`, because nothing typed can express it. The result the
harness produced — before, two components blocking; after, one fallen log level-wide — is not
obtainable from `spatial.raycast` at any channel, at any sample density.

The `initialOverlap` flag those receipts carry is the other reason a hand-rolled version is a trap:
`F-spatial-no-clear-footprint-search` `#2` records a caller dropping to `python.execute` /
`sphere_trace_multi` for camera-corridor clearance and discovering that a one-way segment sweep is
**direction-blind when the sweep starts already overlapping**. Two independent passes have now
hand-rolled a sweep and each hit a different edge of it.

## Ask

    spatial.sweep {
      shape:        "capsule" | "box" | "sphere",
      radius?, halfHeight?, halfExtent?,           // per shape
      start, end,                                  // or start + direction + distance
      rotation?,                                   // box only
      channel? | profile?,                         // see F-trace-channel-vocabulary-incomplete
      traceComplex?, multiHit?, onlyActors?, actorFilter?, onlyClasses?
    }

Three properties earned above:

1. **Publish the same hit shape `spatial.raycast` already publishes** — including
   `simpleCollisionShapes` and `renderGeometryHit`, which exist because a mesh with no simple
   collision still blocks a complex trace (`B-trace-complex-hits-render-geometry`). A sweep verb
   without them re-opens a defect the line trace already closed.
2. **`initialOverlap` per hit, and say what it means.** A sweep that begins inside geometry is the
   normal case for a character standing on the ground, and the flag is the difference between "this
   path is blocked" and "you asked from inside a wall".
3. **`profile` as an alternative to `channel`**, because "as a pawn would" is a profile
   (`UCollisionProfile::GetChannelAndResponseParams`,
   `C:/UE_5.8/.../Classes/Engine/CollisionProfile.h:213`) and that is what both hand-rolled versions
   actually used.

## Scope note: this does not, by itself, answer the question either

A capsule sweep on the Visibility channel answers a question nobody asked. The channel vocabulary is
`F-trace-channel-vocabulary-incomplete`, filed separately because it is one shared helper replacing
four if-chains rather than a new verb, and because a fixer can land either alone. **Neither alone
makes "can a player walk here" askable; both together do.**

## Not a duplicate of

- **`F-spatial-no-clear-footprint-search`** (IN-REVIEW, Medium, encounters 2) — **the nearest
  neighbour, and it is not this.** Its ask is an occupancy *search*: "does a WxH footprint fit near
  here", and `#3` shipped `spatial.find_clear_placement` backed by a static box **overlap**. Three
  differences that keep them apart: an overlap tests a pose and a sweep tests a path; that verb is
  box-only and picks its own poses; and its channel is hardcoded. What it does contribute is the
  prior independent encounter of this gap, in its `#2`, cited above — that entry is evidence for
  this ticket, not coverage of it.
- **`F-trace-channel-vocabulary-incomplete`** (OPEN, filed from this pass) — its partner. See the
  scope note.
- **`F-spatial-raycast-no-batch-multi-origin`** (OPEN, Low) — more rays per call. Same shape (a
  line), different arity. A batch of rays is still not a capsule.
- **`B-raycast-screen-lacks-trace-diagnostics`** (OPEN, Medium, encounters 2) — diagnostic-field
  parity between the two existing line-trace verbs. Directly relevant as a constraint on the ask
  (property 1 above), not as an owner: a third trace verb shipped without those fields would make
  that ticket's problem worse.
- **`B-trace-complex-hits-render-geometry`** (DONE, High) — `traceComplex` resolving against render
  triangles. The property a new verb must inherit, not a duplicate.
- The board's four `navigation.*` tickets (`B-level-build-navigation-no-completion-signal`,
  `B-navigation-rebuild-navigation-no-completion-signal`,
  `B-nav-agent-properties-restored-on-registration`, `B-nav-link-proxy-appends-default-link`) are all
  navmesh **build/config** defects. None is a query, and a built navmesh answers a different question
  anyway (where an agent may path, not what a capsule collides with).

## Severity

**Medium**, on the rubric's soft-blocker band: *"doable, but only via a documented workaround, a
source dive, or many extra calls."* Both passes that needed a sweep got one, via `python.execute`
and `SystemLibrary.*_trace_*`. That is a workaround and it works, so this is not the "hard blocker,
no workaround" band.

**Bump-up to High declined**, for the reason the joint case is joint: the impact that would justify
High — that traversability is unaskable — is shared with
`F-trace-channel-vocabulary-incomplete`, and charging it to both tickets would count one gap twice
in the picker's ordering. Charged once, to neither, both sit at Medium and a fixer taking either will
read the other.

**Bump-down to Low declined.** Low is friction: docs, naming, a spill. Two independent passes have
now written their own sweep harness, and each discovered a different correctness trap in doing so
(`initialOverlap` direction-blindness there, the simple-vs-complex geometry question here) — which
is the definition of a capability the surface should own rather than each caller re-deriving.

## History
- `#1-no-swept-shape-anywhere` `OPEN` reporter — Filed from a collision-and-seating fix pass on
  `/Game/Maps/PW_VegetationTest` (editor build 10:44; plugin source HEAD `962275fa`), which needed
  128 pawn-profile capsule sweeps to verify that a groundcover collision fix had taken, and wrote
  them in `python.execute` because nothing typed can sweep. Every absence above was grepped rather
  than assumed: `SweepSingle`/`SweepMulti`/`SweepTest`/`ComponentSweepMulti` return four hits in
  `Source/` and all four are audio automation test names; `FCollisionShape` appears twice, at
  `SpatialTraceUtils.cpp:408` (`MakeBox`, the static overlap) and
  `Tests/Actor/TestShapeExtentPhysicsBody.cpp:118` (`MakeSphere`, test-only); the
  `CapsuleTrace`/`BoxTrace`/`SphereTrace` hits are BPIR node-name aliases at
  `Compiler/CodeFunctionResolver.cpp:65-67`. Dedup: searched the board for `sweep`, `swept`,
  `capsule`, `sphere_trace`/`box_trace`/`capsule_trace`, `trace_multi`, `FCollisionShape`,
  `walkab*`, "player walk", `traversal`, `navmesh`, and every `F-spatial-*` file. Nothing asks for a
  swept query. `F-spatial-no-clear-footprint-search` is the closest and shipped a static box
  overlap; its `#2` records a prior independent caller hand-rolling `sphere_trace_multi` and hitting
  the initial-overlap trap, which is cited above as evidence and folded into the ask rather than
  being treated as coverage.
