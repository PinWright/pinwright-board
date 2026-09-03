---
id: B-call-nonobject-args-dispatches-defaults
title: "The call transport replaces present non-object argument envelopes with empty objects, so malformed input can dispatch a method's destructive defaults"
status: OPEN
severity: High
category: bug
tags: [transport, call, validation, coercion, false-success, run-tests]
encounters: 1
lastSeen: 2026-09-03T23:17:31+03:00
---

# Non-object `call` arguments are silently replaced with `{}`

## What happens

`Transport/McpRequestCore.cpp:631-644` accepts a present `params.arguments` value of any
non-object JSON type and substitutes an empty object. That routes the request to the wiki-root
success path at `:720-744` instead of returning invalid params. More dangerously, a valid outer
object containing `method` plus a present non-object `args` passes `bHasArgsField` at `:717-718`,
then `:787-800` replaces that value with `{}` and dispatches the requested method.

This is not only a cosmetic fallback. For
`{method:"system.run_tests", args:"PinWright.transport"}`, the empty payload reaches
`SystemControlHandler.cpp:140-142`, selects `bRunAll`, builds `Automation RunAll` at `:490-492`,
and starts the job at `:551`. A malformed narrow test request therefore launches the entire
automation suite and returns a normal job receipt.

## Why it matters

The caller cannot distinguish malformed input from a deliberately empty argument object. On
methods whose empty payload means “all”, “clear”, or a default target, this can run expensive or
destructive work the caller did not request. Severity is High: this is silent false success on
the transport choke point, with a concrete `system.run_tests` execution path.

## What should happen

Reject a present non-object `params.arguments` and a present non-object inner `args` with JSON-RPC
`-32602` before wiki routing or dispatch. Add failure-direction request-core tests that send
string/array/null values and assert both the error and that a capture handler was never invoked.
This is the envelope-level counterpart of the runtime `FParamSpec` gate from
`B-param-type-never-validated`; do not silently synthesize `{}` for a value the caller supplied.

**Workaround:** Always send both `params.arguments` and inner `args` as JSON objects.

## Related

- Catalog: `accepted-parameter-silent-noop`, `coercion-slot-unit-direction-drift`
- `E-call-unknown-arg-fields-silently-ignored` — sibling validation at the same transport layer;
  it covers unknown field names and string typing for `method`/`path`, not object typing here.
- `B-param-type-never-validated` — per-handler value-shape gate after dispatch.

## History
- `#1-nonobject-args-default-dispatch` `OPEN` reporter — Source-read the full request path: both
  object slots substitute `{}` on wrong shape, and malformed inner args can dispatch
  `system.run_tests` as `Automation RunAll`. No RPC, build, or editor run was performed.
