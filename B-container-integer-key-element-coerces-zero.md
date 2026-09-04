---
id: B-container-integer-key-element-coerces-zero
title: "Container map keys and set elements use unchecked Atoi conversions, so malformed values can overwrite, add, or remove integer zero"
status: IN-REVIEW
severity: High
category: bug
tags: [container, map, set, coercion, integer, false-success]
encounters: 1
lastSeen: 2026-09-03T23:17:31+03:00
---

# Invalid container key and element values become integer zero

## What happens

`container.map.set` accepts a string key and, for an `FIntProperty` key, writes
`FCString::Atoi(*Key)` then unconditionally marks conversion successful
(`Handlers/Utility/UtilityPropertyHandler.cpp:2561-2601`). It commits the pair at `:2638` and
echoes the original string at `:2652-2658`. A key such as `"not-an-int"` therefore adds or
overwrites key `0` while the response says the key was `"not-an-int"`.

The set family implements the same unchecked conversion locally. `container.set.add` maps any
non-number through `FCString::Atoi` at `:2954-2958` and commits it at `:2981`; numeric fractions
are cast directly to `int32`. `container.set.remove` repeats the conversion while scanning at
`:3045-3050`, so a request to remove `"not-an-int"` can delete a real `0` element and return
success. The sibling `contains` path uses the same conversion mechanism. Because `value` is
declared `any`, the top-level ParamSpec gate cannot infer the reflected element type.

## Why it matters

Malformed input targets a valid, unrelated key/element. Map set can clobber its value and set
remove can delete it, with normal success responses. Severity is High silent wrong-target data.

## What should happen

Convert the key or element once, before `RootObject->Modify()` and before any map/set mutation or
scan. Require a complete, finite, in-range integral parse for integer properties and reject other
JSON shapes with a typed conversion error. Build the response from the canonical converted value,
not the request string. Share the strict scalar conversion used by reflected-property import
rather than maintaining another Atoi path.

**Workaround:** Use canonical integer strings for integer map keys and native whole JSON numbers for integer set elements; read back the container after mutation.

## Fix

Implemented the root-cause fix in `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Utility\UtilityPropertyHandler.cpp`: integer map keys and set elements now consume the shared strict scalar parser from `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Utils\JsonUtils.h/.cpp` before allocation, `Modify()`, mutation, or scanning. Malformed, fractional, overflowing, and non-scalar values return `INVALID_PARAMS` instead of targeting integer zero, and successful map responses use canonical integer keys.

Regression coverage is in `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\Utility\TestContainerIntegerScalarCoercion.cpp`, using fixture `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\Utility\TestContainerIntegerScalarCoercionHost.h` and automation id `PinWright.container.IntegerScalarCoercion`. Float container coercion, array rollback behavior, and unrelated handlers were deliberately left unchanged.

## Related

- Catalog: `coercion-slot-unit-direction-drift`, `wrong-target-scope-or-identity`, `request-echo-not-result-readback`
- `B-property-import-malformed-scalars-coerce-zero` — sibling helper-based value conversion; these key/element paths are separate local code.
- `B-container-set-fname-lookup-broken` — different element-type branch and already in review.

## History
- `#1-atoi-targets-zero` `OPEN` reporter — Source-read map set and the set add/remove/contains
  family and confirmed malformed values reach key/element zero with no parse verdict. No container
  was mutated during the scan.
- `#2-checked-integer-containers` `IN-REVIEW` developer — Reused the shared strict scalar conversion for map keys and set elements, validating before Modify/mutation/scan and returning canonical values; added `PinWright.container.IntegerScalarCoercion` regression coverage.
