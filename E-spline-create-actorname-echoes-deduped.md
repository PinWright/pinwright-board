---
id: E-spline-create-actorname-echoes-deduped
title: "spline.create_spline_actor / create_spline_mesh_actor give the caller no signal that the engine deduplicated the actor to <name>_0 on a label collision — the response's actorName is the shared (non-unique) label and no field flags the rename, so follow-up spline calls keyed on actorName hit the WRONG actor"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [spline, create_spline_actor, create_spline_mesh_actor, actorname, label-collision, deduplication, result-misreport, wrong-target, add_spline_point, get_splines_info]
---

# `spline.create_spline_actor` reports a non-unique label as `actorName` and gives no signal when a collision deduplicated the new actor (on a name collision)

`spline.create_spline_actor` spawns its actor with
`SpawnParams.Name = *ActorName; NameMode = ESpawnActorNameMode::Requested`
(`SplineHandler.cpp:217-218`) and then `SetActorLabel(*ActorName)`
(`:228`). When an actor with that **internal object name already exists**
in the level, UE's `Requested` mode does **not** fail — it silently
generates a unique object name (`<ActorName>_0`, `_1`, …). The handler then
sets the new actor's **display label** to the requested name anyway (labels
are allowed to collide), so the new actor ends up with:

- internal object name (the addressable, unique key, = the `actorPath` leaf): `<ActorName>_0`
- display label: `<ActorName>` (now shared with the pre-existing actor)

But the response (`:276`) reports
`actorName = NewActor->GetActorLabel()` = the **requested** name
(`<ActorName>`), **not** the unique object name (`<ActorName>_0`). The
single field a caller would feed into the next spline call therefore
**does not address the actor that was just created**.

This breaks the natural create → edit → readback chain in the same
namespace. Every other spline verb resolves an actor via
`It->GetActorLabel() == ActorName || It->GetName() == ActorName` and
**`break`s on the FIRST match in iteration order** (`SplineHandler.cpp:73`).
Passing the echoed `actorName` (the shared label) resolves to whichever
same-labelled actor iterates first — in practice the **pre-existing** one,
not the newly created `<name>_0`. So `add_spline_point` /
`set_spline_point_position` / `set_spline_type` /
`scatter_meshes_along_spline` / `get_splines_info`, all keyed on the value
the create call handed back, silently operate on the **wrong actor**. The
caller gets a success-shaped chain and a `get_splines_info` "confirmation"
that describes the wrong spline, with no signal anything diverged.

This is the **result-misreport / silent-wrong-target** half of the
colliding-label problem, distinct from the input-side docs ticket
`E-actor-name-resolution-label-collision` (IN-REVIEW), which only tells
*generic* `actor.*` callers to prefer the object name; it does not touch the
spline namespace and, crucially, the spline create verb here actively hands
back a name that resolves wrong — the caller has nothing to disambiguate
*with* because `actorName` lies and `actorPath` (which DOES carry the truth,
`...PersistentLevel.<name>_0`) is never surfaced as the thing to key off.
It is also distinct from `B-get-splines-info-ignores-spline-mesh`
(SplineMeshComponent class-coverage in the readback).

## Replay-confirmed repro (live editor, `mcp__editor-automation__call`)

Setup — one actor named `ReplayCheck_Spline` already exists (4 Curve points):

1. `spline.create_spline_actor` `{actorName:"ReplayCheck_Spline", location:{x:9000,y:9000,z:0}, bClosedLoop:false, splineType:"Linear", points:[{location:{x:0,y:0,z:0}},{location:{x:100,y:100,z:0}}]}`
   → `{"actorName":"ReplayCheck_Spline", "actorPath":"/Game/Maps/ExampleProjectWelcome.ExampleProjectWelcome:PersistentLevel.ReplayCheck_Spline_0", "pointCount":2, "splineLength":141.42, "closedLoop":false, ...}`
   — note the **mismatch**: `actorName:"ReplayCheck_Spline"` but `actorPath` leaf is `ReplayCheck_Spline_0`. The new actor is addressable only as `ReplayCheck_Spline_0`.

2. `spline.get_splines_info` `{actorName:"ReplayCheck_Spline"}`  (i.e. the name the create call returned)
   → `{"actorName":"ReplayCheck_Spline", "pointCount":4, ... points all type "Curve" ...}`
   — resolves to the **pre-existing** 4-point Curve actor, NOT the 2-point Linear actor just created.

