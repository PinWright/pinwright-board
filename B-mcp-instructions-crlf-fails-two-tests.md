---
id: B-mcp-instructions-crlf-fails-two-tests
title: "mcp-instructions.md checks out CRLF while both embedded templates are LF, so two tests fail permanently on every Windows checkout"
status: DONE
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
- `#2-eol-pin-already-landed-plus-guard` `IN-REVIEW` developer — Root cause was already fixed by option 1 before this pass: the plugin `.gitattributes` (in the 0.8.0 open-source commit `8748c637`, its only history) carries `mcp-instructions.md text eol=lf`, with a comment explaining the byte-exact contract. Evidence it holds on a Windows-shaped checkout: `git -c core.autocrlf=true -c core.eol=crlf archive HEAD mcp-instructions.md` yields 0 CR (the unpinned `README.md` yields 2004 CR under the same flags); the working file is 794 bytes / 0 CR, matching the 794-char templates; `git check-attr` reports `text: set`, `eol: lf`. Comparisons left strict on purpose (normalising them would hide real CRLF drift in the literals). What was missing: both existing comparisons pass on Linux without the pin, so nothing failed if the pin were dropped. Added `test_canonical_instructions_checkout_is_pinned_lf` (`Content/Python/tests/test_mcp_proxy_editor_start.py`, `ProxyStdioLifecycleTest`): runs `git check-attr eol -- mcp-instructions.md` from the plugin root and asserts `eol: lf`; skips when git is absent or the plugin is not a git checkout (Fab/zip installs). Failure direction: with the pin line removed from `.gitattributes`, `git check-attr` reports `eol: unspecified` and the assertion fails. Passes with the engine Python alongside `test_initialize_includes_pinwright_usage_instructions`. No C++ change; `PinWright.infra.request_core.Initialize` needs no edit once the file checks out LF.
- `#3-verified-linux` `DONE` tester — Fix commit 03bd90a5. Passed non-skipped in run3/full: `PinWright.infra.request_core.Initialize`. Python run3 is OK, including `test_initialize_includes_pinwright_usage_instructions` and the new `ProxyStdioLifecycleTest.test_canonical_instructions_checkout_is_pinned_lf`. That one is not skipped here: the plugin is a git checkout, and `git check-attr eol -- mcp-instructions.md` returns `eol: lf`. Acceptance (option 1, pin with `.gitattributes`): verified at HEAD: plugin `.gitattributes` carries `mcp-instructions.md text eol=lf`, and `git -c core.autocrlf=true -c core.eol=crlf archive HEAD mcp-instructions.md` yields 0 CR, which reproduces the Windows-checkout conversion. The comparisons stay strict. Coverage limit: no clone on a real Windows host was run, so the archive simulation stands in for it.
