---
id: E-create-procedural-terrain-no-label-set
title: "environment.build.create_procedural_terrain applies the caller's actorName to the object name only and never calls SetActorLabel, so the World Outliner label stays the generic class default 'Actor' and the success response echoes actorName='Actor' — the name the caller passed is dropped from the label and misreported in the response"
status: OPEN
severity: Medium
category: ergonomic
tags: [environment, terrain, create_procedural_terrain, actorname, actor-label, label-not-set, result-misreport, create-verb-no-label-set]
encounters: 1
lastSeen: 2026-07-11T05:54:38.6647086+03:00
---

# `create_procedural_terrain` takes `actorName`, uses it for the internal object name but never `SetActorLabel`s it, so the outliner label is the generic `"Actor"` and the response's `actorName` echoes `"Actor"` (not the requested name)

`environment.build.create_procedural_terrain` accepts an `actorName` parameter
and applies it to the spawned actor's **internal object name** via
`SpawnParams.Name`, but it **never calls `SetActorLabel`**. As a result the
actor's editor **display label** falls back to the class-derived default
`"Actor"`. Two concrete consequences:

1. **World Outliner label is generic `"Actor"`.** Unlike the sibling create
   verbs `create_sky_sphere` / `create_fog_volume` — which route through
   `SpawnActorInActiveWorld(..., Label)` and DO `SetActorLabel(name)` — the
   terrain actor is unlabelled, so a human scanning the outliner for the name
   they requested (e.g. `MarshTerrain`) does not see it; it shows as `Actor`.

2. **The success response misreports `actorName`.** The handler first writes
   `actorName = TerrainActor->GetName()` (the object name), but then calls
   `AddActorVerification`, whose shared helper **overwrites** `actorName` with
   `Actor->GetActorLabel()`. Since no label was ever set, `GetActorLabel()`
   returns the generic `"Actor"`. So a caller who passes
   `actorName:"MarshTerrain"` and gets a `success` back reads
   `actorName:"Actor"` in the response — a value they never supplied. The real
   object name survives only buried in the `actorPath` leaf.

The actor IS created correctly and IS still discoverable by object name
(`actor.list` / `actor.find_by_name` report `name:"MarshTerrain"` alongside
`label:"Actor"`), so this is not a functional failure — it is a
naming/result-misreport ergonomic defect: the name the caller asked for is
silently dropped from the label and echoed back wrong.

## Root cause (source-confirmed, live plugin tree)

`Plugins/PinWright/Source/PinWright/Private/Handlers/Environment/EnvironmentHandler.cpp`,
`environment.build.create_procedural_terrain`:

- The caller's name goes to the OBJECT name only, and the actor is spawned raw
  with no label call anywhere in the handler (lines 1084-1086, 1110):

  ```cpp
  FActorSpawnParameters SpawnParams;
  SpawnParams.Name = FName(*ActorName);
  SpawnParams.NameMode = FActorSpawnParameters::ESpawnActorNameMode::Requested;
  ...
  AActor* TerrainActor = World->SpawnActor<AActor>(AActor::StaticClass(), Location, Rotation, SpawnParams);
  ```

  Grepping the whole handler body confirms there is no `SetActorLabel` call.

- The response `actorName` write (line 1210) is dead — it is clobbered:

  ```cpp
  Resp->SetStringField(TEXT("actorName"), TerrainActor->GetName());   // :1210 (overwritten below)
  ...
  AddActorVerification(Resp, TerrainActor);                            // :1231
  ```

- `AddActorVerification` (`Utils/AssetUtils.cpp:1056`) sets `actorName` to the
  label:

  ```cpp
  Response->SetStringField(TEXT("actorName"), Actor->GetActorLabel());
  ```

- By contrast the siblings label correctly. `create_sky_sphere` (:305-306) and
  `create_fog_volume` (:485-486) both call
  `SpawnActorInActiveWorld<AActor>(..., Label)`, and that helper
  (`Utils/AssetUtils.h:422-425`) does the label set the terrain handler omits:

  ```cpp
  if (Spawned && !OptionalLabel.IsEmpty())
  {
      Spawned->SetActorLabel(OptionalLabel);
  }
  ```

So within `environment.build`, only `create_procedural_terrain` skips
`SetActorLabel` (the sky/fog verbs use a different, correct code path) — this is
a single-method defect, not a family.

## Replay-confirmed repro (live editor, `mcp__pinwright__call`)

