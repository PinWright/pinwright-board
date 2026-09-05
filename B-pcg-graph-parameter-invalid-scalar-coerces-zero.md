---
id: B-pcg-graph-parameter-invalid-scalar-coerces-zero
title: "pcg.add_graph_parameter accepts invalid numeric and boolean strings, coerces them to zero or false, and returns valueSet:true without readback"
status: IN-REVIEW
severity: High
category: bug
tags: [pcg, graph-parameter, coercion, atoi, atof, bool, silent-wrong-value, request-echo, validation]
encounters: 1
lastSeen: 2026-09-03T00:00:00Z
---

# Invalid scalar text becomes a valid but unintended graph-parameter value

`PCGGraphParameters.cpp:113-125` parses values with `FString::ToBool` and the permissive
`FCString::Atoi/Atoi64/Atof/Atod` family. Those conversions do not report parse failure: an invalid
numeric string such as `"not-a-number"` becomes zero and an unrecognised boolean becomes false.
The RPC deliberately declares `value` as a string (`:253`), so these malformed strings pass the
dispatcher. The handler then adds the property, gets a successful typed bag write, marks the graph
dirty, and responds only with `valueSet:true` (`:303-337`); it never reads the stored value back.

A typo therefore returns success and silently changes graph behaviour to a plausible default-like
value. `pcg.list_graph_parameters` can reveal the damage only in a separate call.

## What should happen

Use strict, full-string scalar parsing before `MutateUserParameters`; reject non-finite numbers and
unrecognised boolean tokens without adding or changing a property. Return the stored typed value
read from the bag after mutation, separate from any requested text. Tests should cover trailing
junk, empty text, non-finite floats, and invalid booleans, and prove the bag is unchanged on error.

**Workaround:** validate scalar text client-side, then call `pcg.list_graph_parameters` and compare
the typed value after every add/upsert.

## Related

`F-pcg-authoring-parity` added the CRUD surface but does not cover invalid-value coercion.

## Fix

Root cause: `pcg.add_graph_parameter` converted scalar text with permissive `ToBool`/`Atoi`/
`Atoi64`/`Atof`/`Atod` calls inside the live bag mutation, so malformed input silently became a
default-like value. The handler now parses numeric and boolean values with the shared strict
`Utils/JsonUtils.*` parsers before `MutateUserParameters`, returns typed `INVALID_VALUE`, and
includes the stored scalar readback in successful responses.

Files changed:
- `Plugins/PinWright/Source/PinWrightPCG/Private/Handlers/PCG/PCGGraphParameters.cpp`
- `Plugins/PinWright/Source/PinWrightPCG/Private/Tests/PCG/PCGGraphParametersTests.cpp`
- `Plugins/PinWright/Docs/wiki-src/pcg.md`

Behavioral test: `PinWright.pcg.graph_parameter.InvalidScalarRejectedWithoutMutation` covers
trailing-junk, non-finite, empty, and invalid-boolean strings and proves existing values and new
descriptors are unchanged on rejection.

Deliberately unchanged: the shared strict parser implementation, property-bag CRUD type list,
name sanitization, list/remove handlers, and valid scalar conversion semantics. Verification is
source-only in this turn; no build, editor, or automation run was performed.

## History
- `#1-source-pattern-scan` `OPEN` reporter — Permissive string conversions feed a successful property-bag setter and the response reports only `valueSet:true`, so invalid text becomes 0/false without an error or readback. Source-only; no RPC was run.
- `#2-strict-scalar-reject` `IN-REVIEW` developer — Added pre-mutation strict scalar parsing, typed invalid-value rejection, stored-value readback, and handler-level unchanged-state tests. Source-only; runtime verification remains for the tester.
