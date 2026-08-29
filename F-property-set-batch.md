---
id: F-property-set-batch
title: "property.set is one property per call with no generic batch form — authoring one PCG graph cost ~110 calls, while actor.set_component_properties proves the multi-property request shape already exists in this codebase"
status: OPEN
severity: Medium
category: feature
tags: [property, property-set, batch, authoring, pcg, reflection, ergonomic, call-count]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
---

# The generic reflection writer has no batch form, and it is the one that most needs one

`property.set` takes exactly one `propertyName` and one `value`
(`Handlers/Utility/UtilityPropertyHandler.cpp:1020-1026` — `objectPath`, `propertyName`, `value`,
`markDirty`). There is no `property.set_many`, no array-of-pairs form, and no way to write two
properties on one object in one round trip.

Authoring one PCG graph this session took roughly **110 individual `property.set` calls**. That is
not a pathological case: `property.set` is the fallback writer for every namespace that has no typed
setter, so the object with the most properties to configure is always the one with the least typed
support.

## The request shape already exists in this codebase

`actor.set_component_properties` takes a whole property bag in one call —
`RPC_PARAM_REQ("properties", "object", "Map of UPROPERTY names to JSON values; …")`
(`Handlers/Actor/ComponentHandler.cpp:187-193`), applied in one loop with per-property failures
collected into `warnings` (`:240`, `:311`, `:340`, `:356`). `actor.add_component` takes the same bag
as an optional param (`:48`). So the shape, the JSON parse, the per-entry error channel and the
"one write pass, one notification, one dirty" flow are all already written and shipping. The generic
writer simply never got the form.

**This is a shape precedent, not a quality one, and the ticket says so deliberately.**
`actor.set_component_properties` carries three known defects, all filed, all now `DONE`, all `High`:

- `B-set-component-properties-no-change-notification` (DONE, High, `encounters: 3`) — it stored
  UPROPERTYs by reflection and fired no `PostEditChangeProperty`, so every property whose effect
  lives in that override was a silent no-op.
- `B-set-component-properties-drops-warnings` (DONE, High) — it built the `PropertyWarnings` array
  and dropped it, so per-property failures vanished under an unqualified success, contradicting the
  verb's own registered summary. The dropped-warnings code path is still commented at
  `ComponentHandler.cpp:383`.
- `B-set-component-properties-staticmesh-shadow-corruption` (DONE, High) — it wrote `StaticMesh` by
  raw `FProperty` store, leaving the engine's shadow copy stale.

Every one of those is a *batch-specific* failure: a bag verb that loops, swallows per-entry errors,
and skips per-object engine notification. A `property.set` batch that copies the shape without
copying those lessons reintroduces three High bugs at once.

## The constraint a designer must handle: one notification per object, not one per property

`B-property-set-object-hop-notification-noop` (filed this session, OPEN, High) shows the
**single**-property path already notifies the wrong object when the dotted path crosses a `UObject`
boundary: `ResolveNestedPropertyPath` hops the container onto the inner object
(`Utils/PropertyInspection.cpp:336-337`) while `NotifyReflectedPropertyChanged` dispatches to
`RootObject` (`Handlers/Utility/UtilityPropertyHandler.cpp:292`, `:300-303`), so the inner object's
`PostEditChangeProperty` override never runs — with `applied: true`, `markedDirty: true` and a
matching `property.get` all agreeing.

A batch verb must not multiply that. The requirement is **one correct notification per affected
object**, not one per property and not one for the batch:

- N properties on one object collapse to one `PostEditChangeProperty` on that object — cheaper than
  today and correct.
- N properties whose dotted paths land on *different* objects need N distinct notifications, each
  addressed to the object the value actually landed on. A batch that notifies `RootObject` once for
  the whole bag is the object-hop bug with an N× multiplier.

That ordering also interacts with the existing `markDirty=false` restore, which the current handler
documents must run last (`UtilityPropertyHandler.cpp:1252-1256`).

## Precedent: the identical argument, one namespace over

`F-vehicle-wheel-asset-batch-properties` (OPEN, Low, `encounters: 2`) makes structurally the same
case for `vehicle.*`:

> But once the asset exists, the only way to write properties back is
> `vehicle.set_wheel_asset_property`, whose wiki page is explicit that it **"Write[s] a single
> property on a wheel-asset CDO"** — one `propertyName`/`value` pair per call, each defaulting to
> `save:true`.

and names a batched sibling that proves the shape but cannot reach the target:

