---
id: E-volume-create-name-vs-volumename
title: "volume.create_* and every volume verb name the actor slot 'volumeName' with no 'name' alias — agents default to 'name' and eat an UNKNOWN_PARAMS round-trip per create"
status: OPEN
severity: Low
category: ergonomic
tags: [volume, param-alias, name, volumename, create_trigger_box, create_physics_volume, drift, docs]
encounters: 1
lastSeen: 2026-06-17T03:38:54Z
---

# The whole `volume.*` namespace uses `volumeName`; agents reach for the generic `name` and pay one UNKNOWN_PARAMS per create verb

Same param-name guessability family as `E-geometry-create-name-vs-actorname`
(OPEN), `E-static-mesh-describe-param-name-drift` (OPEN), and the DONE
path/param-alias precedents (`E-blueprint-param-name-path-vs-assetpath`,
`E-widget-asset-path-alias-drift`, `E-material-editor-param-name-drift`) — but
here the slot is the **actor-name** slot and the namespace is `volume.*`, which
none of those sweeps touched.

The whole `volume.*` namespace declares its actor-name slot as **`volumeName`**
(`RPC_PARAM_OPT("volumeName", ...)` on the create verbs, `RPC_PARAM_REQ` on the
operate verbs), with **no `name` alias** anywhere:
- `VolumeHandler.cpp:411-413` `volume.create_trigger_box` — `RPC_PARAM_OPT("volumeName", "string", "Name for the volume")`, read via `Ctx.GetString(TEXT("volumeName"), TEXT("TriggerVolume"))` (:423).
- `VolumeHandler.cpp:788-790` `volume.create_physics_volume` — identical `volumeName` slot (:803).
- Every other create verb (blocking/audio/postprocess/kill-z/nav/pain/etc., including the macro-generated block at :616) and every operate verb (`set_volume_extent` :1282, `set_volume_properties` :1328, `set_volume_bounds` :1394, `remove_volume` :1599) all declare `volumeName` and nothing else.

Unlike `E-geometry-create-name-vs-actorname`, the `volume.*` namespace is
**internally consistent** — create and operate verbs agree on `volumeName`, so
the "spelling flips mid-build" argument from the geometry ticket does NOT apply
here, and there is no intra-namespace drift to fix. The friction is purely
**cross-namespace default-guess collision**: the generic actor-name slot is
`name` in the verbs agents reach for first (`actor.spawn`, the entire
`geometry.create_*` family per `E-geometry-create-name-vs-actorname`,
`material.create_material` per `E-material-editor-param-name-drift #2`), so an
agent priming a "create a named volume" intent on that muscle memory types
`name` and the uniformly-`volumeName` volume namespace rejects it on the first
call. CLAUDE.md's "camelCase and snake_case aliases" rule does not cover this —
`name` and `volumeName` are distinct names, not casing variants.

## Repro (verbatim, from the audited arena-blockout task)

Two create verbs, each a one-shot misuse-then-correct:

1. `volume.create_trigger_box {name:"ArenaEntryTrigger", ...}`
   → `[UNKNOWN_PARAMS] Unknown parameter(s) for 'volume.create_trigger_box': [name]. Valid parameters: [volumeName, location, rotation, boxExtent, extent].`
   Retry with `{volumeName:"ArenaEntryTrigger", ...}` → succeeds.
2. `volume.create_physics_volume {name:"ArenaWaterPool", ...}`
   → `[UNKNOWN_PARAMS] Unknown parameter(s) for 'volume.create_physics_volume': [name]. Valid parameters: [volumeName, location, rotation, extent, bWaterVolume, fluidFriction, terminalVelocity, priority].`
   Retry with `{volumeName:"ArenaWaterPool", ...}` → succeeds.

Both errors are accurate and self-correcting — `UNKNOWN_PARAMS` lists the valid
param set, so the agent fixed each on the very next call with zero blocked
progress. All later same-actor calls (`set_volume_bounds`,
`set_volume_properties`, `set_volume_extent`) used `volumeName` and succeeded
first try. Friction note (verbatim): *"first create_trigger_box/create_physics_volume
calls failed because I guessed param 'name' instead of 'volumeName'; the
UNKNOWN_PARAMS error listed valid params so I corrected on the next try."* Pure
first-call guessability overhead — two wasted round-trips for the same `name`
misguess, once per create verb.

## What it should do

Reuse the dispatcher `FParamSpec` alias machinery from
`E-blueprint-param-name-path-vs-assetpath #4` (the change that made alias-only
params validate at the wire level). Standardize the alias *set*, not the
canonical name, so existing `volumeName` callers keep working:
- Annotate the `volumeName` slot across the `volume.*` namespace with a `name`
  alias (and accept `name` in the `Ctx.GetString(TEXT("volumeName"), ...)`
  reads). Because the whole namespace is uniform, one shared alias annotation
  covers all create + operate verbs at once.
- Note: `E-material-editor-param-name-drift #2` deliberately left
  `create_material`'s `name`/`path` UNALIASED to keep create-slot names
  explicit. The volume case is a cleaner alias candidate than that one — every
  volume verb already agrees on `volumeName`, so adding `name` as an accepted
  alias removes the cross-namespace default-guess trap without introducing any
  new ambiguity inside the namespace.

Docs angle (`docs/wiki-src/volume.md`): the overlay is a 4-line prelude with no
per-method sections — there is no `### volume.create_trigger_box` /
`### volume.create_physics_volume` section to teach that the actor-name slot is
`volumeName` (not the generic `name`). Until aliases land, a short note on the
namespace overlay (or a per-create-verb H3) stating the slot is `volumeName`
across the whole namespace closes the discovery gap. This is the same overlay
that `E-volume-type-filter-discovery` already flags for a missing
`### volume.get_volumes_info` section — both wants live on `docs/wiki-src/volume.md`.

This is distinct from the judge-filed `B-blocking-volume-no-brush-geometry`
(that ticket is the brush/collision tool bug on blocking volumes; this is purely
the create-verb param-name guessability) and from `E-volume-type-filter-discovery`
(that is the `volumeType` *filter value* vocabulary on the readback path; here
the *param name* `volumeName` is the friction, and the values were never in
question).

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the arena-blockout task
  (seed `volume.set_volume_bounds`; outcome `tool_bug`, judge filed
  `B-blocking-volume-no-brush-geometry`). Distinct PROCESS angle: two
  misuse-then-correct round-trips, `volume.create_trigger_box {name:...}` and
  `volume.create_physics_volume {name:...}`, each rejected with an accurate
  `[UNKNOWN_PARAMS]` listing `volumeName` and corrected on retry; all subsequent
  `volumeName` calls succeeded first try. Source: `VolumeHandler.cpp:413`/`:790`
  (create slots), `:1282`/`:1328`/`:1394`/`:1599` (operate slots) — the entire
  namespace declares `volumeName` with no `name` alias, so the namespace is
  internally consistent (unlike `E-geometry-create-name-vs-actorname`'s
  create=`name`/operate=`actorName` drift); the friction is the cross-namespace
  default-guess `name` (taught by `actor.spawn`, `geometry.create_*`,
  `material.create_material`). New namespace, not covered by
  `E-geometry-create-name-vs-actorname` (OPEN),
  `E-static-mesh-describe-param-name-drift` (OPEN), or the DONE
  path/param-alias precedents. Fix: dispatcher `FParamSpec` alias from
  `E-blueprint-param-name-path-vs-assetpath #4`, aliasing the namespace
  `volumeName` slot to accept `name`; plus a `docs/wiki-src/volume.md` note (same
  overlay `E-volume-type-filter-discovery` already targets).
