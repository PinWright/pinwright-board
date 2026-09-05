---
id: E-inspect-filter-no-property-value-match
title: "system.inspect.find_objects_by_class's filter matches only name, class name and object path — counting components by their assigned asset (DecalMaterial, Niagara Asset) needs one property read per returned row"
status: OPEN
severity: Low
category: ergonomic
tags: [system-inspect, find_objects_by_class, filter, property-filter, projection, components, decals, niagara, weapons]
encounters: 1
lastSeen: 2026-09-06T00:00:00Z
---

# The filter can select by what a component is called, never by what it points at

`system.inspect.find_objects_by_class` is the live-instance discovery verb. Its `filter` is
documented as, and is, an identity match only:

> "Pattern matched against each instance's name, class name, and full path; an instance is kept if
> any of the three matches."

`Plugins/PinWright/Source/PinWright/Private/Handlers/Environment/EnvironmentHandler.cpp:1316`

For components, the question a caller usually has is not what the instance is named — engine-assigned
names like `DecalComponent_7` carry no information — but **which asset it is configured with**.
That is a property on the instance, and the filter cannot see properties.

## What this costs — measured

Counting the decals of one material means reading `DecalMaterial` on every returned row:

```
system.inspect.find_objects_by_class {"className": "DecalComponent"}     -> N rows
property.get <each row's path> {"propertyName": "DecalMaterial"}         -> N calls
```

Same shape for Niagara components of one system (`Asset`), static mesh components of one mesh
(`StaticMesh`), audio components of one cue (`Sound`). The verb narrows by class and then hands back
everything of that class, so the actual selection happens client-side at one round trip per
candidate. The larger the world, the worse the ratio — and the filter that *would* have cut the set
down is already accepted, already documented, and simply matches the wrong thing for this intent.

The neighbouring parameters do not help: `includeDerived` narrows by class, `limit` truncates
(making a count wrong rather than cheap), and there is no `fields` projection on this verb that
could at least return the property alongside the row.

## What is asked for

Either, and both are small:

1. **A property-valued filter.** Something in the shape the surface already uses elsewhere — e.g.
   `propertyFilter: {"DecalMaterial": "/Game/FPS/FX/Materials/MI_Decal_Impact"}`, matched with the
   same case-insensitive-substring default and the same `matchMode` / `caseSensitive` modifiers
   `filter` already takes (`EnvironmentHandler.cpp:1317-1318`), so the vocabulary stays consistent.
   Matching an object-typed property against its asset path covers every case listed above with one
   rule.
2. **Or, failing that, a `fields` projection** so the property comes back on the row and the caller
   filters client-side without a second call per instance. Strictly worse than (1) for counting, but
   it collapses N+1 calls into 1 and the projection mechanism already exists on the sibling verbs
   (`actor.list`, `list_objects`).
3. **Or, at minimum, say so on the verb page.** The parameter help states the three things `filter`
   matches; it does not state that a property value is *not* among them, and the natural reading of
   "an instance is kept if any of the three matches" is a list of conveniences rather than a
   boundary. One sentence — "`filter` never inspects property values; to select by assigned asset,
   read the property per row" — turns a discovery into documentation.

## Severity

**Low** — pure friction, the rubric's bottom band. Nothing is wrong, nothing is blocked, and the
answer is exact; it costs N extra calls to get. No reach bump: `find_objects_by_class` is a
discovery verb reached for deliberately, not an every-session call, and the property-read fallback
is obvious once the limitation is known. The ask is ordered above so the cheapest option (3) is a
docs line, which is the right size for the impact.

## Related

- `F-inspect-find-non-actor-uobject` (IN-REVIEW) — **the ticket that created this verb.** Its `#2`
  shipped `find_objects_by_class` with the filter as it stands. This is a follow-on capability ask on
  what shipped, not a failure of it, so it is filed separately rather than appended.
- `E-inspect-find-by-class-no-limit-spills`, `E-inspect-list-objects-no-limit-spills` — the same
  family of response-shaping gaps on the sibling inspect verbs (limits and spills); this one is about
  *selection*, not volume, though a working property filter would reduce spill as a side effect.
- `E-component-read-filter` (DONE) — the precedent for exactly this ask on a different verb family:
  `nameMatch` + `componentClass` were added to `actor.describe` / `actor.get_components` /
  `blueprint.scs.get` through one shared filter helper. Note both of those are still *identity*
  filters — the property-value axis this ticket asks for was not part of that work on any verb.
- `B-actor-list-filter-case-mismatch` — a defect in the shared filter contract this verb inherits
  (`EnvironmentHandler.cpp:1339` records the contract as shared with `list_objects` / `actor.list`),
  so a fixer touching the filter should read both.

## History
- `#1-filed` `OPEN` WEAPONS-critic — Filed during a WEAPONS critic review round 4. `system.inspect.find_objects_by_class`'s `filter` matches instance name, class name and object path only — confirmed verbatim in the param help at `Handlers/Environment/EnvironmentHandler.cpp:1316` — so a component cannot be selected by the asset it is configured with. Counting the decals of one material (`DecalMaterial`) or the Niagara components of one system (`Asset`) therefore costs one `property.get` per returned row after the class query, and the same applies to static mesh components by `StaticMesh` and audio components by `Sound`; engine-assigned instance names like `DecalComponent_7` carry no information, so the identity filter cannot substitute. The neighbouring params do not help — `includeDerived` narrows by class, `limit` truncates (which makes a count wrong rather than cheap), and this verb has no `fields` projection that could at least return the property on the row. Ask, cheapest last: a property-valued filter using the vocabulary already present (`propertyFilter: {Prop: value}`, reusing the case-insensitive-substring default and the `matchMode`/`caseSensitive` modifiers at `:1317-1318`), matching object-typed properties against their asset path so one rule covers every case; or a `fields` projection collapsing N+1 calls into 1, as the sibling verbs already have; or one sentence on the verb page stating that `filter` never inspects property values, since the current help lists what it matches without saying that is a boundary. Dedup: `F-inspect-find-non-actor-uobject` (IN-REVIEW) is the ticket that created this verb and shipped the filter as it stands — this is a follow-on capability ask on what shipped, not a failure of it; `E-inspect-find-by-class-no-limit-spills` / `E-inspect-list-objects-no-limit-spills` are response-volume gaps on the siblings, not selection; `E-component-read-filter` (DONE) is the precedent for adding filters to a verb family but added only identity filters (`nameMatch`, `componentClass`), never a property-value axis, on any verb; `B-actor-list-filter-case-mismatch` is a defect in the shared filter contract this verb inherits (`EnvironmentHandler.cpp:1339`), worth reading together. Severity Low, pure friction: nothing wrong, nothing blocked, the answer is exact and costs N extra calls, and no reach bump since this is a deliberately-reached discovery verb.
