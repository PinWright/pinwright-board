---
id: E-get-component-property-no-subfield-select
title: "actor.get_component_property can't scope to a struct sub-field (property.get already can via dotted path, but only by object-path/UPROPERTY addressing) — reading one scalar by friendly component name returns the whole struct and spills to disk"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [actor, get_component_property, response-size, struct, subfield, docs]
---

# actor.get_component_property has no way to read a single sub-field of a struct property (its sibling property.get already does — but addresses by object path / UPROPERTY, not friendly component name)

`actor.get_component_property` (`Handlers/Actor/ComponentHandler.cpp:462`) takes
`actorName` + `componentName` + a single `propertyName`, looks up that one
`FProperty`, and serializes its **entire** value via `ExportPropertyToJsonValue`.
There is no way to ask for just one scalar field *inside* a struct-valued
property — no dotted `propertyName` (`BodyInstance.CollisionEnabled`), no
field allow-list, no `nameMatch` on struct members.

**Important context — the dotted struct sub-field read already exists, on the
sibling verb `property.get`.** `property.get`
(`Handlers/Utility/UtilityPropertyHandler.cpp:1232`) advertises `propertyName`
"supports nested paths with dots" and resolves it through
`ResolveNestedPropertyPath` (`Utils/PropertyInspection.cpp:26`), which hops both
`FObjectProperty` (into a subobject) and `FStructProperty` (into a struct member)
segments and returns just the leaf — so
`property.get { objectPath: <component object path>, propertyName: "BodyInstance.CollisionEnabled" }`
already returns the single enum, well under the spill threshold, today with zero
new code. The genuine remaining gap is therefore **narrower than "no sub-field
read exists anywhere"**: `actor.get_component_property` addresses a component by
its friendly runtime name (`StaticMeshComponent0`, via `Component->GetName()`),
which is the ergonomic entry point an agent already on the `actor.*` route
reaches for — and *that* verb has no sub-field selection, while the verb that has
sub-field selection (`property.get`) requires the full component object path or a
dotted walk through the component's UPROPERTY name (the orthogonal
component-path-discovery gap tracked by the OPEN
[E-property-route-no-component-path-discovery](E-property-route-no-component-path-discovery.md)).
So bridging the dotted-path read onto `actor.get_component_property`'s friendly
component-name addressing is the real, novel, unaddressed work here.

This collides head-on with the (correct) fix in
[B-properties-struct-export-text-fallback](B-properties-struct-export-text-fallback.md):
struct properties like `FBodyInstance` are now fully decomposed into a structured
JSON object (`AngularDamping`, `bAllowTickOnDedicatedServer`, the collision
profile, all of it) instead of an opaque export-text string. That decomposition
is the right behavior for a dump, but it makes the whole-struct read **large** —
the full `FBodyInstance` JSON exceeds the 10000-char direct-HTTP spill threshold
([E-http-response-spill](E-http-response-spill.md)), so a read intended to confirm
one boolean (`CollisionEnabled == NoCollision`) writes a file under
`Saved/EditorAutomation/HttpResponses/` and the agent has to grep that file for
the one field it wanted.

This is the *struct sub-field* analogue of [E-component-read-filter](E-component-read-filter.md)
(DONE): that ticket added `nameMatch`/`componentClass` to choose **which
components** to read; this is the orthogonal "which **field of one property**"
axis. It is also distinct from
[E-property-route-no-component-path-discovery](E-property-route-no-component-path-discovery.md)
(component-path *discovery*, not field selection) — here the agent already knew
the component (`StaticMeshComponent0`) and the property (`BodyInstance`); the only
gap is that it could not narrow to `BodyInstance.CollisionEnabled`.

## Evidence (this task — `actor.set_collision` staging, outcome clean)

A "disable collision on two placed UELogo props, then confirm" task. Both
verify-readbacks hit the spill:

- `actor.get_component_property UELogo StaticMeshComponent0.BodyInstance` →
  `outputTooLong: written to disk; read NoCollision from file`
- `actor.get_component_property UELogo2 StaticMeshComponent0.BodyInstance` →
  `outputTooLong: written to disk; read NoCollision from file`

Friction note (verbatim): *"the only wrinkle was actor.get_component_property on
BodyInstance exceeding the 10000-char display threshold and spilling to a
HttpResponses JSON file, so I had to grep that file for CollisionEnabled to
confirm NoCollision instead of reading it inline."*

The intent — read one scalar to confirm a write took — is the common
read-after-write check; a full-struct spill + manual grep is disproportionate
friction for it. Both confirmation steps in the task paid this toll.

## What it should do

Let the caller scope `actor.get_component_property` to a sub-field of a
struct-valued property so the response is the one value, not the whole struct:

1. **Dotted `propertyName`** — accept `propertyName: "BodyInstance.CollisionEnabled"`
   and walk the reflected struct (one `FStructProperty` hop per segment), returning
   just the leaf value. **Reuse the existing `ResolveNestedPropertyPath`**
   (`Utils/PropertyInspection.cpp:26`) — the same resolver `property.get`/`property.set`
   already use — rather than re-deriving a struct walk; when `propertyName` contains a
   `.`, resolve via `ResolveNestedPropertyPath(Component, propertyName, …)` and export
   the returned leaf from the returned container. Smallest viable slice; directly kills
   this spill (one enum value, well under 10 KB). Keeps the friendly component-name
   addressing the agent already used.
