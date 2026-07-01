---
id: E-actor-verbs-reject-actorpath-slot
title: "actor.* read verbs require 'actorName' with NO aliases — they reject 'actorPath' (the key actor.spawn/actor.duplicate emit and volume.*/world.*/gas.* accept) AND the documented 'objectPath'"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [actor, param-alias, actorname, actorpath, objectpath, get_components, spawn, drift]
---

# `actor.*` consumer verbs reject `actorPath`, the slot name their own producers emit and other namespaces accept

Same param-name-drift class as `E-blueprint-param-name-path-vs-assetpath` (DONE,
the canonical dispatcher-alias fix), `E-asset-path-vs-assetpath-list-drift`
(OPEN), and `E-geometry-create-name-vs-actorname` (OPEN) — but here the drift is
on the **actor-identity** slot, and the trap is a producer→consumer one **inside
the `actor.*` namespace**: a verb hands back an actor reference under the key
`actorPath`, and the next verb that consumes that reference refuses `actorPath`.

The actor-identity slot splits three ways across the surface, with `actorPath`
accepted everywhere *except* the `actor.*` reader verbs that an agent reaches for
right after a spawn:

- `actor.spawn` / `actor.spawn_instance` / `actor.duplicate` **return** the new
  actor's identity in a field named **`actorPath`** — `SpawnHandler.cpp:214,322`
  (`Data->SetStringField(TEXT("actorPath"), Spawned->GetPathName())`) and
  `LifecycleHandler.cpp:151`.
- But the `actor.*` reader/operator verbs declare **`actorName`** via
  `RPC_PARAM_REQ`, which sets an **empty alias list** (`ParamSpec.h:31-32`). The
  wire-level validator `RpcDispatcher::ValidateHandlerParams` (`RpcDispatcher.cpp:70`)
  satisfies a required param only via its declared name or `.Aliases`/`.TypedAliases`
  (`PayloadHasParamOrAlias`, `:22-50`) — and runs **before** the handler body. So
  any payload that omits the literal `actorName` key fails with
  `MISSING_REQUIRED_PARAM 'actorName'`, **including** an `objectPath`-only payload:
  the handler *bodies* read `Ctx.GetStringFirstOf({actorName, objectPath})`
  (`ComponentHandler.cpp:331`, `DescribeHandler.cpp:21`), but that `objectPath`
  branch is **dead code** — validation rejects the request first, so `objectPath`
  is NOT actually accepted today either. `actor.get_components` —
  `ComponentHandler.cpp:324` `RPC_PARAM_REQ("actorName", ...)` + `:331` read;
  identically `actor.describe` (`DescribeHandler.cpp:12,21`), and the rest of
  `actor.*` (`QueryHandler.cpp:82,312`, `ActorPropertyHandler.cpp`,
  `ComponentHandler.cpp:37,171,417,466`).
- Meanwhile **`actorPath` IS a declared input slot in other namespaces** that
  identify a target actor: `world.unload_actor` (`WorldPartitionHandler.cpp:162`),
  six `volume.*` verbs (`VolumeHandler.cpp:1664,1706,1748,1794,1848,1909`),
  `sequencer.*` rail/crane (`SequencerHandler.cpp:460,471`), `gas.*`
  (`GASHandler.cpp:2995`). So an agent who learned `actorPath` from any of those
  (or from the spawn response) reasonably reuses it against `actor.*` and gets a
  hard error.

CLAUDE.md's "camelCase and snake_case aliases" rule does not cover this —
`actorName`, `objectPath`, and `actorPath` are three distinct names, not casing
variants. The verb description *advertises* the `objectPath` alias, and the body
*reads* it, but the spec declares no aliases at all — so neither `objectPath` nor
`actorPath` actually validates: the description over-promises. And the error
message names only the canonical (`Missing required parameter 'actorName'`)
without listing accepted aliases, so the caller cannot self-correct from the
error.

## Repro (verbatim, from the audited task)

