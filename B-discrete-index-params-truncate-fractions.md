---
id: B-discrete-index-params-truncate-fractions
title: "Discrete index parameters accept fractional JSON numbers, truncate them to int32, and can mutate the wrong array element or local player"
status: OPEN
severity: High
category: bug
tags: [parameters, integer, coercion, index, container, session]
encounters: 1
lastSeen: 2026-09-03T23:17:31+03:00
---

# Fractional indices are silently truncated

## What happens

The declared-type gate treats `number` and `integer` identically and accepts every JSON number
(`Handlers/ParamTypeCheck.h:337-345`). `FHandlerContext::GetInt` then calls
`FJsonObject::GetIntegerField` (`Handlers/HandlerContext.cpp:45-50`), whose UE 5.8 implementation
is a direct C-style cast from double to `int32`
(`C:/UE_5.8/Engine/Source/Runtime/Json/Private/Dom/JsonObject.cpp:409-412`).

All four Utility array index slots are declared `number` and read through that path:
remove (`UtilityPropertyHandler.cpp:2299-2304`), insert (`:2371-2377`), get (`:2438-2443`), and
set (`:2492-2498`). Thus `index:1.9` becomes `1`; remove/set mutates element 1 and returns success
for that truncated index. `session.remove_local_player` repeats the same mechanism for
`playerIndex` (`SessionsHandler.cpp:109-116`), so a fractional selector can remove the wrong player.

## Why it matters

Range checks run after truncation, so they certify a different target than the caller supplied.
This is High silent wrong-target behavior on destructive methods.

## What should happen

Add one exact-int reader/validator that accepts only finite, integral values in `int32` range, and
use it for discrete slots. Retype these declarations to `integer` and stop normalizing `integer`
to unrestricted `number`, or enforce the distinction at the handler helper if wire compatibility
requires keeping existing declarations. Failure must occur before `Modify`, removal, insertion, or
player teardown. Add negative cases for fractions and out-of-range numbers.

**Workaround:** Send whole JSON numbers only and confirm the returned/read-back target identity.

## Related

- Catalog: `coercion-slot-unit-direction-drift`, `wrong-target-scope-or-identity`
- `B-param-type-never-validated` — fixed wrong JSON shapes but deliberately normalizes
  `integer` to `number`; it does not reject non-integral numbers.

## History
- `#1-fractional-index-truncation` `OPEN` reporter — Source-read the shared accessor, declared-type
  gate, UE JSON cast, and destructive array/player call sites. No runtime mutation was performed.