2. **Optional field filter** — mirror `property.list`'s vocabulary
   (`nameMatch` / `propertyNames` allow-list, `Handlers/Utility/UtilityPropertyHandler.cpp`)
   onto `get_component_property` so a struct read can be narrowed to named members.

Both are additive/optional; default (whole-property) behavior is unchanged.

## Docs steer (ships now, no build)

Amend the `actor.get_component_property` section of `docs/wiki-src/actor.md`
(tag `docs`) to warn that reading a struct UPROPERTY (`BodyInstance`, `BodySetup`,
`SensesConfig`, any `F...Instance`) returns the full decomposed struct and will
spill to `Saved/EditorAutomation/HttpResponses/` past the ~10 KB threshold — for
a single read-after-write confirmation prefer the typed verb that returns the
scalar (e.g. read collision via `actor.set_collision` round-trip / the collision
profile readback) or, once available, a dotted `propertyName`. The page currently
has no note about the spill behavior on struct properties.

**Workaround (today, zero new code):** route the read through `property.get`'s
dotted path instead of `actor.get_component_property` — `property.get
{ objectPath: <component object path or actor>, propertyName: "BodyInstance.CollisionEnabled" }`
resolves the nested segment via `ResolveNestedPropertyPath` and returns only the
leaf enum, far under the 10 KB threshold, so it never spills. The catch is that
`property.get` needs the component's full object path or a dotted walk through its
UPROPERTY name rather than the friendly `StaticMeshComponent0` runtime name (the
discovery gap in `E-property-route-no-component-path-discovery`). Failing that,
after a whole-struct `get_component_property` spills, read the written
`HttpResponses/<...>.json` artifact and grep for the wanted field
(here: `CollisionEnabled`), as the originating task did.

## History
- `#2-reword-and-fix` `IN-REVIEW` developer — Reword + implement (Fix #1, dotted `propertyName`). Reword: title/body now state the dotted struct sub-field read already exists on the sibling `property.get` (`UtilityPropertyHandler.cpp:1232` → `ResolveNestedPropertyPath`, `PropertyInspection.cpp:26`), so the real gap is bringing it to `actor.get_component_property`'s friendly component-name addressing (`Component->GetName()`), distinct from the object-path/UPROPERTY addressing `property.get` requires and from the OPEN `E-property-route-no-component-path-discovery`; Workaround now points at `property.get`'s dotted route; Fix #1 now reuses `ResolveNestedPropertyPath` rather than re-deriving a walk. Fix: `Handlers/Actor/ComponentHandler.cpp` `actor.get_component_property` — when `propertyName` contains a `.`, resolve the nested path against the matched component via `ResolveNestedPropertyPath(Component, PropertyName, …)` and export only the leaf via `ExportPropertyToJsonValue`; non-dotted path unchanged (whole-property, default behavior preserved). Param doc updated to note dotted nested paths are supported; added `#include "Utils/PropertyInspection.h"`. Regression test: `Private/Tests/Actor/TestGetComponentPropertyNestedPath.cpp` — exercises production `ResolveNestedPropertyPath` + `ExportPropertyToJsonValue` on a transient `USceneComponent`, reading the leaf `PrimaryComponentTick.bCanEverTick` (scalar bool) and asserting it is a single JSON boolean, NOT the whole decomposed `FActorComponentTickFunction` object (which is what a non-dotted read returns and what would spill); fails if the dotted-leaf routing is reverted. Did not compile/run (later phase). Fix #2 (field-filter vocabulary) left for a follow-up; not implemented. Released claim.
- `#1-initial-audit` `OPEN` reporter — Process-friction audit of an `actor.set_collision` staging task (outcome clean). `actor.get_component_property` has no struct sub-field selection: reading `BodyInstance` to confirm one scalar (`CollisionEnabled == NoCollision`) serializes the whole decomposed `FBodyInstance` (post-[B-properties-struct-export-text-fallback]), exceeds the 10000-char spill threshold ([E-http-response-spill]), writes to `Saved/EditorAutomation/HttpResponses/`, and forced a manual grep of that file. Both readbacks (UELogo, UELogo2) hit it; friction note quoted verbatim in body. Proposed: dotted `propertyName` (`BodyInstance.CollisionEnabled`) and/or a `nameMatch`/`propertyNames` field filter on `get_component_property` (additive), plus a `docs/wiki-src/actor.md` overlay note (tag `docs`). Dedup: ripgrep across OPEN/closed — `E-component-read-filter` (DONE) filters *which components*, not struct sub-fields; `E-property-route-no-component-path-discovery` (OPEN) is component-path discovery, not field selection; `E-http-response-spill`/`E-asset-dump-oversized-fields` are the spill *mechanism* / dump-side size caps, not a read-time field selector on this verb; `B-properties-struct-export-text-fallback` (DONE) is the struct-decomposition fix this friction is a downstream consequence of. No existing ticket covers sub-field selection on `actor.get_component_property`.