> The sibling `vehicle.set_suspension` *is* exactly this batched suspension verb […] but it only
> resolves a `UChaosWheeledVehicleMovementComponent` host (`componentPath`), so it cannot touch a
> **standalone** wheel asset — leaving standalone wheel assets with no batched edit at all.

Verified verbatim. The relationship of that ticket to this one is containment, not duplication: a
generic `property.set` batch would give `vehicle.*` its batched standalone-asset edit for free, and
the domain ticket can then close as covered or narrow to the typed ergonomics it still wants.

## Where the evidence lives

`F-pcg-set-node-property` (OPEN, High) defers explicitly to `property.set` as the existing generic
route. Its `#1`, verbatim:

> `property.set` (`Handlers/Utility/UtilityPropertyHandler.cpp`) is a generic reflection setter that
> works on arbitrary `UObject`s by `objectPath`; a PCG node's `UPCGSettings` sub-object has a real
> object path surfaced by `pcg.inspect`, so editing node settings is *reachable* today
> (if unpolished).

The ~110-call figure is the direct quantification of *"unpolished"*.

Two qualifications, both from that ticket's own `#2` and both load-bearing here. First, its `#1`
rationale is **superseded**: `pcg.inspect` emits `settingsClass` — a class path, not the settings
object's path (`PCGGraphInspect.cpp:47`, `:50-54`) — so the caller must hand-construct
`<graphPath>:<nodeName>.<settingsSubobjectName>`. Second, that is why `F-pcg-set-node-property` was
re-severitied Low → High. So the ~110 calls were paid on top of a path the caller had to reconstruct
by hand; the batch form would not fix that half, and this ticket does not claim it does.

## What it should do

```
property.set_many(objectPath, properties /* { "Name": value, "Nested.Path": value, … } */,
                  markDirty?)
  -> { applied: [{property, value}], warnings: [...],
       notifiedObjects: [<paths actually notified>], markedDirty, pendingSave }
```

Reuse `actor.set_component_properties`' bag parse and `warnings` channel rather than a new parser;
resolve every path first, group the resolved leaves by the object each landed on, write, then notify
once per distinct object. `notifiedObjects` is not decoration — it is what makes the object-hop
constraint above verifiable from the response instead of by reading source.

## Dedup

Board search across all statuses for `property.set_many`, `property.set_batch`, "multi-property",
"batch property write", "generic batch property" returns **nothing**. The 16 `F-*batch*` tickets on
the board are every one of them domain-scoped, so the generic gap is real rather than assumed:

`F-actor-batch-tag-set` (tags), `F-add-mapping-batch-keys` (input mappings),
`F-add-variables-batch` (blueprint variables), `F-animation-set-curve-key-no-batch` (curve keys),
`F-batch-pin-defaults` (DONE — pin defaults), `F-console-batch-get-cvar-values` (cvar reads),
`F-effect-draw-debug-shapes-batch` (debug shapes), `F-gameplay-tags-add-no-batch` (gameplay tags),
`F-geometry-boolean-batch-tools` (boolean ops), `F-geometry-lod-settings-batch` (LOD ladder),
`F-geometry-uv-prep-pipeline-batch` (UV pipeline), `F-graph-batch-delete-clear-mode` (node delete),
`F-sequencer-batch-keyframes` (IN-REVIEW, Medium — keyframes),
`F-spatial-raycast-no-batch-multi-origin` (rays), `F-texture-sampling-settings-batch` (texture
sampling settings), `F-vehicle-wheel-asset-batch-properties` (wheel-asset properties).

Not one of them is the reflection writer. `F-batch-pin-defaults` (DONE) is the standing proof that
this board accepts and lands batch forms.

## Severity

**Impact = Medium, and it fits the rubric literally.** Medium is *"soft blocker. Doable, but only via
a documented workaround, a source dive, or **many extra calls**"*. One PCG graph cost ~110 calls.
Nothing is wrong, nothing is blocked, and every individual write round-tripped — this is call count
and nothing else, which rules out High (no silent wrong data, no false success) and rules out the
High/Medium "hard blocker" row (the task completed).

