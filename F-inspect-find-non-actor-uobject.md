---
id: F-inspect-find-non-actor-uobject
title: "system.inspect can't discover live non-actor UObjects (widget-owned helpers, plain UObjects) — actor-only enumeration blocks object.call_function on them"
status: IN-REVIEW
severity: Medium
category: feature
tags: [system-inspect, uobject, discovery, non-actor, object-call-function, pie, runtime]
encounters: 1
lastSeen: 2026-07-03T00:00:00Z
---

# system.inspect can't discover live non-actor UObjects

`system.inspect.list_objects` and `system.inspect.find_by_class` enumerate only
Actors — both loop `TActorIterator<AActor>` over the resolved world
(`EnvironmentHandler.cpp:1421` and `:1527`). A live **non-actor** UObject — e.g.
a UMG-widget-owned helper like `UAppPhysicsConfigBridge`, or any plain UObject —
is invisible, so there is no MCP path to obtain its object reference. That ref is
the one missing input to `object.call_function` (`F-object-call-function`, DONE),
which can already invoke a UFUNCTION on any object path *once you hold the path*.
Discovery is the unsolved half.

The adjacent discovery verbs don't cover it: `list_subsystems`
(`F-inspect-list-subsystems`, DONE) enumerates subsystem arrays; the singletons
handler (`F-inspect-game-singletons`) covers GameInstance/GameMode/PC/etc.;
`search_classes` (`F-search-api-native-uclasses`, DONE) searches UClass
*definitions*, not live instances; `inspect_object` resolves a UObject you can
already name. A widget-owned helper is none of those. This is exactly the code
gap `F-wiki-runtime-uobject-inspection` (DONE, docs-only) deferred to a sibling
ticket, and that `B-inspect-misses-pie-world` (DONE) did not touch — its fix
added PIE *actors* via `TActorIterator`, still actor-only. Also note
`list_objects` has no class filter today (only a name-substring `filter`); a
class filter is part of the ask.

**Workaround:** `python.execute` (walk `GetObjectsOfClass` / `find_object` by hand
for the ref, then `object.call_function`). Only escape hatch; brittle and untyped.
**Fix:** an object-discovery RPC iterating UObjects by class
(`GetObjectsOfClass` / `FThreadSafeObjectIterator`) with a class filter,
world/outer scoping, and a sane cap — OR extend `list_objects` with a
`nonActors`/`uobjects` mode plus a class filter.

## History
- `#1-actor-only-blocks-uobject-discovery` `OPEN` reporter — Confirmed actor-only in current source: `list_objects` loops `TActorIterator<AActor>` at `EnvironmentHandler.cpp:1421` (valid params `[filter, world, limit, fields, namesOnly]`, no class filter; `filter` is name/class substring only), `find_by_class` loops `TActorIterator<AActor>` at `:1527`. No non-actor UObject instance-discovery RPC exists (`list_subsystems`/singletons/`search_classes`/`inspect_object` all checked — none enumerate arbitrary live non-actor instances by class). Session repro (goal: reach live `UAppPhysicsConfigBridge`, owned by the `PhysicsWebBrowser` UMG widget in a PIE drone-edit screen, to call its `SetPart` UFUNCTION via `object.call_function`): `list_objects {filter:"AppPhysicsConfigBridge", classNames:[...]}` → `[UNKNOWN_PARAMS] ... Valid parameters: [filter, world, limit, fields, namesOnly]` (no class filter); `list_objects {filter:"AppPhysicsConfigBridge", namesOnly:true, limit:10}` → `{objects:[], count:0, totalMatches:0, world:"auto", worldPath:".../UEDPIE_0_L_Core.L_Core"}` (PIE world resolved — `B-inspect-misses-pie-world` fix works for actors — but 0 results because the bridge is not an actor); `{filter:"Bridge"}` → also 0. No MCP route to the widget-owned UObject; had to abandon the direct-UFUNCTION-call approach. Invocation (`F-object-call-function`) is DONE; only discovery is missing. Deferred out of `F-wiki-runtime-uobject-inspection` (docs-only).
- `#2-add-find-objects-by-class` `IN-REVIEW` developer — GO. Implemented the non-actor discovery half as a new dedicated verb `system.inspect.find_objects_by_class` (matches the F-inspect-list-subsystems / F-inspect-game-singletons pattern of a fresh `system.inspect.*` handler rather than overloading the actor-only `list_objects`). It enumerates live UObject INSTANCES of a class via `GetObjectsOfClass` (the global object hash) — so non-actors (widget-owned helpers, components, plain UObjects) are now reachable — emitting `{name, path, class, outer}` rows where `path` is the reference `object.call_function` consumes, closing the discovery→invocation chain. Footgun controls per the fix note: CDOs/archetypes excluded unless `includeDefaults=true`, garbage/unreachable always skipped, `world` scoping (`any` default, or `editor`/`pie`/`auto` via `IsIn`/`GetWorld`), and a `limit=100` cap with `totalMatches`/`truncated`. `className` resolves through the shared `ResolveClassByName` (short name / `/Script/…` / content path), returning `CLASS_NOT_FOUND` on a miss. Files: `Plugins/PinWright/Source/PinWright/Private/Handlers/Environment/EnvironmentHandler.cpp` (new handler + `MakeInspectUObjectRow` + `UObjectHash.h` include), `Plugins/PinWright/Source/PinWright/Private/Tests/World/TestEnvironmentHandlers.cpp` (regression test). Test `PinWright.system.inspect.find_objects_by_class.FindsLiveNonActorUObject` builds an in-code non-actor UDataTable probe (transient package, GUID-unique name) and asserts (1) the new verb finds it by class+name with a usable path, (2) the actor-only `find_by_class` never returns it, (3) the CDO is excluded by default and only appears with `includeDefaults=true`.