3. `spline.get_splines_info` `{actorName:"ReplayCheck_Spline_0"}`  (the real object name, from the `actorPath` leaf)
   → `{"actorName":"ReplayCheck_Spline_0", "pointCount":2, ... points type "Linear" ...}`
   — only the unique object name reaches the newly created actor.

So the value the create call put in `actorName` is unusable for addressing
its own result; the caller must instead parse the `actorPath` leaf — which
the response documents nothing about and which a reasonable agent does not
know to prefer (it took the obvious `actorName`).

### Field demonstration of the downstream corruption

In the originating realism task (garden-path spline, seed `spline`) the
attempt agent created `GardenPath_Spline`, appended two points, nudged a
point, set types, scattered meshes, and read back — all `ok` — and
self-reported "5 points, closedLoop=false, all Curve" as confirmed by
`get_splines_info`. Replayed against the non-reset level (a
`GardenPath_Spline` already present), the very same sequence produced a
**7-point** spline whose readback showed the two appended points DUPLICATED
(indices 3&5 = (1800,400,0), indices 4&6 = (2400,0,0)) — because the
follow-up calls keyed on the colliding label landed on a different actor
than the agent believed it had created, while `get_splines_info` (same
key) "confirmed" a count that was never what the agent built. The
create-verb name echo is the mechanism that lets that whole chain go wrong
silently.

## What it should do

Give the caller an unambiguous, machine-checkable signal that a collision
deduplicated the new actor, and surface the round-trippable key — **without
redefining what `actorName` means**.

Why not "just report `GetName()` as `actorName`": across the product
`actorName` uniformly means the **display label**. The shared
`AddActorVerification` helper (`Utils/AssetUtils.cpp:939`, ~127 call sites)
sets `actorName = Actor->GetActorLabel()` and surfaces the unique key
separately as `actorPath = Actor->GetPathName()` (the
`E-actor-verification-actorpath-is-map-path` fix deliberately KEPT
`actorName = label` and added the object-path value). Both spline create verbs
call `AddActorVerification` immediately AFTER their own `actorName` write
(`SplineHandler.cpp:282`/`:1186`), so the helper **clobbers** any manual
`actorName` the verb sets first — editing the `:276`/`:1182` write to
`GetName()` would change nothing on the wire. Redefining `actorName` to the
object name in spline alone would also make spline the lone namespace where
`actorName` ≠ label, breaking symmetry with every sibling verb and the ~127
verification sites. The label-vs-internal-name asymmetry tickets all resolve
the same way — by ADDING a field, never by redefining `actorName`
(`E-inspect-find-by-tag-internal-name-not-label`,
`E-networking-actorname-internal-name-only`).

Concrete fix, both create verbs (`spline.create_spline_actor`
`SplineHandler.cpp:275-282` and `spline.create_spline_mesh_actor` `:1181-1186`):

1. Capture the requested name and the resulting unique object name, then
   compare. After spawning, `NewActor->GetName()` differs from the requested
   name exactly when UE's `Requested` NameMode deduplicated on a collision.
2. Drop the now-dead manual `actorName = GetActorLabel()` write (the
   verification helper sets it). Let `AddActorVerification` own `actorName`
   (= label) and `actorPath` (= the unique object path, whose leaf
   round-trips through the sibling resolver `FindActorByName` at `:73`).
3. AFTER `AddActorVerification`, add three response fields the helper does
   **not** touch: `requestedName` (what the caller asked for),
   `actorObjectName` (`NewActor->GetName()` — the unique key the sibling
   verbs resolve to), and `nameWasDeduplicated` (bool: `actorObjectName !=
   requestedName`). When `true`, the caller knows the label collided and must
   key follow-ups on `actorObjectName` (or the `actorPath` leaf), not
   `actorName`.

