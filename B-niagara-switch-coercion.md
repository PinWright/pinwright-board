---
id: B-niagara-switch-coercion
title: "Integer Niagara static switches silently coerce fractional, unparsable, and unsupported JSON values"
status: DONE
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
- `#2-strict-integer-switch-values` `IN-REVIEW` developer — Still reproduced in source (the cited lines had moved to `NiagaraEditTypes.cpp` `EncodePinDefault` Integer case: `static_cast<int32>`, `FCString::Atoi`, zero fallback). Fix in the shared static-switch coercion path, `Source/PinWright/Private/Handlers/Niagara/NiagaraEditTypes.cpp`: new `TryReadIntegralSwitchNumber` (finite, integral, within int32) used for Integer switches and for the enum-switch index in `ResolveEnumOption` (which also serves enum module inputs now — `1.9` used to truncate to branch 1 there too); strings must be a full integer literal (`IsIntegerLiteral` + `LexTryParseString`, trimmed); null/array/object refused; boolean 0/1 kept. The handler already sends `INVALID_VALUE` on encode failure and range-validates only after it. Bool switches were left as they were (out of this ticket's scope). Tests: `PinWright.niagara.set_static_switch.IntegerValueRejectsMalformed` (new, `Tests/Niagara/TestNiagaraStaticSwitchInteger.cpp`), fractional-index case added to `PinWright.niagara.set_static_switch.EnumBranchResolution`. Wiki `niagara.md` (`set_static_switch`) and CHANGELOG updated.
- `#3-review-int32-wrap` `IN-REVIEW` developer — Review follow-up: an integer string past int32 (`"4294967298"`) still wrapped to branch 2, because `LexTryParseString` into an `int32` parses through a 64-bit `strtol` and truncates. New `TryParseIntegerLiteral` (`NiagaraEditTypes.cpp`) caps the literal at 11 characters, parses it as int64 and range-checks it through `TryReadIntegralSwitchNumber`. It is used by the Integer-switch encoder and by the enum index path in `ResolveEnumOption`. Tests: the `"4294967298"` case was added to `PinWright.niagara.set_static_switch.IntegerValueRejectsMalformed` and `.EnumBranchResolution`. Also: `ResolveEnumOption`'s no-enum and no-entries messages no longer say "static switch" (module inputs reach them too), and a JSON null is reported as `null`.
- `#4-verified-linux` `DONE` tester — Verified on Linux, UE 5.8, PinWright 7230b41d (commit 75e314ce). run3/full passed non-skipped: `PinWright.niagara.set_static_switch.IntegerValueRejectsMalformed`, `.IntegerValueValidation`, `.EnumBranchResolution` and `.EnumBranchTablePublished`. Acceptance: on an Integer switch, `IntegerValueRejectsMalformed` refuses fractional 1.9 and -0.5, 1e12, the strings `abc`, `1x`, `1.9`, `4294967298` and empty, plus null, an array and an object. Integral 2.0 -> `2`, -3 -> `-3`, trimmed ` 4 ` -> `4` and boolean true -> `1` still encode, so the boolean 0/1 contract is retained. Range validation runs after type validation (`IntegerValueValidation`). The fractional and int32-overflow index cases on the enum path are in `EnumBranchResolution`. Bool switches were deliberately left unchanged, which is outside this ticket's scope.
