---
id: E-bare-object-name-not-resolved
title: "property.get / object.call_function resolve a bare object name only when it is an actor; a PIE world subsystem needs the full `/Game/Maps/UEDPIE_0_<Map>.<Map>:<Name>` path and the OBJECT_NOT_FOUND error gives no hint"
status: OPEN
severity: Low
category: ergonomic
tags: [property, object, call-function, object-resolution, pie, subsystem, object-not-found, discoverability]
encounters: 1
lastSeen: 2026-09-28T09:32:00Z
rice: [1, 2, 1, 2]
priority: 8
---

# Short names work for actors, not for other live objects

During PIE on the PDS map editor (`/Game/Maps/L_PDS_Stadium`), reading the gizmo world subsystem:

- `property.get {objectPath: "SKGMLEGizmoWorldSubsystem_0", ...}` -> `OBJECT_NOT_FOUND`
- `object.call_function {objectPath: "SKGMLEGizmoWorldSubsystem_0", ...}` -> `OBJECT_NOT_FOUND`
- the same calls with `/Game/Maps/UEDPIE_0_L_PDS_Stadium.L_PDS_Stadium:SKGMLEGizmoWorldSubsystem_0` worked.

Cause, plugin `8fcc0b2a`:

- `property.*` uses `ResolveObjectForProperty` (`Handlers/Utility/UtilityPropertyHandler.cpp:94-196`):
  `FindObject(nullptr, path)`, asset/`StaticLoadObject` candidates for `/`-prefixed paths, then
  `FindActorByName`. A bare name that is not an actor falls through all of them. The error is
  `Unable to find object at path SKGMLEGizmoWorldSubsystem_0.` (`:1184`).
- `object.call_function` uses `ResolveUObjectByPath` (`Utils/AssetUtils.cpp:2451-2493`): `StaticFindObject` +
  `StaticLoadObject` only, not even the actor fallback. Error: `Object not found: <path>`.

So the two verbs resolve short names differently, and neither says what path form it wants. The PIE path form
(`UEDPIE_<n>_` package prefix, `:` before the subobject) is not something a caller guesses.

**Workaround:** get the full `objectPath` from `system.inspect.list_subsystems {scope: "World"}` (or from an
actor/inspect listing) and pass that.

**Fix (proposed):** share one resolver between `property.*` and `object.*`. For a bare name (no `/`), after the
actor lookup, try `StaticFindObject` with the resolved query world (PIE-first) and its persistent level as outer,
and the world's subsystems by name; if still nothing, make `OBJECT_NOT_FOUND` say which forms were tried and
point at `system.inspect.list_subsystems` / `system.inspect.list_objects` for the full path.

## History
- `#1-bare-subsystem-name-not-found` `OPEN` reporter - UE 5.8, host `/sdb-disk/src/unreal/unreal-fpv` (Linux), plugin `8fcc0b2a`, PIE on L_PDS_Stadium in map-editor mode. Bare name returned OBJECT_NOT_FOUND until the UEDPIE path was built by hand. Cheap.
