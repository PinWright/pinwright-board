---
id: E-volume-type-filter-discovery
title: "volume.get_volumes_info type filter is undiscoverable — volumeType wants unprefixed class name, create echoes the A-prefixed form"
status: OPEN
severity: Low
category: ergonomic
tags: [volume, docs, discovery, filter, class-name]
encounters: 1
lastSeen: 2026-06-17T00:56:02Z
---

# volume.get_volumes_info type filter is undiscoverable

Filtering `volume.get_volumes_info` to one volume class is a guessing game. The
handler takes two optional string params, `filter` (name) and `volumeType`
(type), both documented with one-word descriptions and no accepted values. To
list blocking volumes the caller must arrive at exactly `volumeType="BlockingVolume"`
— but the obvious candidates all silently fail or error first, because:

1. `volumeType` matches via `ClassName.Contains(VolumeType)` against
   `Volume->GetClass()->GetName()`, which returns the **unprefixed** runtime
   class name (`BlockingVolume`). The `A`-prefixed form `ABlockingVolume`
   therefore matches nothing and returns `{volumes:[]}` with **no error** — a
   silent empty result, not a hint that the value was wrong.
2. `create_blocking_volume`'s own success payload echoes
   `volumeClass:"ABlockingVolume"` (the `A`-prefixed form). An agent that read
   that field back as the filter value gets zero results — the create response
   actively misleads the filter.
3. `filter` is a **name** filter (substring of `GetActorLabel()`), not a type
   filter, so `filter="BlockingVolume"` also returns zero (no actor is *labeled*
   "BlockingVolume"). The two params look interchangeable from their terse docs.

Net: the agent burned three throwaway calls — `volumeClass` (UNKNOWN_PARAMS) →
`filter=BlockingVolume` (0 results) → `volumeType=ABlockingVolume` (0 results) —
before landing on `volumeType=BlockingVolume`. Every wrong attempt that used a
*valid* param name returned a clean empty list, so there was no error to learn
from; discovery was pure trial and error.

## What's wrong

`Source/PinWright/Private/Handlers/Volume/VolumeHandler.cpp:1538-1542`:

```cpp
REGISTER_RPC_HANDLER("volume.get_volumes_info", "volume", "Get information about all volumes in the level",
    RPC_PARAMS(
        RPC_PARAM_OPT("filter", "string", "Name filter"),
        RPC_PARAM_OPT("volumeType", "string", "Volume type filter")
    ))
```

The match at line 1510-1513 is `ClassName.Contains(VolumeType)` where
`ClassName = Volume->GetClass()->GetName()` — i.e. the unprefixed name and a
substring contains-test. Nothing documents the accepted vocabulary, the
unprefixed convention, or that it differs from the `volumeClass` the create
handlers emit (which carry the `A` prefix, e.g. line 700
`SetStringField("volumeClass", "ABlockingVolume")`).

## What it should do

Either of (preferably both):

1. **Ergonomic:** make `volumeType` tolerant of the form callers already have —
   strip a leading `A` before the contains-test (and/or accept the
   `volumeClass` value the create handlers echo verbatim), so
   `ABlockingVolume`, `BlockingVolume`, and `blocking` all resolve. An empty
   match could optionally surface the set of class names present in the level so
   a wrong filter is self-correcting instead of a silent `[]`.
2. **Docs (`docs/wiki-src/volume.md`):** the overlay is a 5-line prelude with no
   `### volume.get_volumes_info` section. Add one that (a) distinguishes
   `filter` (name substring) from `volumeType` (class substring), (b) states the
   accepted `volumeType` values are unprefixed runtime class names
   (`BlockingVolume`, `TriggerBox`, `PostProcessVolume`, …) — explicitly NOT the
   `A`-prefixed `volumeClass` string returned by the create methods, and (c)
   notes the special-cased `"Trigger"` value (line 1552) that matches all
   `ATriggerBase` actors.

## Evidence

From the arena-walls task (focus `volume.create_blocking_volume`). Friction
note, verbatim: *"discover the type filter is volumeType=\"BlockingVolume\" not
\"ABlockingVolume\"/\"filter\""*. Call-log shows four `get_volumes_info` readback
attempts at the end: `volumeClass` → `[UNKNOWN_PARAMS]`; `filter=BlockingVolume`
→ ok but 0 volumes; `volumeType=ABlockingVolume` → ok but 0 volumes;
`volumeType=BlockingVolume` → ok, the 5 named volumes. Three wasted round-trips
for one "list the blocking volumes" intent.

Distinct from `B-blocking-volume-no-brush-geometry` (that ticket is the brush
being null / zero collision; this is purely the *filter-value discovery* friction
on the readback path) and from the param-name-drift E-tickets
(`E-widget-remove-widget-param-name` etc., which are about *param names*; here
the param name `volumeType` is correct — it's the accepted *value vocabulary*
that's undiscoverable and contradicted by the create response).

## History
- `#1-initial-audit` `OPEN` reporter — Filed from the arena-walls struggle audit (seed `volume.create_blocking_volume`). Listing blocking volumes cost three failed calls before `volumeType="BlockingVolume"` worked: `volumeClass` (UNKNOWN_PARAMS), `filter="BlockingVolume"` (name filter, 0 results), `volumeType="ABlockingVolume"` (0 results — the create handler echoes `volumeClass:"ABlockingVolume"` but the filter matches the unprefixed `GetClass()->GetName()="BlockingVolume"` via `ClassName.Contains`). Source: `VolumeHandler.cpp:1477` (registration), `:1497`/`:1510-1513` (volumeType contains-match on unprefixed name), `:1496`/`:1515-1518` (filter is name/label substring), `:1552` (special-cased "Trigger"). Wrong-but-valid filter values return a silent empty list, so there is no error to learn from. Proposed: strip leading `A` / accept the echoed `volumeClass` form, and document `volumeType`'s accepted vocabulary in a new `### volume.get_volumes_info` section of `docs/wiki-src/volume.md`.
- `#2-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place and verified against plugin HEAD `ef8a1f1b`. `:1477` had drifted onto an INVALID_ARGUMENT bounds check in a different handler; the quoted `REGISTER_RPC_HANDLER("volume.get_volumes_info", …)` block with its `filter` / `volumeType` `RPC_PARAM_OPT`s matches verbatim at `:1538-1542`. Sweep-wide record, including every case that could not be repointed: `E-module-rename-citation-sweep`.
