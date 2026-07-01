---
id: E-property-objectpath-no-assetpath-alias
title: "property.get/set/list require 'objectPath' with no 'assetPath' alias — a caller arriving from a blueprintPath/assetPath flow guesses 'assetPath' and hard-fails MISSING_REQUIRED_PARAM, costing a wiki-nav + retry"
status: OPEN
severity: Low
category: ergonomic
tags: [property, param-alias, objectpath, assetpath, drift, docs]
encounters: 1
lastSeen: 2026-06-25T07:14:35Z
---

# `property.*` reflection verbs require `objectPath` and reject the `assetPath` spelling every adjacent asset/blueprint verb teaches

Same param-name-drift class as the DONE `E-blueprint-param-name-path-vs-assetpath`
(the canonical dispatcher-`FParamSpec`-alias fix), the open
`E-asset-path-vs-assetpath-list-drift` (`path`↔`assetPath` across `asset.*`),
and `E-niagara-set-property-emitter-alias` — but here it surfaces on a **third
name pair**, `objectPath` ↔ `assetPath`, inside the `property.*` reflection
namespace.

`property.get`, `property.set`, and `property.list` (and the whole
`container.*` family in the same TU) all declare a bare
`RPC_PARAM_REQ("objectPath", "string", "Object path or actor name")` and read
body-side via `Ctx.GetString(TEXT("objectPath"))`, with **no `assetPath`
alias** (UtilityPropertyHandler.cpp: `property.set` :825/:833,
`property.get` :1221/:1229, `property.list` :1527/:1541, plus every
`container.*` verb at :1800/:1876/:1919/:1948/…). So a caller who just spent a
whole task in the `blueprintPath`/`assetPath` idiom (every `networking.*`,
`blueprint.*`, and `asset.*` verb names its target `blueprintPath` or
`assetPath`) and then drops to `property.get` to read one CDO field naturally
reuses `assetPath` and gets a hard
`[MISSING_REQUIRED_PARAM] Missing required parameter 'objectPath' (type: string)`.

The `objectPath` slot description — "Object path **or actor name**" — does not
mention an asset/blueprint path, so the slot name itself does not advertise
that a `/Game/...` asset path is the accepted value here; the agent has to read
the `property.md` overlay (a wiki-nav) to discover the canonical spelling, then
retry. CLAUDE.md's "camelCase and snake_case aliases" rule does not cover this:
`objectPath` and `assetPath` are distinct names, not casing variants, and the
slot carries neither as an alias, so guessing wrong is a hard error, not a
silent accept.

## Repro (verbatim, from the audited task)

Struggle-audit of the `networking.set_net_role` multiplayer-pickup build
(`/Game/Multiplayer/BP_HealthPickup`: create Actor BP → add replicated int
`HealthAmount` + bool `bIsConsumed` → `set_property_replicated` ×2 →
`set_net_role(ROLE_Authority)` → `configure_replicated_movement(true)` →
`set_net_dormancy(DORM_Initial)` → `create_rpc_function(ServerConsumePickup,
Server, reliable)` → compile → save; outcome `gap`, the readback gap filed by
the judge as `F-networking-info-no-rpc-detail`). After `get_networking_info`
came up short on the replicate-movement flag (the `F-` gap), the agent fell
back to a raw `property.get` to confirm `bReplicateMovement` and tripped the
alias drift:

1. `property.get {assetPath:"/Game/Multiplayer/BP_HealthPickup", propertyName:"bReplicateMovement"}`
   → `[MISSING_REQUIRED_PARAM] Missing required parameter 'objectPath' (type: string)`
2. (wiki-nav to `property.get.md` to find the right slot)
3. `property.get {objectPath:"/Game/Multiplayer/BP_HealthPickup", propertyName:"bReplicateMovement"}`
   → `{"propertyName":"bReplicateMovement","value":true,...}`

One `assetPath`→`objectPath` round-trip plus one extra wiki-nav, recovered
immediately, zero blocked progress. Friction note (verbatim): "property.get
rejects the assetPath alias (needs objectPath), costing one extra wiki-nav and
retry." Pure guessability / round-trip overhead, the exact shape of the
precedent param-alias-drift tickets.

## What it should do

Reuse the dispatcher `FParamSpec` alias machinery landed in
`E-blueprint-param-name-path-vs-assetpath #4` (and reused by the DONE material +
widget + the IN-REVIEW asset-readback fixes). Annotate the `property.get` /
`property.set` / `property.list` (and the `container.*`) `objectPath` spec with
an `assetPath` alias and read body-side via `GetStringFirstOf({"objectPath",
"assetPath"})`, so `{assetPath:...}` validates at the wire level and resolves
the same way. Do not make the canonical param optional per-handler (the
anti-pattern called out in `E-blueprint-param-name-path-vs-assetpath #3` — it
leaves the alias undiscoverable).

**Docs angle (`docs/wiki-src/property.md`):** even with the alias, the overlay
should state up front that the target slot is `objectPath` and that it accepts a
`/Game/...` asset/blueprint path (the CDO), an in-level actor name, or a full
object path — so a caller arriving from an `assetPath` flow doesn't have to fail
once to learn the slot name. (Reading the overlay first already avoids the wasted
call; the alias makes the wrong guess harmless either way.)

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the
  `networking.set_net_role` multiplayer-pickup build
  (`/Game/Multiplayer/BP_HealthPickup`, outcome `gap`; the readback gap itself
  filed by the judge as `F-networking-info-no-rpc-detail`, whose `#8` entry
  replays this same task). Distinct PROCESS angle from that `F-` gap: this is the
  `property.get` `assetPath`→`objectPath` param round-trip the agent hit when it
  fell back to a raw reflection read to confirm `bReplicateMovement`. First call
  `property.get {assetPath:...,propertyName:bReplicateMovement}` hard-failed
  `[MISSING_REQUIRED_PARAM] Missing required parameter 'objectPath' (type:
  string)`; after a wiki-nav to `property.get.md`, the retry with
  `{objectPath:...}` returned `value:true`. Source confirms the bare
  `objectPath` slot with no `assetPath` alias across the `property.*`/`container.*`
  family (UtilityPropertyHandler.cpp: `property.set` :825/:833, `property.get`
  :1221/:1229, `property.list` :1527/:1541, container verbs :1800+). Sibling of
  the param-alias-drift cluster (`E-blueprint-param-name-path-vs-assetpath` DONE,
  `E-asset-path-vs-assetpath-list-drift` OPEN, `E-niagara-set-property-emitter-alias`),
  here on the `objectPath`↔`assetPath` axis in the `property.*` reflection
  namespace — no existing ticket covers this pair. Friction note (verbatim):
  "property.get rejects the assetPath alias (needs objectPath), costing one extra
  wiki-nav and retry." One round-trip + one wiki-nav, recovered immediately, zero
  blocked progress. Fix: dispatcher `FParamSpec` alias annotating the `property.*`
  `objectPath` slot to accept `assetPath` + `GetStringFirstOf` body read; plus a
  `docs/wiki-src/property.md` overlay note that the slot accepts an asset/blueprint
  path, an actor name, or a full object path.
