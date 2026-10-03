---
id: B-foliage-paint-location-object-refused
title: "`foliage.paint` refuses `location: {x,y,z}` with PARAM_TYPE_MISMATCH although the handler reads that shape, because `location` was an untyped alias of the `array` param `locations`"
status: DONE
severity: Low
category: bug
tags: [foliage, foliage-paint, param-spec, declared-type, param-type-mismatch, alias, typed-alias]
encounters: 1
lastSeen: 2026-10-02T20:22:29Z
---

# A shape the handler reads is unreachable through the type gate

Found by the class contract test
`PinWright.infra.dispatcher.ParamTypeGate.ShapedReadersDeclaredToAdmitTheirShape` (ticket
`B-omit-slot-chain-declared-integer`, #2) on its first suite run:

```
Expected 'foliage.paint.location is read as object (FoliageHandler.cpp:798) but declared 'array',
which the type gate refuses for that shape' to be true.
```

## Evidence

`Source/PinWright/Private/Handlers/Environment/FoliageHandler.cpp`, `foliage.paint` registration:

```cpp
RPC_PARAM_OPT_ALIAS("locations", "array", "Array of {x,y,z} positions ...", "location"),
RPC_PARAM_OPT("position", "object", "Single {x,y,z} position (alternative to locations)"),
```

The body reads `location` in **both** shapes: `TryGetArrayField(TEXT("location"))` as a
`locations` fallback (`:774`) and `TryGetObjectField(TEXT("location"))` as a `position`
fallback (`:798`). An untyped alias inherits its spec's type
(`RpcDispatcher.cpp` `CollectDeclaredTypesByWireName`), so `location` was declared `array`, and
`location: {"x":0,"y":0,"z":0}` was refused `PARAM_TYPE_MISMATCH` before the body ran. The object
branch at `:798` was dead code. Present since the 0.8.0 import.

Workaround before the fix: send `position` (or `locations: [{...}]`).

## History

- `#1-filed-from-shape-contract-test` OPEN reporter — Filed from the run1 suite failure of
  `PinWright.infra.dispatcher.ParamTypeGate.ShapedReadersDeclaredToAdmitTheirShape`. Old defect
  exposed by the new test, not introduced by it. No existing foliage test sends `location` to
  `foliage.paint`, which is why nothing caught it.
- `#2-typed-alias-array-object` IN-REVIEW developer — `location` moved from an untyped alias of
  `locations` to a typed alias (`FParamSpec::TypedAliases`) declared `array|object`, matching both
  reads in the body; `locations` itself stays `array`. Files:
  `Source/PinWright/Private/Handlers/Environment/FoliageHandler.cpp` (registration only),
  `docs/wiki-src/foliage.md` (one sentence under `foliage.paint` naming `location`'s two shapes),
  `CHANGELOG.md` (Fixed entry). Regression test is the contract test above: reverting the
  declaration makes it fail on exactly this site. Fastcheck OK; not yet run in an editor.
- `#3-verified-linux` `DONE` tester — Passed non-skipped in run3/full: `PinWright.infra.dispatcher.ParamTypeGate.ShapedReadersDeclaredToAdmitTheirShape` (the test that failed on exactly this site in run1 now passes, so the real gate admits an object for `foliage.paint.location`), plus every `PinWright.foliage.paint.*` test (e.g. `ValidParamsNoCrash`, `NonObjectLocationEntriesPlaceNothing`, `LocationWithoutCoordinatesIsReportedNotPlacedAtTheOrigin`). Acceptance met: `location` is a typed alias declared `array|object`, matching both reads in the body, while `locations` stays `array`. Coverage limit: no test dispatches `foliage.paint {location:{x,y,z}}` end to end; the object shape is shown at the type gate, not by a placed instance.
