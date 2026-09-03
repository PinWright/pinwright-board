---
id: B-niagara-switch-coercion
title: "Integer Niagara static switches silently coerce fractional, unparsable, and unsupported JSON values"
status: OPEN
severity: Medium
category: bug
tags: [niagara, static-switch, validation, integer, coercion]
encounters: 1
lastSeen: 2026-09-03T20:21:31+03:00
---

# Integer static switches silently coerce malformed values

## What happens

Integer switch encoding initializes the value to zero, converts booleans to 0/1,
truncates JSON numbers with `static_cast<int32>`, parses strings with permissive
`FCString::Atoi`, and leaves unsupported JSON types at zero
(`Source/PinWright/Private/Handlers/Niagara/NiagaraEditTypes.cpp:2556-2572`).
The handler validates only the resulting integer option and then commits that pin
default (`Handlers/Niagara/NiagaraEditHandler.cpp:3149-3200`). Thus `1.9`, an
unparsable string, or an object can become a valid branch index without an error.

## Why it matters

Invalid caller input can silently select branch zero or another unintended branch,
and the mutation is then compiled and optionally saved. Severity is Medium: this is
silent wrong data, discounted for the malformed-input edge path and the response
echo that callers can verify.

## What should happen

Accept only explicitly supported forms. Require JSON numbers to be finite integral
values, parse any supported string form strictly with full-consumption checks, reject
null/array/object input, retain the established boolean 0/1 contract, and run option
range validation only after type validation. Add regressions for each rejected form.

## Workaround

Send an integral JSON number (or the documented boolean form) and verify the echoed
value before saving.

## Related

- `B-niagara-static-switch-bool-writes-zero-on-int-switch` — wave-6 ticket fixed
  boolean conversion and integer range validation; its review exposed the remaining
  permissive coercions.

## History
- `#1-filed-wave-6-follow-up` `OPEN` reporter — Source-only verification confirmed truncating numeric conversion, permissive `Atoi`, and zero fallback at `NiagaraEditTypes.cpp:2556-2572`, followed only by coerced-range validation and commit at `NiagaraEditHandler.cpp:3149-3200`. No Niagara mutation, build, test, editor, or MCP call was run. Severity Medium because the silent wrong branch requires malformed input and the echoed value provides a manual check.
