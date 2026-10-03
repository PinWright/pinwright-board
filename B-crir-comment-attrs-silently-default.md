---
id: B-crir-comment-attrs-silently-default
title: "CRIR accepts malformed or unknown comment attributes and reports success after replacing them with defaults"
status: DONE
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

- `#2-closed-comment-schema` `IN-REVIEW` developer — Still reproducible before the fix (source: `ParseCommentInstruction` skipped bare flags/unknown keys and stored size/color unvalidated; `TryParseDoubleTuple` uses `Atod`, so `size=(wide,tall)` even became 0x0, not the default). `CRIR/CRIRParser.cpp`: closed schema `size`/`color` only, each at most once, exact 2/4-tuple of finite numbers (plain or `%g` exponent form, which the decompiler emits); a bare flag, unknown key (`colour=`), duplicate, empty value, wrong arity, trailing/empty component, unit suffix, inf/overflow fails the parse with new parse code `CRIR_BAD_COMMENT_ATTR` on that line, before any mutation. Also rewrote `TryReadFirstToken`'s one-pass loop (behaviour identical) because the modified file is compiled standalone by adaptive unity and clang rejects it with `-Wunreachable-code-loop-increment`. `CRIR/CRIRCompiler.cpp`: stale "silently fall back" comment corrected. Test `PinWright.CRIR.Parse.CommentAttrsRefuseInvalid` (in `Tests/Assets/TestCRIRComment.cpp`; 12 refusal cases asserting code+line, 3 acceptance cases asserting the stored tuples). Docs: `docs/crir-language-reference.md` (comment), `docs/wiki-src/controlrig.md`, `CHANGELOG.md`. Fastcheck OK (clang -fsyntax-only, UBT flags); not run in an editor yet.

- `#3-review-followups` `IN-REVIEW` developer — Added `CRIR_BAD_COMMENT_ATTR` to the error table in `docs/crir-language-reference.md`, added the `size= (1,2)` refusal case to `PinWright.CRIR.Parse.CommentAttrsRefuseInvalid`, and noted in `CHANGELOG.md` that a comment with non-finite size/color decompiles to `inf`/`nan`, which the closed schema now refuses on recompile (decompiler unchanged). Fastcheck OK.
- `#4-verified-linux` `DONE` tester — Passed non-skipped in run3/full: `PinWright.CRIR.Parse.CommentAttrsRefuseInvalid` (13 refusal cases asserting code `CRIR_BAD_COMMENT_ATTR` and line, including `size=(wide,tall)`, `colour=`, bare flags, wrong arity, duplicates, non-finite and `size= (1,2)`; 3 acceptance cases asserting the stored tuples). Acceptance met: the closed `size`/`color` schema is validated in the parser with exact finite 2/4-tuples and a line-specific typed parse error before any mutation, and defaults apply only to omitted attributes. Doc verified: `docs/crir-language-reference.md` lists the new error code (#3).
