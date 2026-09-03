---
id: B-bpir-override-errors-line
title: "BPIR override diagnostics discard authored source positions and report Line -1"
status: OPEN
severity: Medium
category: bug
tags: [bpir, compiler, override, diagnostics, source-position]
encounters: 1
lastSeen: 2026-09-03T20:21:31+03:00
---

# Override diagnostics lose the BPIR source line

## What happens

Override creation and signature validation append `FCompileError(-1, ...)` for
failures in expected-signature construction and signature comparison
(`Source/PinWright/Private/Compiler/BpirCompiler.cpp:4614-4631`). The later
pre-existing override resolution path does the same for resolution, expected
signature, and actual signature failures (`BpirCompiler.cpp:5249-5290`). These
errors are raised while processing a parsed authored block, but the diagnostic
throws away its BPIR location.

## Why it matters

Users receive `Line -1` for actionable override mistakes and must isolate blocks or
correlate messages by function name. Severity is Medium: compilation fails visibly,
but locating the failure requires a source dive or repeated smaller calls.

## What should happen

Carry the authored block's source position into every override-specific compile
error, with tests covering resolution and signature mismatch diagnostics in a
multi-entry document.

## Workaround

Compile overrides one at a time and use the function name in the error message to
locate the offending block.

## Related

- `B-add-function-outputs-become-inputs`
- `B-interface-function-with-outputs-unimplementable`
- `B-bpir-single-named-output-forced-returnvalue`

These are the wave-6 function-output tickets whose grouped review exposed the
diagnostic-position gap.

## History
- `#1-filed-wave-6-follow-up` `OPEN` reporter — Source-only verification confirmed override error construction with line `-1` at `BpirCompiler.cpp:4614-4631,5249-5290`. No compile, test, editor, or MCP call was run. Severity Medium because the error is visible and compilation stops, but diagnosis requires isolation or source correlation.
