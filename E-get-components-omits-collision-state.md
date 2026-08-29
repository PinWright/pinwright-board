---
id: E-get-components-omits-collision-state
title: "`actor.get_components` emits name, class, path and a scene transform and nothing else, so \"which components block a player\" cannot be asked of it — answering it on one level meant a 312-component `python.execute` census, and the two fields that have to be read TOGETHER (CollisionEnabled and the response container) disagree on 71 of its 147 rows"
status: OPEN
severity: Medium
category: ergonomic
tags: [actor, get_components, readback, collision, collision-profile, collision-channel, body-instance, object-type, ism, hism, pawn, census, missing-field, level-building]
encounters: 1
lastSeen: 2026-08-29T21:55:00+03:00
---

# The component list knows what a component is and nothing about what it does

`actor.get_components` (`Private/Handlers/Actor/ComponentHandler.cpp:454`) builds one entry per
component (`:502-537`) and the entry is complete at four fields:

    ComponentHandler.cpp:508      name
    ComponentHandler.cpp:509-511  class          (class path)
    ComponentHandler.cpp:512      path
    ComponentHandler.cpp:519-522  relativeLocation   } USceneComponent only
    ComponentHandler.cpp:525-528  relativeRotation   }  (the handler's only
    ComponentHandler.cpp:531-534  relativeScale      }   type-specific branch)

plus `components` (`:540`) and `count` (`:541`). There is **no `UPrimitiveComponent` cast anywhere in
the handler**, and therefore no `collisionEnabled`, no `collisionProfileName`, no `objectType` and no
per-channel response container.

## What that costs, measured

Build: the editor's **10:44** PinWright build; source read at HEAD `962275fa`. Level
`/Game/Maps/PW_VegetationTest`. Scripts and receipts:
`X:/src/unreal/EAContentExamples58/dev/grasscollide/`.

The question was "why does grass have player collision, and what exactly is blocking?". Answering it
required `dev/grasscollide/gc_census.py`: a `python.execute` walk of **312** components producing
**147** collision-bearing rows (`out/census_before.json`), each carrying the seven fields the RPC
surface does not publish — `collisionEnabled`, `profile`, `objectType`, `respVisibility`,
`respPawn`, `bodies`, `instances`. Three facts out of that census, none of them reachable from
`get_components`:

**1. One boolean cannot answer the question.** After the fix pass, **83 of 147** rows have
`respVisibility != respPawn` (`out/census_after.json`). Twelve of them are `QUERY_AND_PHYSICS` with
Visibility `Block` and Pawn `Ignore` — components deliberately kept as valid ground-probe surfaces
while no longer blocking a walking player. "Does this component collide" and "does this component
block a pawn" are different questions on this level, and only the second is the one anyone asked.

**2. The two fields have to be read together or the census is wrong.** **71 of 147** rows read
`collisionEnabled: NO_COLLISION` while `respPawn` reads `ECR_BLOCK`. The response container is
simply never consulted while collision is disabled, so a readback that published responses without
the enable state — or the enable state without the responses — would produce a confidently wrong
answer in either direction. Any `collision` block added here must carry both, in one row.

**3. It is a per-component question and the actor is the wrong unit.** `ZoneF_Veg` is a single
actor carrying 15 HISM components and 5888 instances; seven of its components mattered and eight did
not.

## What DOES exist, so the ask is stated honestly

"Nothing anywhere reports component collision" would be refutable. Three things report something:

- **`actor.get_component_property`** (`ComponentHandler.cpp:605`) reads one dotted field per call,
  and its own `propertyName` doc (`:609`) uses **`'BodyInstance.CollisionEnabled'`** as the worked
  example. It is the intended route today. At seven fields x 312 components that is 2184 calls, which
  is why the census was written in Python instead. `property.get`
  (`Private/Handlers/Utility/UtilityPropertyHandler.cpp:1556`) is the same shape, generic.
- **`editor.set_view_mode`** (`Private/Handlers/Editor/ViewportHandler.cpp:367`) emits a named
  `collisionEnabled` (`:326`) and `respondsToChannel` (`:327`). But it is **per actor**, aggregated
  across components (`bAny*`), row-capped, restricted to actors *not drawn* in the chosen collision
  view mode, and it is a side effect of a **mutating** verb that switches the viewport's persistent
  view mode. It reports no profile name and no per-channel table.
- **`actor.describe`** will leak `BodyInstance` as a sparse archetype diff
  (`Private/Utils/ActorDescribeBuilder.cpp:105-107`) — but only when it happens to be overridden, as
  an exported struct blob, and never when collision matches the archetype. It cannot answer "what is
  this component's profile".

None of the three is a component-level collision readback, and the first is the only one that is not
a side effect.

## Ask

An opt-in `collision` object on each `UPrimitiveComponent` entry:

    collision: {
      enabled:      "QueryAndPhysics",              // ECollisionEnabled
      profileName:  "Custom",
      objectType:   "WorldStatic",                  // ECollisionChannel
      responses:    { "Pawn": "Ignore", "Visibility": "Block", ... },
      instanceBodies: 41                            // ISM/HISM only, see below
    }

Opt-in (a `includeCollision` / `fields` flag) rather than always-on, because this verb already spills
on populated actors and the board carries three tickets about response size on the enumeration verbs
(`E-actor-list-no-limit-spills`, `E-actor-describe-no-header-only-read`,
`E-blueprint-list-no-projection-spills`). `enabled` and `responses` must be published together for
the reason in fact 2 above.

