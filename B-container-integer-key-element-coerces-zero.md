---
id: B-container-integer-key-element-coerces-zero
title: "Container map keys and set elements use unchecked Atoi conversions, so malformed values can overwrite, add, or remove integer zero"
status: OPEN
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

## Related

- Catalog: `coercion-slot-unit-direction-drift`, `wrong-target-scope-or-identity`, `request-echo-not-result-readback`
- `B-property-import-malformed-scalars-coerce-zero` — sibling helper-based value conversion; these key/element paths are separate local code.
- `B-container-set-fname-lookup-broken` — different element-type branch and already in review.

## History
- `#1-atoi-targets-zero` `OPEN` reporter — Source-read map set and the set add/remove/contains
  family and confirmed malformed values reach key/element zero with no parse verdict. No container
  was mutated during the scan.
