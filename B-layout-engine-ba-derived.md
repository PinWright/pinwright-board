---
id: B-layout-engine-ba-derived
title: "PinWright's BPIR node layout engine self-describes as a port of the proprietary Blueprint Assist formatter and mirrors its names and structure, shipping under MIT without any notice; it must be replaced by PinWright's own implementation"
status: OPEN
severity: High
category: bug
tags: [layout, licensing, bpir, gap-analysis-2026-09-30]
encounters: 1
lastSeen: 2026-09-30T12:00:00Z
---

# PinWright's BPIR node layout engine is BA-derived and must be replaced

**Priority: P0.** PinWright's current Blueprint node layout engine is derived from
the Blueprint Assist (BA) marketplace plugin and must be replaced.

- **The engine.** It lives in `Source/PinWright/Private/Compiler/NodeLayoutEngine.{h,cpp}`
  and `Source/PinWright/Private/Compiler/NodeLayoutParameterFormatter.{h,cpp}`.
  `RunLayoutPass` in `Source/PinWright/Private/Compiler/BpirCompiler.cpp` calls it
  after every BPIR compile or insert.
- **The derivation was self-declared.** Before the reference cleanup, the source
  comments, `docs/bpir-compiler-internals.md` and `docs/lessons.md` described the
  engine as a port of BA's formatter. They cited BA source files and line numbers,
  and `lessons.md` told agents to read BA source before changing layout behaviour.
- **Names and structure match BA.** The engine mirrors BA's names and pass
  structure.
- **BA's licence.** BA is a paid Fab/Marketplace product. Its source headers read
  "All Rights Reserved" and it ships no redistribution licence.
- **PinWright's licence.** PinWright is published under MIT. `THIRD_PARTY_NOTICES.md`
  carries no notice for BA.

A public MIT repository whose code describes itself as a port of proprietary code,
and mirrors that code's names and structure, is a licensing exposure. The
exposure is independent of how well the code works.

## Decision (user, 2026-09-30)

1. **Replace the engine.** The current engine is replaced with PinWright's own
   implementation, `F-graph-layout-core`.
2. **Copying policy for the rewrite (user clarification, 2026-09-30).** Reading
   Blueprint Assist source, and reading PinWright's current layout code, is
   allowed. What is not allowed is direct copying without change: no verbatim or
   near-verbatim code, identifiers or structure. The new engine uses its own names,
   decomposition and algorithm, as specified in `F-graph-layout-core`. It must not
   carry over the current engine's function and type names or its pass structure.
3. **Remove the references.** Every Blueprint Assist reference is removed from the
   plugin repo: source comments, docs, the `lessons.md` instruction, and test
   comments. A separate agent is doing this now, as its own change.

## Acceptance criteria

- `rg -i "blueprint ?assist|BA's|BA formula|fpwong"` over `Plugins/PinWright`
  (source, docs, tests, `wiki-src`) returns nothing.
- The BA-derived files are deleted once `F-graph-layout-core` replaces the call
  site in `BpirCompiler.cpp`. No code path still reaches them.
- The replacement shares no verbatim or near-verbatim code, identifiers or pass
  structure with Blueprint Assist or with the deleted engine. The reviewer confirms
  this in the `IN-REVIEW` entry.

## Severity justification

**High, worked as P0.** The rubric has no licensing class. The impact is legal
exposure on every public release, not a runtime defect, and it outranks the
functional layout work, which depends on this rewrite anyway.

## History
- `#1-initial-report` `OPEN` reporter — Gap analysis 2026-09-30: the BPIR layout engine self-describes as a port of the proprietary Blueprint Assist formatter and mirrors its names and structure, with BA file:line citations in docs and an instruction in lessons.md to read BA source; PinWright is MIT and carries no notice. User decision recorded: replace it with PinWright's own implementation (F-graph-layout-core) under the policy "reading BA source and the current code is fine; no direct copying without change — no verbatim/near-verbatim code, identifiers or structure"; all BA references removed from the plugin repo (separate agent, in progress).