**One thing this block would NOT have caught, said plainly.** On an instanced component the
per-instance bodies carry their own filter data, copied once from the component's `BodyInstance` and
never refreshed (`B-collision-write-skips-instance-bodies`). A `collision` block read off
`BodyInstance` would have reported `Pawn: Ignore` on all twelve components while seven of them were
still blocking a pawn sweep — i.e. it would have agreed with the lie. That is an argument for the
block carrying something instance-derived (`instanceBodies` count at minimum, ideally a flag when
the live filter data differs from the template), not an argument against the block: the census this
verb could not serve is what found the divergence in the first place.

The **write** counterpart is `F-component-collision-channel-write`.

## Not a duplicate of

- **`F-capture-drawn-primitive-manifest`** (OPEN, Medium) — **the physics-side twin, and the right
  cross-link.** That ticket asks `render.capture_open_level` to name the instanced primitives it
  *drew*; this one asks `actor.get_components` to name what each component *blocks*. Same underlying
  hole seen from two sides — the surface can trace against collision and render pixels, and can
  enumerate neither. Note the sharpest overlap and why they stay separate: that ticket's
  load-bearing implementation constraint is *"do not build it on a physics query"*, because this
  project's vegetation is largely collisionless and a physics-backed manifest would silently omit it.
  It therefore deliberately routes **around** collision, which is exactly the thing this ticket asks
  to be reported. Different verb, different unit (a drawn instance in a frustum vs a component's
  configuration), opposite relationship to the physics world.
- **`E-get-component-property-no-subfield-select`** (DONE, Medium) — shipped the dotted read that is
  the current workaround, and its own note names "read collision via `actor.set_collision`
  round-trip" as the alternative. It made one field cheap; it did not make the enumeration possible.
- **`E-component-read-filter`** (DONE, Medium) — shipped `nameMatch` / `componentClass` on this
  verb. Input filters: *which* components are listed. This is *what each row carries*.
- **`E-actor-get-components-bp-cdo-omits-scs`** (OPEN, Low) — SCS templates missing from the BP-CDO
  path. Also about which components are listed.
- **`E-actor-describe-no-header-only-read`** (IN-REVIEW, Low) — wants *fewer* fields, for spill.
  Directly relevant as a constraint on the ask, which is why the block is opt-in.
- **`B-get-components-renders-empty-to-caller`** (IN-REVIEW, Medium) — a transport dead band that
  happens to be named after this verb.

## Precedent for the ask shape

- **`E-volume-get-info-omits-physics-properties`** (OPEN, Low, encounters 3) — `volume.get_volumes_info`
  echoes name/class/location/extent and never reads back the physics properties its own sibling verb
  writes. Same argument, different verb, and the board has accepted it three times.
- **`B-static-mesh-missing-collision-body-counts`** (DONE, Low) — collision body counts added to the
  static-mesh dump. The asset-side precedent for publishing collision facts in a readback.

## Severity

**Medium**, on the rubric's clause *"a readback omits a field and forces a fallback"*. The fallback
here is real and was paid: 312 components walked in `python.execute` because the typed route is one
field per call.

**Bump-up to High declined.** `actor.get_components` does run in almost every session, which is the
rubric's trigger — but the honest scope of the affected method is `actor.get_components` *when the
caller needs collision state*, and that is a minority of its calls. This is the same reading
`B-property-set-object-hop-notification-noop` applied when it declined the same bump ("the modifier
is about the affected method, and `property.set` as a method is not affected"). Rating this above
measured silent-false-success defects would mis-order the queue.

**Bump-down to Low declined.** Low is "pure friction: docs, discoverability, naming, a response spill
that only forces a `Read`". The information is not merely awkward to reach — through typed verbs it
is reachable only one field at a time, and the question it blocks ("can a player walk here") has no
answer anywhere on the surface (`F-trace-channel-vocabulary-incomplete`,
`F-spatial-swept-shape-query`).

## History
- `#1-no-collision-fields-on-the-component-list` `OPEN` reporter — Filed from a collision-and-seating
  fix pass on `/Game/Maps/PW_VegetationTest` (editor build 10:44; plugin source HEAD `962275fa`),
  triggered by the user report "why does grass have player collision?". Answering it needed a
  per-component census of `collisionEnabled` / profile / object type / per-channel responses across
  312 components, which `actor.get_components` cannot supply and `actor.get_component_property`
  supplies one field at a time; the pass wrote `dev/grasscollide/gc_census.py` instead. Field list
  above verified line by line against `ComponentHandler.cpp:502-541` — the `USceneComponent` branch
  at `:513` is the handler's only type-specific branch and there is no `UPrimitiveComponent` cast in
  the file. Dedup: searched the board for `get_components`, `collision_enabled`/`collisionEnabled`,
  `collisionProfileName`, `object_type`/`objectType`, `BodyInstance`, "which components", and every
  `E-actor-*` / `E-component-*` file. Nothing asks for collision state on a component readback.
  Three claims from the field report were narrowed before filing: "no verb anywhere reports it" is
  false — `editor.set_view_mode` publishes an aggregated per-actor `collisionEnabled` /
  `respondsToChannel` (`ViewportHandler.cpp:326-327`) and `actor.get_component_property` names
  `BodyInstance.CollisionEnabled` in its own parameter doc, so the body states both; and the report's
  "18 components blocked Visibility while ignoring Pawn" is **12** by the census
  (`out/census_after.json`), all of them components the pass itself diverged, with 83 of 147 rows
  differing between the two channels once the 71 disabled-collision rows are counted. Added on top
  of the original finding, because it changes what the block must contain: 71 rows read
  `NO_COLLISION` with `respPawn: ECR_BLOCK`, so publishing responses without the enable state would
  be a confidently wrong census.