**Reach modifier: bump-up to High declined, and here is what is being declined.** `property.set`
itself does run in almost every session, which by the letter of the modifier would bump this to High.
Two reasons not to take it. (a) The *method* is every-session; the *gap* is not. Writing one or two
properties — the every-session shape — costs one or two calls and feels like nothing. The tax only
becomes visible in a configure-many-properties-on-one-object burst, which is the PCG / asset-authoring
shape, not the daily one. The modifier exists to reorder work by how often the **gap** is hit, and
weighting by method frequency here would overstate it. (b) High is where this board keeps
silent-wrong-data tickets, including the three `set_component_properties` bugs cited above and
`B-property-set-object-hop-notification-noop` on this very verb. Putting pure call-count friction in
the same band would push a ticket where the caller is being lied to behind one where they are merely
typing more.

**Net: Medium** — above the Low band the domain batch tickets sit in, because those each cost 4-8
calls in one namespace while this one cost ~110 in the namespace every other namespace falls back to.

## Same shape as

- `F-vehicle-wheel-asset-batch-properties` (OPEN, Low) — the identical argument for `vehicle.*`;
  containment, not duplication.
- `F-texture-sampling-settings-batch` (OPEN, Low) — same "4 fields, 4 write+save round trips" shape.
- `F-batch-pin-defaults` (DONE) — the landed precedent that batch forms get built here.

## Related

- `F-pcg-set-node-property` (OPEN, High) — where the ~110-call evidence came from, and the ticket
  that names `property.set` as the current route.
- `B-property-set-object-hop-notification-noop` (OPEN, High) — the correctness constraint any batch
  design must satisfy; a batch must not multiply it.
- `B-set-component-properties-no-change-notification` / `-drops-warnings` /
  `-staticmesh-shadow-corruption` (all DONE, High) — the three ways the existing bag verb got it
  wrong, i.e. the regression list for a new one.
- `F-property-set-omit-oversized-echo` (filed this session) — the other half of the same authoring
  loop: what each of those ~110 calls echoes back.

## History
- `#1-110-calls-for-one-pcg-graph` `OPEN` reporter — Authoring one PCG graph on host project EAContentExamples58 took roughly 110 individual `property.set` calls. `property.set` is one `propertyName`/`value` pair per call with no batch form (`Handlers/Utility/UtilityPropertyHandler.cpp:1020-1026`), and it is the fallback writer for every namespace without a typed setter, so the objects with the most to configure are the ones with the least typed support. The multi-property request shape is already shipping in this codebase: `actor.set_component_properties` takes `RPC_PARAM_REQ("properties", "object", …)` and applies the bag in one loop with a per-property `warnings` channel (`Handlers/Actor/ComponentHandler.cpp:187-193`, `:240`, `:311`, `:340`, `:356`), and `actor.add_component` takes the same bag optionally (`:48`) — cited as a SHAPE precedent only, because that verb carries three DONE High defects (`B-set-component-properties-no-change-notification`, `-drops-warnings`, `-staticmesh-shadow-corruption`), all of them batch-specific failure modes a new bag verb would reinherit. Design constraint from this session's other finding: `B-property-set-object-hop-notification-noop` (OPEN, High) shows the single-property path already notifies `RootObject` when the dotted path hops onto a different `UObject` (`Utils/PropertyInspection.cpp:336-337` vs `UtilityPropertyHandler.cpp:292`, `:300-303`), so a batch needs ONE CORRECT NOTIFICATION PER AFFECTED OBJECT — not one per property, not one for the bag — and should publish `notifiedObjects` so that is checkable from the response. Precedent: `F-vehicle-wheel-asset-batch-properties` (OPEN, Low, encounters 2) makes the structurally identical argument for `vehicle.set_wheel_asset_property`, with `vehicle.set_suspension` as a batched sibling that cannot reach a standalone asset — verified verbatim; a generic batch contains it. Evidence source: `F-pcg-set-node-property` (OPEN, High) `#1` defers to `property.set` as the generic route "reachable today (if unpolished)" — verified verbatim — and its `#2` supersedes the rest of that rationale (`pcg.inspect` emits `settingsClass`, a class path, not the settings object path: `PCGGraphInspect.cpp:47`, `:50-54`), so the ~110 calls were paid on a hand-reconstructed path; this ticket does not claim to fix that half. Dedup: board search for `property.set_many` / `property.set_batch` / "multi-property" / "batch property write" returns nothing, and all 16 `F-*batch*` tickets are domain-scoped (enumerated in the body), so the generic gap is real, not assumed. Severity Medium — the rubric's "doable, but only via many extra calls" fits literally; the every-session reach bump-up to High is declined because the method is every-session while the gap is the authoring-burst shape, and because High on this board holds silent-wrong-data tickets on this same verb.
