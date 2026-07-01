---
id: E-environment-build-create-no-name-param
title: "environment.build.create_sky_sphere / create_fog_volume accept no name/label param and hardcode the actor label — naming a created environment actor costs an extra property.set ActorLabel call apiece"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [environment, sky, fog, name, label, actor-label, no-name-param, property-set, workaround]
---

# `environment.build.create_*` give the new actor a hardcoded label and expose no name slot — the caller pays a `property.set ActorLabel` per actor to honor a requested name

When a task asks for a named environment actor (`CanyonSky`, `CanyonValleyFog`),
two of the `environment.build.create_*` helpers cannot produce that name at
creation, and there is **no `actor.rename` / `actor.set_label` RPC** to fix it
afterward, so the only path is a follow-up `property.set` on the editor-only
`ActorLabel` property — one extra round-trip per created actor.

This is a different shape from the param-name-guessability family
(`E-volume-create-name-vs-volumename` OPEN, `E-geometry-create-name-vs-actorname`
OPEN, `E-effect-actor-name-slot-vs-actorname` OPEN). Those tickets are
"caller typed `name`, the verb wanted `volumeName`/`actorName`, got
`UNKNOWN_PARAMS`, corrected on retry" — the slot exists under another spelling.
Here the slot **does not exist at all**: there is no name/label parameter to
alias to, so no alias annotation can close it. The gap is a missing *capability*
(name-on-create), not a misspelled param.

## Source (handlers confirmed)

`Source/EditorAutomationRpcGateway/Private/Handlers/Environment/EnvironmentHandler.cpp`:

- `environment.build.create_sky_sphere` (:328) is declared `RPC_NO_PARAMS` —
  zero parameters, so no name/label can be supplied.
- `environment.build.create_fog_volume` (:486) declares only
  `RPC_PARAM_OPT("x"/"y"/"z")` (position) — no name/label slot. The handler
  spawns with a **hardcoded** label and echoes it back:
  ```cpp
  AActor* FogVolume = SpawnActorInActiveWorld<AActor>(
      FogClass, Location, FRotator::ZeroRotator, TEXT("FogVolume"));   // :514-515
  ...
  Resp->SetStringField(TEXT("actorName"), FogVolume->GetActorLabel());  // :519
  ```
  So `actorName` in the response is always the engine-assigned `FogVolume`
  (or `FogVolume_2`, …), never the name the caller wanted.

No `actor.rename` / `actor.set_label` RPC exists (grep for
`REGISTER_RPC_HANDLER("actor.(rename|set_label|relabel|label)"` is empty), so the
discovered workaround is the only one: `property.set` targeting the actor's
`ActorLabel`. That works, but the agent had to *guess* that the editor-only label
is reachable through the generic property route — it is not documented as the way
to rename a created actor.

## Repro (from the audited canyon-overlook greybox task; outcome `done`)

The task asked for actors named exactly `CanyonSky` and `CanyonValleyFog`. The
call log shows the forced two-step per actor:

1. `environment.build.create_sky_sphere {}` → success, actor created with the
   engine's default label (`SkySphere`).
2. `property.set` `SkySphere ActorLabel=CanyonSky` → relabel succeeds.
3. `environment.build.create_fog_volume {x:0,y:0,z:0}` → success, label
   `FogVolume`.
4. `property.set` `FogVolume ActorLabel=CanyonValleyFog` → relabel succeeds.

By contrast `environment.spawn_sky_atmosphere {name:"CanyonAtmosphere"}` and
`environment.spawn_volumetric_cloud {name:"CanyonClouds"}` (the modern verbs added
by DONE `F-sky-cloud-reflection-actors`) accept a `name` slot and named their
actors in a single call — so within the *same* namespace the new spawn verbs do
the right thing and the two legacy `build.create_*` helpers don't, which is what
makes the omission surprising.

Friction note (verbatim): *"create_sky_sphere and create_fog_volume accept NO
name/actorName param (wiki confirms), so the requested labels
CanyonSky/CanyonValleyFog were impossible at creation; there is no
actor.rename/set_label method, so I worked around it with property.set ActorLabel
(which did work, but only by guessing that the editor-only label is reachable that
way)."*

## What it should do

Cheapest, most consistent fix — add an optional `name` slot to the two legacy
helpers, mirroring the sibling `environment.spawn_*` verbs that already take
`name`:
- `create_sky_sphere`: change `RPC_NO_PARAMS` → `RPC_PARAMS(RPC_PARAM_OPT("name",
  "string", "Actor label"))`, and `SetActorLabel(name)` on the spawned sphere
  when supplied (defaulting to the current behavior when omitted).
- `create_fog_volume`: add the same `name` `RPC_PARAM_OPT` and pass it as the
  spawn label instead of the hardcoded `TEXT("FogVolume")` (:515).

