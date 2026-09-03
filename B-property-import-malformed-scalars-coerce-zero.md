---
id: B-property-import-malformed-scalars-coerce-zero
title: "Generic reflected-property writes accept malformed scalar strings and report success after coercing them to zero or false"
status: OPEN
severity: High
category: bug
tags: [property, reflection, coercion, false-success, property-import]
encounters: 1
lastSeen: 2026-09-03T23:17:31+03:00
---

# Reflected scalar import treats parse failure as a valid value

## What happens

The in-scope `property.set` handler declares `value` as `any`
(`Handlers/Utility/UtilityPropertyHandler.cpp:1112-1117`) and routes it to
`ApplyJsonValueToProperty` at `:1333-1343`. That helper is outside this scanner's assigned area,
so it was followed only as the root cause: `Utils/PropertyImport.cpp:380-410` accepts every string
for float/double and parses with `FCString::Atod`; `:414-445` accepts every string for int32/int64
and parses with `FCString::Atoi64`; and `:316-335` maps every string other than `"true"` to false.
None checks whether conversion consumed a valid literal.

The helper returns true after writing the fallback value. `property.set` therefore sends success,
runs change notifications, and reads back `0`/`false` at
`UtilityPropertyHandler.cpp:1345-1379` for inputs such as `"not-a-number"` or `"yes"`. The same
helper is reached by in-scope `container.array.append`/`insert`/`set` and map-value writes at
`:2272`, `:2411`, `:2530`, and `:2626`, so malformed scalar values can be committed there too.

## Why it matters

This is a normal generic authoring path. A parse error becomes a real property edit and a trusted
success receipt rather than an error, so callers can zero asset settings or disable a boolean
without intending to. Severity is High.

## What should happen

Make the shared property importer reject malformed strings, non-finite values, integer fractions,
and integer overflow before writing. Reuse the strict literal semantics already defined by
`PinWrightIsStrictNumericLiteral` and `PinWrightIsBooleanLiteral` in `ParamTypeCheck.h`, preferably
through a shared scalar-conversion helper so dispatcher and reflection import cannot drift. Add
handler-level negative tests with independent property readback proving the old value survives.

**Workaround:** For reflected numeric and boolean properties, send native JSON numbers/booleans and read the property back.

## Related

- Catalog: `coercion-slot-unit-direction-drift`, `accepted-parameter-silent-noop`
- `B-param-type-never-validated` — cannot protect `value:any`; this needs runtime-property-aware conversion.
- `B-container-array-conversion-failure-leaves-element` — owns rollback after a genuine conversion error.

## History
- `#1-malformed-scalars-write-fallbacks` `OPEN` reporter — Traced each in-scope Utility call through
  the shared importer and its unconditional Atoi/Atod/boolean fallback. Root cause is outside the
  assigned area and was not scanned beyond this call path. No RPC was executed.
