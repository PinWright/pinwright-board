---
id: B-bpir-timeline-multiline-example-unparseable
title: "BPIR timeline instruction must be one line, but the documented example is multi-line"
status: OPEN
severity: Low
category: bug
tags: [bpir, timeline, docs, wiki-src, parser]
---

# BPIR timeline instruction must be one line, but the documented example is multi-line

The BPIR parser is strictly line-oriented: `FBpirParser::Parse` /
`ParseBody` split `Code` on `\n` (`BpirParser.cpp:455`, `:643`) and process one
physical line at a time, and `ParseTimelineInstruction` resolves args with
`FindMatchingParen` over that single line's `Rest` (`BpirParser.cpp:2360-2378`).
There is no line-continuation / paren-join logic anywhere. So a `timeline`
instruction whose parenthesized args span multiple physical lines cannot parse.

But the documented example presents exactly that multi-line form — `%tl = timeline FadeIn(`
on one line, `Alpha: float_curve(...)` on the next, `) [update -> ...]` on a third
(`wiki-generated/bpir.examples.timeline.md:9-13`, generated from
`docs/wiki-src/bpir.examples.timeline.md:9-13`). Copying it verbatim fails:
`Unmatched '(' in timeline 'FadeIn'` + `Unrecognized instruction:` on the two
following lines. Collapsing the same `timeline` instruction onto ONE line compiles.

The grammar is line-oriented by design (every instruction is one line), so the
right fix routes to the docs, not the parser.

**Workaround:** put the whole `timeline NAME(...) [update -> ..., finished -> ...]`
instruction on a single physical line.
**Fix:** correct the wiki-src overlay `docs/wiki-src/bpir.examples.timeline.md`
so the `timeline` instruction is one line (it regenerates into `wiki-generated/`;
never hand-edit the generated file). Alternatively — larger scope, not
recommended — teach the parser to join a parenthesis-unbalanced line with its
continuation before tokenizing.

## History
- `#1-initial-repro` `OPEN` reporter — Parser is line-oriented (`BpirParser.cpp:455`,`:643`,`:2360-2378`), no paren line-continuation. The documented timeline example (`wiki-generated/bpir.examples.timeline.md:9-13`, src `docs/wiki-src/bpir.examples.timeline.md:9-13`) spans three lines. Verbatim compile via `blueprint.compile_bpir` → `Unmatched '(' in timeline 'FadeIn'`; same instruction on one line compiles. Fix the wiki-src example to a single line.