A "polish Lighting_Realtime" task spawned a point fill light, then went to read
its components:

1. `level.spawn_light {lightType:"Point", location:{300,0,250}}` → spawns
   `PointLight_2` (the response carries the actor reference in an `actorPath`-style
   field, the engine-wide spelling for a spawned actor's path).
2. `actor.get_components {actorPath:"PointLight_2"}` (first try, reusing the
   spawn-side key)
   → `[MISSING_REQUIRED_PARAM] Missing required parameter 'actorName' (type: string)`
3. Retry with `{actorName:"PointLight_2"}` → succeeds (component list returned,
   `LightComponent0` found).

Friction note (verbatim): *"Minor: actor.get_components rejected
objectPath/actorPath and required the key 'actorName' (one wasted call before I
read its wiki page)."* One wasted `is_error` call, corrected on retry, zero
blocked progress — pure guessability/round-trip overhead, the exact shape of the
precedent param-alias-drift tickets. (The friction phrasing names both
`objectPath` and `actorPath` — and the source confirms BOTH are rejected at the
wire level today: the spec carries no aliases, so `ValidateHandlerParams` fails
an `objectPath`-only or `actorPath`-only payload identically, even though the body
*would* read `objectPath` if it ever reached it. The reported failing key was
`actorPath`, per the call-log `args_summary` "actorPath=PointLight_2 (first try,
wrong key)"; the fix repairs both keys at once by attaching the alias set.)

## What it should do

Reuse the dispatcher `FParamSpec` alias machinery landed in
`E-blueprint-param-name-path-vs-assetpath #4` (which made alias-only required
params validate at the wire level in `RpcDispatcher::ValidateHandlerParams`).
Standardize the alias *set* on the actor-identity slot, not the canonical name,
so existing callers keep working:

- Attach the `objectPath` + `actorPath` (+ snake_case `actor_name`) alias set to
  the `actor.*` `actorName` slot — at minimum on the verbs an agent naturally
  chains after a spawn (`actor.get_components`, `actor.describe`) — so the
  dispatcher's `PayloadHasParamOrAlias` / known-params set honor them, and read the
  value via `GetStringFirstOf` over the same key list so the alias resolves
  end-to-end. This also closes the `objectPath` dead-code gap (the body read it but
  validation rejected it first). The reads already tolerate a path *or* a label as
  the value (`FindActorByName` matches by `GetPathName` too), so no resolution
  change is needed — only the key was rejected.
- Make the missing-param error for these slots enumerate the accepted aliases
  (`actorName` / `objectPath` / `actorPath`) instead of naming only the canonical,
  so a wrong-key caller self-corrects from the error without a wiki round-trip.

**Fix:** Added a shared `ActorNameParamUtils` helper (mirroring
`AssetPathParamUtils`) defining the actor-identity key set
`{actorName, objectPath, actorPath, actor_name}` and applied it to
`actor.get_components` (`ComponentHandler.cpp`) and `actor.describe`
(`DescribeHandler.cpp`) — spec via `MakeAliasParamSpec`, body via
`GetStringFirstOf`. Extended `RpcDispatcher::ValidateHandlerParams` to enumerate a
required param's accepted aliases in the `MISSING_REQUIRED_PARAM` message.

The wiki pages are individually correct (each verb documents its own slot and the
`objectPath` alias), so this is a guessability/alias gap, not a docs gap — but if
only a docs steer is wanted, `docs/wiki-src/actor.md` could note that the
`actorPath` returned by spawn/duplicate must be re-spelled `actorName`/`objectPath`
when fed back into `actor.*` consumers.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle (process) audit of the
  `level.set_lighting` "polish Lighting_Realtime" task (outcome tool_bug; the
  async-completion tool bug was filed separately by the judge as
  `B-level-load-no-completion-signal`, a distinct OUTCOME angle). The PROCESS
  friction here: after `level.spawn_light` produced `PointLight_2`, the first
  `actor.get_components {actorPath:"PointLight_2"}` hard-failed
  `[MISSING_REQUIRED_PARAM] Missing required parameter 'actorName'`, corrected on
  retry to `{actorName:...}` — one wasted call, zero blocked progress. Root cause
  confirmed in source: `actor.get_components` accepts only `{actorName, objectPath}`
  (`ComponentHandler.cpp:323,330`), yet `actor.spawn`/`actor.duplicate` emit the
  spawned actor under `actorPath` (`SpawnHandler.cpp:214,322`,
  `LifecycleHandler.cpp:142`), and `actorPath` is a declared input slot in
  `volume.*`/`world.*`/`sequencer.*`/`gas.*` (`VolumeHandler.cpp:1664…`,
  `WorldPartitionHandler.cpp:162`, `SequencerHandler.cpp:460`, `GASHandler.cpp:2527`)
  — so `actorPath` is engine-wide-valid everywhere except the `actor.*` consumers.
  Sibling of the DONE/OPEN path/param-alias-drift tickets, here on the actor-identity
  slot's `actorPath` axis. Dedup (ripgrep over OPEN+closed; qmd unavailable):
  distinct from `E-spawn-returns-actor-not-component-path` (OPEN — spawn returning
  `actorPath` instead of the *componentPath* an agent wants, a value/return-shape
  gap, not the input-key rejection) and from `E-property-route-no-component-path-discovery`
  (OPEN — component-name discovery, not the actor-identity key). Fix: dispatcher
  `FParamSpec` alias from `E-blueprint-param-name-path-vs-assetpath #4`, adding
  `actorPath` to the `actor.*` `actorName`/`objectPath` alias set, plus an
  alias-listing missing-param error.
- `#2-reword-and-fix` `IN-REVIEW` developer — REWORD then implement. Reworded:
  the original body asserted `objectPath` "IS accepted at ComponentHandler.cpp:330";
  verified in source that this is FALSE at the wire level — `RPC_PARAM_REQ` sets an
  empty `.Aliases` (`ParamSpec.h:31-32`), and `ValidateHandlerParams`
  (`RpcDispatcher.cpp:70`) runs `PayloadHasParamOrAlias` (which only checks the
  declared name + `.Aliases`/`.TypedAliases`) BEFORE the handler body, so an
  `objectPath`-only payload failed `MISSING_REQUIRED_PARAM 'actorName'` exactly like
  an `actorPath`-only one; the body's `GetStringFirstOf({actorName, objectPath})`
  `objectPath` branch was dead code. Title/body/repro corrected to state both keys
  were rejected; the spec-level fix repairs both at once. Fix (as GO): added shared
  helper `Handlers/Actor/ActorNameParamUtils.h` (mirrors `AssetPathParamUtils`)
  defining the actor-identity key set `{actorName, objectPath, actorPath, actor_name}`
  via `ParamAliasUtils::MakeAliasParamSpec`; applied it to `actor.get_components`
  (`Handlers/Actor/ComponentHandler.cpp`) and `actor.describe`
  (`Handlers/Actor/DescribeHandler.cpp`) — spec + `GetStringFirstOf` body read.
  Extended `RpcDispatcher::ValidateHandlerParams` (`Dispatch/RpcDispatcher.cpp`) so
  the `MISSING_REQUIRED_PARAM` message enumerates a required param's accepted
  aliases. Regression test `Tests/World/TestActorNameParamAlias.cpp` (two cases,
  modeled on `TestAssetPathParamAlias`): static — both reader verbs register the
  `actorName` slot with `actorPath`+`objectPath` aliases; end-to-end — dispatching
  `{actorPath:<missing actor>}` through the real dispatcher is NOT rejected with
  `MISSING_REQUIRED_PARAM`/`UNKNOWN_PARAMS` (reaches the body → `ACTOR_NOT_FOUND`).
  Reverting the alias fails both cases.