This brings the whole `environment` create/spawn surface to one consistent
"pass `name` to label the actor" convention and removes the per-actor
`property.set` follow-up. (A general `actor.set_label` verb would also solve the
"rename after the fact" half and benefit every namespace whose create verb omits
a label, but that is a broader feature; the per-handler `name` slot is the
in-scope ergonomic fix here.)

## Docs angle (`docs/wiki-src/environment.md`)

Until a `name` slot lands, the discovery gap is real: `environment.md` is a
namespace prelude that does not teach that `create_sky_sphere` /
`create_fog_volume` take no label and that the way to name the result is a
follow-up `property.set ActorLabel`. A one-line note ("`create_sky_sphere` /
`create_fog_volume` do not accept a name; relabel the spawned actor with
`property.set` on `ActorLabel`, or prefer `environment.spawn_sky_atmosphere` /
`spawn_volumetric_cloud` which take `name` directly") closes the discovery half.
The wiki edit itself is the downstream process, not this ticket.

## Not a duplicate of

- `B-export-snapshot-empty-stub` (judge-filed this task) — that is the
  `export_snapshot` empty-body OUTCOME bug; this is the PROCESS naming friction
  on the create verbs.
- `B-create-sky-sphere-stale-path` (IN-REVIEW) — that fixed the dead
  `LoadClass` path so `create_sky_sphere` spawns at all; it did not add a name
  slot. Orthogonal.
- `F-sky-cloud-reflection-actors` (DONE) — added the modern `spawn_*` verbs
  (which DO take `name`); it never touched the legacy `build.create_*` helpers'
  no-name signatures.
- `E-volume-create-name-vs-volumename` / `E-geometry-create-name-vs-actorname` /
  `E-effect-actor-name-slot-vs-actorname` — the param-name-spelling family
  (slot exists under a different name, `UNKNOWN_PARAMS`-then-retry). Here the
  slot is absent entirely, so there is nothing to alias.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the canyon-overlook
  greybox task (seed `environment.build.create_procedural_terrain`; outcome
  `done`; judge filed `B-export-snapshot-empty-stub` for the snapshot bug).
  Distinct PROCESS angle: the requested actor labels `CanyonSky` /
  `CanyonValleyFog` were impossible at creation because
  `environment.build.create_sky_sphere` (`EnvironmentHandler.cpp:328`,
  `RPC_NO_PARAMS`) and `create_fog_volume` (:486, only `x/y/z`) expose no
  name/label slot and `create_fog_volume` hardcodes the spawn label
  `TEXT("FogVolume")` (:515), and no `actor.rename`/`actor.set_label` RPC exists.
  The agent paid one extra `property.set ActorLabel` per actor (2 of 15 calls)
  to honor the names — a workaround it had to guess. Not a param-spelling miss
  (no `UNKNOWN_PARAMS`): the slot is absent, so the alias-family tickets
  (`E-volume-create-name-vs-volumename`, `E-geometry-create-name-vs-actorname`,
  `E-effect-actor-name-slot-vs-actorname`) don't cover it. Fix: add an optional
  `name` slot to both legacy helpers mirroring the sibling `environment.spawn_*`
  verbs (which already take `name`); plus a `docs/wiki-src/environment.md` note.
- `#2-fix` `IN-REVIEW` developer — Added the optional `name` slot to both legacy
  create verbs, mirroring the sibling `environment.spawn_*` convention.
  `environment.build.create_sky_sphere` changed from `RPC_NO_PARAMS` to
  `RPC_PARAMS(RPC_PARAM_OPT("name", ...))`; `environment.build.create_fog_volume`
  gained a `name` `RPC_PARAM_OPT` alongside x/y/z. Both read `Ctx.GetString("name")`
  and pass it as the 4th (label) arg to the existing
  `SpawnActorInActiveWorld(..., Label)` helper, which already calls
  `SetActorLabel` only when the label is non-empty; when `name` is omitted each
  falls back to its historical literal (`"SkySphere"` / `"FogVolume"`) so the
  default World Outliner name is unchanged. The echoed `actorName`
  (`GetActorLabel()`) now reflects the requested name, removing the per-actor
  follow-up `property.set ActorLabel`. File:
  `Source/EditorAutomationRpcGateway/Private/Handlers/Environment/EnvironmentHandler.cpp`.
  Tests (in `Private/Tests/World/TestEnvironmentHandlers.cpp`):
  `create_sky_sphere.HonorsNameSlot` and `create_fog_volume.HonorsNameSlot` —
  each asserts the `name` param is registered, that a unique GUID-suffixed name
  is echoed back verbatim as `actorName` (when the spawn succeeds), and that
  omitting `name` keeps the default label prefix; both fail if the `name` slot is
  reverted. The `docs/wiki-src/environment.md` note is left as the downstream wiki
  process, not part of this code change.