1. `environment.build.create_procedural_terrain {actorName:"ReplayMarshTerrain", sizeX:20, sizeY:20, subdivisions:8, heightScale:100, location:{x:0,y:0,z:0}}`
   → success:
   ```json
   {"actorName":"Actor",
    "actorPath":"/Game/Maps/ExampleProjectWelcome.ExampleProjectWelcome:PersistentLevel.ReplayMarshTerrain",
    "vertices":81,"triangles":128,"sizeX":20,"sizeY":20,"subdivisions":8,
    "material_applied":false,"mapPath":"/Game/Maps/ExampleProjectWelcome",
    "actorGuid":"8028DE994E8E9BBB90C54F83F887D4B5","existsAfter":true,"actorClass":"Actor"}
   ```
   Passed `actorName:"ReplayMarshTerrain"`, got back `actorName:"Actor"`. The
   requested name survives only as the `actorPath` leaf `...ReplayMarshTerrain`.

2. `actor.list {filter:"ReplayMarsh"}` →
   `{"actors":[{"label":"Actor","name":"ReplayMarshTerrain","path":".../PersistentLevel.ReplayMarshTerrain","class":"/Script/Engine.Actor"}], ...}`

3. `actor.find_by_name {name:"ReplayMarshTerrain"}` →
   `{"count":1,"actors":[{"label":"Actor","name":"ReplayMarshTerrain",...}], ...}`

Readback confirms the split: `label:"Actor"` (generic default, never set) vs
`name:"ReplayMarshTerrain"` (the requested name, applied to the object name
only).

## What it should do

Mirror the sibling create verbs: after spawning the terrain actor, call
`TerrainActor->SetActorLabel(ActorName)` (the caller already supplied the name).
Then the World Outliner shows the requested name and the `AddActorVerification`
echo (`actorName = GetActorLabel()`) reports the requested name instead of
`"Actor"` — bringing `create_procedural_terrain` in line with
`create_sky_sphere` / `create_fog_volume`. (The dead `actorName = GetName()`
write at :1210 can also be dropped, since the helper owns the field.)

## Not a duplicate of

- `E-create-procedural-terrain-no-material-echo` (IN-REVIEW) — SAME method, but a
  DIFFERENT root cause: that ticket's load-bearing fix is the missing `material`
  echo. Its `#2` rework touched this label/`actorName` question tangentially and
  drew the WRONG conclusion — it stated "the response `actorName` is the actor
  label, not the internal object name" and dismissed the "bare Actor" concern as
  already-fine. This replay proves the label is the generic `"Actor"` (never
  set), so the echo IS misleading; the `#2` rework mischaracterized the label as
  meaningful. The material echo and the missing `SetActorLabel` are two separate
  defects on the same handler.
- `E-environment-build-create-no-name-param` (IN-REVIEW) — the SIBLING verbs
  `create_sky_sphere` / `create_fog_volume` exposing NO name slot at all. Here
  the slot EXISTS (`actorName`) and is honored for the object name; the defect is
  that it is not also applied to the label. Opposite shape (slot present but
  under-applied vs slot absent), different methods.
- `E-spline-create-actorname-echoes-deduped` (IN-REVIEW) — there the label IS set
  correctly to the requested name; the misreport arises only on a name COLLISION
  (object deduped to `<name>_0`, no signal). Here there is no collision — the
  object name is exactly the requested name; the label is simply never set.
- `E-networking-actorname-internal-name-only` (IN-REVIEW) — an input-side
  RESOLVER that rejects the display label. This is a create-side failure to SET
  the label; different method, different mechanism.

## History
- `#1-initial-repro` `OPEN` reporter — Realism-mode marsh-scene blockout task
  (touched `environment.build.create_procedural_terrain` +
  `create_sky_sphere`/`create_fog_volume`/`set_time_of_day`/`actor.list`/`delete`).
  The attempt agent flagged the labeling inconsistency as a hunch ("set the
  actor's internal name to MarshTerrain but left its editor label as generic
  'Actor', unlike sky/fog creators which set the label"). Replay-confirmed live
  (UE 5.7): `create_procedural_terrain {actorName:"ReplayMarshTerrain"}` returns
  `actorName:"Actor"` and `actor.list`/`find_by_name` show `label:"Actor"` /
  `name:"ReplayMarshTerrain"`. Root cause (source): the handler sets
  `SpawnParams.Name = FName(*ActorName)` (EnvironmentHandler.cpp:1085) but never
  calls `SetActorLabel`, and `AddActorVerification` (AssetUtils.cpp:1056)
  overwrites the response `actorName` with `GetActorLabel()` = the class-derived
  `"Actor"`; siblings avoid this by routing through
  `SpawnActorInActiveWorld(..., Label)` which `SetActorLabel`s (AssetUtils.h:424).
  Fix: add `TerrainActor->SetActorLabel(ActorName)` after spawn. severity
  rationale: impact=result-misreport (caller trusts a wrong `actorName`, but the
  true name is recoverable via `actorPath` / readback `name`, so a soft misreport
  with a workaround, not an unrecoverable silent lie) × reach=every-session? no —
  a normal (not rare) environment-build create path -> Medium.
