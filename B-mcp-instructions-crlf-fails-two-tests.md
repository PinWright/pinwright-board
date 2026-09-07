---
id: B-mcp-instructions-crlf-fails-two-tests
title: "mcp-instructions.md checks out CRLF while both embedded templates are LF, so two tests fail permanently on every Windows checkout"
status: OPEN
severity: Medium
category: bug
tags: [tests, crlf, line-endings, autocrlf, mcp-instructions, windows, false-red]
encounters: 1
costly: 1
lastSeen: 2026-08-13T00:00:00Z
---

# CRLF/LF mismatch fails two tests on every Windows checkout

The canonical `mcp-instructions.md` (plugin root) is checked out with **CRLF** line endings under
the default Windows `core.autocrlf=true`, while the two embedded copies of that same text are
**LF**. Two tests load the file and compare it byte-for-byte against a rendered template, so both
fail — permanently, on a clean tree, with no code change required to reproduce.

## The two failing tests

1. **C++** — `PinWright.infra.request_core.Initialize`
   (`Source/PinWright/Private/Tests/Infra/TestMcpRequestCore.cpp:174`)
   `LoadFileToString(Plugin->GetBaseDir()/"mcp-instructions.md")`, substitutes
   `{{PINWRIGHT_WIKI_DIRECTORY}}`, then `TestEqual(... Instructions, ExpectedInstructions)`.
   Fails with expected/actual that are **textually identical** — the whole diff is `\r`.

2. **Python** — `test_initialize_includes_pinwright_usage_instructions`
   (`Content/Python/tests/test_mcp_proxy_editor_start.py:686`)
   `assertEqual(MCP_INSTRUCTIONS_TEMPLATE, canonical_instructions)`, failing
   `'...flow.\n\nThe wiki is flat:...' != '...flow.\r\n\r\nThe wiki is flat:...'`.

## Measurements

```
mcp-instructions.md      : 808 bytes, CR=14, LF=14      (CRLF on disk)
McpRequestCore.cpp copy  : 794 chars, sha256[:16] 4f3db3e1daccf41b   (LF)
mcp_proxy.py copy        : 794 chars, sha256[:16] 4f3db3e1daccf41b   (LF)
```

808 − 794 = **14**, exactly one CR per line. The C++ and Python copies are byte-identical to each
other, so **the documented cross-copy invariant is intact** — the drift is only file-vs-template.

## This is pre-existing, not caused by recent work (proof)

- `mcp-instructions.md` is **unmodified** in git (`git status --short` reports nothing for it).
- Grepping the `McpRequestCore.cpp` working diff for every distinctive template line
  (`generates and refreshes`, `wiki is flat`, `index.md is the root`, `namespace.method`,
  `Invocation modes`) returns **zero hits** — the template region was never touched.
- Therefore both tests fail identically on a clean checkout.

The Python failure was already known and treated as an accepted baseline. **The C++ twin was not
known**, and it has the same single root cause.

## Impact

The suite can never be fully green on a stock Windows clone, so "2 failures" becomes the normal
state and real regressions hide inside that noise. A contributor reasonably reads the C++ failure as
their own breakage.

**Fix (pick one):**
1. Add `.gitattributes` pinning the file: `mcp-instructions.md text eol=lf` (then re-normalize).
   Cleanest — makes the on-disk bytes match the templates everywhere.
2. Normalize line endings on both sides of both comparisons before asserting
   (`Text.ReplaceInline(TEXT("\r\n"), TEXT("\n"))` / `.replace("\r\n", "\n")`). Keeps the
   byte-identity contract for the two templates while making the file comparison ending-agnostic.

Option 1 is preferred: it fixes the cause rather than the symptom, and keeps the tests strict.

## History
- `#1-initial-repro` `OPEN` reporter — "mcp-instructions.md checks out CRLF (808 bytes, 14 CR) under core.autocrlf=true while both embedded templates are LF (794 chars, identical sha256), so PinWright.infra.request_core.Initialize and test_initialize_includes_pinwright_usage_instructions both fail on any clean Windows checkout. Proven pre-existing: the .md is unmodified in git and the McpRequestCore.cpp diff contains no template lines. The Python half was a known baseline; the C++ twin was not. Fix: pin with .gitattributes 'text eol=lf', or normalize endings on both comparisons."