This keeps `actorName = label` (convention-aligned), changes the actual wire
output (new fields are written after the helper, so they aren't clobbered),
and gives the caller a one-bool collision signal plus the exact key to chain.

**Workaround:** ignore the returned `actorName` after a create; key every
follow-up spline call on the **leaf of `actorPath`** (the unique object
name, e.g. `GardenPath_Spline_0`) — or, after the fix, on the new
`actorObjectName` field / when `nameWasDeduplicated` is `true` — which is the
only value that resolves to the actor just created when the requested label
was already taken.

## History
- `#1-initial-repro` `OPEN` reporter — Realism-mode struggle audit of a garden-path spline task (seed `spline`; create_spline_actor → add_spline_point ×2 → set_spline_point_position → set_spline_type → scatter_meshes_along_spline → get_splines_info; attempt outcome `done`, self-reported clean "5 points confirmed"). Replay-confirmed live (UE 5.7): when an actor of the requested name already exists, `spline.create_spline_actor` spawns with `NameMode=Requested` (auto-dedups the object name to `<name>_0`) but echoes `actorName=GetActorLabel()`=the requested name (`SplineHandler.cpp:217-218,228,276`), while the real addressable name is the `actorPath` leaf `<name>_0`. The namespace resolver (`:73`, label-OR-name, first-match-`break`) then routes follow-up calls keyed on the echoed name to the WRONG (pre-existing) actor; replay produced a 7-point spline with the two appended points duplicated, and `get_splines_info` "confirmed" the wrong actor with no error — explaining the attempt's false "5 points" self-report. Isolated control: a FIRST creation with a fresh name returns a correct, round-trippable `actorName` and a clean 4-point append (3→4) — the defect manifests only on name collision. Dedup (ripgrep over OPEN+closed; qmd unavailable): not `E-actor-name-resolution-label-collision` (input-side docs for generic `actor.*`, doesn't touch the spline create echo), not `B-get-splines-info-ignores-spline-mesh` (SplineMeshComponent readback coverage), not `E-geometry-create-name-vs-actorname` (name-vs-actorName slot drift). Culprit method: `spline.create_spline_actor` (seed was the `spline` namespace nav). Proposed fix: report `NewActor->GetName()` (the unique object name) as `actorName` from the spline create verbs (`:276` and the spline-mesh create `:1182`), optionally add an `actorLabel` field and a deduplication flag.
- `#2-reword+fix` `IN-REVIEW` developer — Reworded then fixed. The reword: the original `Fix:` (change `:276`/`:1182` `actorName` to `NewActor->GetName()`) is a **no-op** — both create verbs call `AddActorVerification` immediately after (`SplineHandler.cpp:282`/`:1186`), and that shared helper (`Utils/AssetUtils.cpp:939`, ~127 sites) re-sets `actorName = GetActorLabel()`, clobbering the verb's write. The established convention (landed in `E-actor-verification-actorpath-is-map-path`, IN-REVIEW) keeps `actorName = label` and surfaces the unique key as `actorPath = GetPathName()`; the asymmetry-fix family (`E-inspect-find-by-tag-internal-name-not-label`, `E-networking-actorname-internal-name-only`) always ADDS a field rather than redefining `actorName`. Retitled/reworded to: keep `actorName = label`, drop the dead manual write, and add a collision SIGNAL after the helper. Fix applied to `Source/EditorAutomationRpcGateway/Private/Handlers/Geometry/SplineHandler.cpp`, both `spline.create_spline_actor` and `spline.create_spline_mesh_actor`: removed the redundant `actorName = GetActorLabel()` write (the helper owns it), and AFTER `AddActorVerification` added three fields it does not touch — `requestedName` (the input name), `actorObjectName` (`NewActor->GetName()`, the unique key the sibling resolver `FindActorByName` at `:73` matches), and `nameWasDeduplicated` (`actorObjectName != requestedName`). Verb-summary/`splineType` param help unchanged. Regression test: `Tests/World/TestGeometryHandlers.cpp` → `EditorAutomationRpcGateway.spline.create_spline_actor.DeduplicatedNameSignalled`. It pre-spawns a spline actor with a fixed label via the production `spline.create_spline_actor`, then creates a SECOND actor with the SAME requested name through the same production handler and captures the response: asserts `nameWasDeduplicated == true`, `actorObjectName != requestedName` (the engine renamed it to `<name>_0`), `requestedName` equals the requested label, and that `actorObjectName` resolves to a DIFFERENT live actor than the pre-existing one (round-trips, unlike the shared `actorName` label). A control first-creation with a fresh unique name asserts `nameWasDeduplicated == false` and `actorObjectName == requestedName`. Reverting the fix (removing the fields, or restoring the no-op `actorName=GetName()` write) fails the test. Files: `Source/EditorAutomationRpcGateway/Private/Handlers/Geometry/SplineHandler.cpp`, `Source/EditorAutomationRpcGateway/Private/Tests/World/TestGeometryHandlers.cpp`. Not compiled/run here (a later phase verifies green).
