---
id: B-property-path-silent-world-fallback
title: "property.list/get silently resolve an unresolvable level-path tail to the World instead of OBJECT_NOT_FOUND"
status: IN-REVIEW
severity: High
category: bug
tags: [property, object-resolution, silent-success, wrong-object]
---

# property.* object resolver silently climbs to the World on an unresolvable level-path tail

When `property.list` (and `property.get`) is given a full level-style object
path of the form `…:PersistentLevel.<Actor>.<Subobject>` whose trailing
segment(s) do **not** resolve (a typo'd / nonexistent subobject name, or even a
nonexistent actor name), the resolver does **not** return `OBJECT_NOT_FOUND`.
Instead it silently drops the unresolvable tail and climbs all the way up to the
`World` object, then returns `ok: true` with the World's property list (and
`existsAfter: true`). The response's `objectPath`/`className` are changed to the
World, but a caller who trusts the success status and `count` is handed a
property set for a completely different object than it asked about.

This is the "silent success / wrong-object" class of tool bug. It is especially
misleading for `property.list` because the call returns a clean success with a
populated property array, so an agent doing discovery before a `property.set`
can easily believe it is inspecting the fog component when it is actually
reading the World. (Observed in the field: an attempt that guessed the wrong
component subobject name `ExponentialHeightFogComponent0` got a silent
World fallback and only noticed because the property names looked wrong.)

The resolver clearly *has* a correct error path — a bare bad name
(`TotallyBogusActorName_12345`) returns a clean `[OBJECT_NOT_FOUND]`, and the
attempt log shows the plainer dotted form `DirectionalLight_0.DirectionalLightComponent`
also returns `[OBJECT_NOT_FOUND]`. It is specifically the level-qualified
(`…:PersistentLevel.…`) form with an unresolvable tail that bypasses the error
and falls back to the World.

## What it should do

An unresolvable trailing path segment should fail with `OBJECT_NOT_FOUND`
naming the segment that could not be resolved — never silently substitute a
different (ancestor) object and report success. At minimum the resolver must not
return an object whose path differs from the requested leaf without erroring.

## Verbatim repro (live, replay-confirmed)

World/level under test: `/Game/Global/DemoRoom/TestRoom.TestRoom` (open editor
world; actors `ExponentialHeightFog_0`, `DirectionalLight_0` placed).

Bad SUBOBJECT tail — silently returns the World:
```
call("property.list", {
  "objectPath": "/Game/Global/DemoRoom/TestRoom.TestRoom:PersistentLevel.ExponentialHeightFog_0.ExponentialHeightFogComponent0",
  "includeValues": false, "includeDefault": false,
  "includeOverrideState": false, "includeMetadata": false })
-> ok:true
   {"objectPath":"/Game/Global/DemoRoom/TestRoom.TestRoom","className":"World",
    "properties":[],"count":0,"assetClass":"World","existsAfter":true}
```

Bad ACTOR tail — same silent World fallback:
```
call("property.list", {
  "objectPath": "/Game/Global/DemoRoom/TestRoom.TestRoom:PersistentLevel.NonExistentActor_99.Whatever" })
-> ok:true  {"className":"World","count":8, ...}   # World props, NOT an error
```

`property.get` shares the underlying resolver flaw (resolves the same bad path
to the World, then fails downstream at the property lookup against the World):
```
call("property.get", {
  "objectPath": "/Game/Global/DemoRoom/TestRoom.TestRoom:PersistentLevel.ExponentialHeightFog_0.TotallyBogusSubobjectXYZ",
  "propertyName": "FogDensity" })
-> [PROPERTY_NOT_FOUND] Property FogDensity not found on object /Game/Global/DemoRoom/TestRoom.TestRoom
```
Note the error names `…TestRoom.TestRoom` (the World) — proof the bad subobject
segment was silently dropped and the World substituted before the property lookup.

Correct-error contrast (the error path works for a bare bad name):
```
call("property.list", { "objectPath": "TotallyBogusActorName_12345" })
-> [OBJECT_NOT_FOUND] Unable to find object at path TotallyBogusActorName_12345.
```

Step ladder showing the resolver reports the last-resolved ancestor, not the leaf:
- `…:PersistentLevel`                                  -> className `Level`   (ok)
- `…:PersistentLevel.ExponentialHeightFog_0`           -> the actor           (ok, correct)
- `…:PersistentLevel.ExponentialHeightFog_0.<bad>`     -> className `World`    (silent wrong object)

## Fix sketch

In the shared `property.*` object resolver (PropertyUtils / path resolution),
when a trailing path component fails to resolve against its parent, return
`OBJECT_NOT_FOUND` for the requested path rather than returning the
last-successfully-resolved ancestor (or `StaticFindObject`'s World/outer
fallback). Guard against the `:PersistentLevel.<…>` form specifically, where the
unresolvable tail currently climbs to the World.

## History
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed on live editor: a `…:PersistentLevel.<Actor>.<bad-subobject>` (and `…:PersistentLevel.<bad-actor>.<x>`) path to `property.list` returns `ok:true` with the World's properties instead of `OBJECT_NOT_FOUND`; `property.get` resolves the same path to the World and fails downstream with `PROPERTY_NOT_FOUND` against `TestRoom.TestRoom`. A bare bad name (`TotallyBogusActorName_12345`) correctly errors `OBJECT_NOT_FOUND`, so only the level-qualified form bypasses the error path.
- `#2-resolver-ancestor-guard` `IN-REVIEW` developer — Root-caused to the shared `property.*` resolver `ResolveObjectForProperty()` in `Source/EditorAutomationRpcGateway/Private/Handlers/Utility/UtilityPropertyHandler.cpp`: its direct `FindObject(nullptr, *TrimmedPath)` hit (and the fall-through `StaticLoadObject`) accepted UE's `ResolveName` ancestor fallback — on a `…:PersistentLevel.<Actor>.<bad-tail>` path the trailing segment fails and the lookup climbs to the last resolvable outer (the World), which was returned unchecked. Added an `IsAncestorFallback(Resolved, RequestedPath)` guard: reject any resolved object whose `GetPathName()` is a strict prefix of the requested path followed by a `:`/`.` subobject delimiter (i.e. an ancestor, not the requested leaf), and drop it so resolution falls through to `OBJECT_NOT_FOUND`. Exact-leaf hits (equal-length path) and the unrelated Blueprint→CDO redirect, asset-path normalization, and actor-by-name modes are untouched. Regression test `Source/EditorAutomationRpcGateway/Private/Tests/Utility/TestPropertyResolverAncestorFallback.cpp` (`property.resolver.AncestorFallbackErrors`) spawns a real actor into the editor world and asserts a bad-subobject tail, a bad-actor tail, and a `property.get` bad tail all return `OBJECT_NOT_FOUND` (never `ok:true`/`className:World`), with a positive control that the real actor path still resolves to the actor; reverting the guard fails it.
- `#3-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 2 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
