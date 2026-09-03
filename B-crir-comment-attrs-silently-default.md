---
id: B-crir-comment-attrs-silently-default
title: "CRIR accepts malformed or unknown comment attributes and reports success after replacing them with defaults"
status: OPEN
severity: Medium
category: bug
tags: [crir, control-rig, comments, validation, silent-default, false-success]
---

# Invalid CRIR comment attributes silently become size 400x300 and black

`ParseCommentInstruction` accepts bare flags, skips them, and ignores every key except `size` and `color` (`Plugins/PinWright/Source/PinWright/Private/CRIR/CRIRParser.cpp:442-506`). It stores those two values without validating their tuple arity or numeric types. During emission, failed `TryParseDoubleTuple` calls leave the hard-coded `Size(400,300)` or black color in place (`CRIR/CRIRCompiler.cpp:431-446`), then `AddCommentNode` succeeds and the outer compile returns `bSuccess=true` (`:1580-1586`). No warning identifies the discarded text.

For example, `comment "Note" size=(wide,tall) color=(1,0,0)` compiles successfully but authors neither requested value. A typo such as `colour=` is dropped completely. The caller cannot distinguish a correctly applied request from a defaulted one without decompiling or inspecting the graph.

Validate the closed comment-attribute schema in the parser: reject unknown/bare keys and require exact finite numeric tuples of 2 and 4 values before mutation. Return a line-specific typed parse error. Do not use compiler defaults for a parameter that was present but invalid; defaults are only for omitted attributes.

**Workaround:** use exact `size=(number,number)` and `color=(number,number,number,number)` forms, then decompile to verify them.

## Related

- Catalog: `accepted-parameter-silently-dropped`, `accepted-parameter-silent-noop`, `coercion-slot-unit-direction-drift`

## History

- `#1-pattern-scan` `OPEN` reporter — Source-confirmed parser acceptance, compiler defaulting, and successful return; no editor, build, or test was run.
