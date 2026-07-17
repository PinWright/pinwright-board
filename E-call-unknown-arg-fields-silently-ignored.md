---
id: E-call-unknown-arg-fields-silently-ignored
title: "call tool silently ignores unknown top-level argument fields; errors carry no doc pointer"
status: IN-REVIEW
severity: High
category: ergonomic
tags: [mcp-transport, call-tool, discoverability, error-messages]
encounters: 2
lastSeen: 2026-07-17T10:12:43Z
---

# call tool silently ignores unknown top-level argument fields; errors carry no doc pointer

The single `call` tool routed purely on the presence of `method` and `args`.
Any *other* top-level argument field was silently ignored: the request did not
match the execute shape, fell through the routing to the "no `method`" arm, and
returned the wiki **root** namespace index instead of an error. An agent who
guessed the wrong field name got a wrong-but-plausible result (the root
reference) with no signal that its field was unrecognized.

This is exactly what happened in the field. The host-project `CLAUDE.local.md`
described the tool as `call(path?, args?)`, and the agent-facing `wiki.md` guide
named the field `path` throughout, so an agent naturally passed `path:
"<namespace.method>"`. Because `path` was not `method`, the call silently
returned the root index rather than the intended method page. The agent, seeing
a reference instead of the page it asked for, then made three more `call`
attempts that each failed with `-32602` - and none of those error results ever
named the offending field or pointed at the on-disk doc page that would have
explained the correct shape. One seeded-by-docs wrong guess cascaded into four
blind failures. Recoverable, but only after the agent abandoned the field name
by trial and error.

Two gaps compound here: (1) an unknown field is a silent no-op that produces a
misleading success-shaped result, and (2) error results for a known method carry
no pointer back to the wiki page that documents the tool.

**Fix:** `path` is now accepted as an alias for `method` (conflicting values in
both fields are rejected with `-32602`). Any *other* unknown top-level argument
field is rejected with `-32602` naming the offending field(s), the valid fields
(`method`, `args`), and the wiki root index path - no more silent fall-through
to the root index. The `call` tool description gains a sentence stating the
`path` alias and that any other field is rejected, and the improved
missing-`method` error names the valid fields and the alias. Every ERROR result
whose method is known now carries a doc pointer: a final `Docs: <absolute wiki
page path>` text line plus a top-level `docs: {page, wiki}` field (the exact
method page if on disk, else the nearest parent namespace page by stripping
dotted segments, else `index.md`; omitted entirely when no wiki tree exists on
disk; the structured `docs` field survives the oversize-spill rewrite). The
wiki-discovery `hint` now also states the flat-dotted on-disk page naming scheme
(root index `index.md`, namespace pages `<namespace>.md`, method pages
`<namespace.method>.md`), implementing the layout note proposed in
`E-wiki-page-filenames-dotted-flat-undiscoverable`. Successes are unchanged.

## History
- `#1-initial-repro` `OPEN` reporter - The `call` tool silently ignored unknown top-level argument fields: a request with `path:` (or any non-`method`/`args` field) failed to match the execute shape, fell through to the "no `method`" arm, and returned the wiki root index instead of an error. An agent seeded by `CLAUDE.local.md`'s `call(path?, args?)` and `wiki.md`'s pervasive `path` naming passed `path:` and got the root reference instead of the intended method page, then made three more blind `-32602` failures because no error result named the wrong field or pointed at the on-disk doc page. Fixed: `path` is now an alias for `method` (conflicts rejected `-32602`); any other unknown field is rejected `-32602` naming the offending field(s), the valid fields, and the root index path; error results for known methods carry a `Docs:` text line plus a `docs: {page, wiki}` field; the discovery `hint` states the flat-dotted page naming scheme (implements what `E-wiki-page-filenames-dotted-flat-undiscoverable` proposes). Docs, the stdio proxy descriptor, and the host config updated to match.
- `#2-implemented` `IN-REVIEW` developer - Full fix implemented (uncommitted) in `Transport/McpRequestCore.h/.cpp` (unknown-field rejection, `path` alias fold with conflict rejection, string type check, `ResolveDocPageForMethod` walk-up resolver, `docs` field + `Docs:` text line on error results via a new `WrapToolResult` `Method` param, updated `GWikiDocHint` and `call` descriptor) and `Transport/SocketHttpServer.cpp` (completion lambda captures and forwards `Decision.Method`). Eight new automation tests plus three extended in `Tests/Infra/TestMcpRequestCore.cpp` (`PinWright.infra.request_core.ToolsCall.*`: PathAliasWiki, PathAliasExecution, MethodPathConflict, UnknownArgField, ArgFieldType, ErrorDocs, ErrorDocsWalkUp, ErrorDocsSpillSurvival). Docs updated: plugin `CLAUDE.md`, `docs/wiki-src/mcp-transport.md`, `docs/wiki-src/wiki.md`, `docs/arch.md`, `docs/wiki-src/README.md`, `Content/Python/mcp_proxy.py` descriptor, host `CLAUDE.local.md`. Not yet compiled or run - verification pending (compile + `Automation RunTests PinWright.infra.request_core` + live MCP smoke: path-alias page fetch, bogus-field rejection, args-without-method message, UNKNOWN_ACTION walk-up Docs line).
- `#3-suite-green-committed` `IN-REVIEW` developer - Compile clean and full `Automation RunTests PinWright` suite green on UE 5.6 (3428 passed, 0 failed, no crashes); all 16 `PinWright.infra.request_core.ToolsCall.*` tests pass including the 8 new ones. Committed and pushed as `77856a0a` (plugin repo master). Live MCP smoke against a restarted interactive editor still outstanding - left for the mcp-review verification pass.
- `#4-recurred-fix-absent-from-tree` `IN-REVIEW` reporter - Additional evidence (2026-07-17, editor built from the current plugin tree, HEAD `e4d55a1f` on plugin `master`): the exact predicted failure recurred verbatim. An agent called `call` with `{path: "asset.dump_folder"}` (no `method`, no `args`) and silently got the wiki ROOT index - the unknown `path` field was ignored and the request fell through to the root-index arm at `Transport/McpRequestCore.cpp:353-378`. Three execute attempts then failed: (1) `{path:"asset.dump_folder", args:{folderPath:"/App"}}`, (2) the same with `args:{method:..., params:{...}}`, (3) `{path:..., args:{folderPath:"/App", __use_method_field:true}}` - each returned `MCP error -32602: tools/call to 'call' requires either a non-empty 'method' or no fields.` from the args-present-without-`method` arm at `McpRequestCore.cpp:408-413`; none named the offending `path` field or pointed at a doc page. The agent recovered only by fetching the MCP tool schema out-of-band. Source-verified the IN-REVIEW fix is NOT present in this tree: `McpRequestCore.cpp` routing (lines 348-413) still reads only `method`/`args` with no `path` alias fold, no unknown-top-level-field rejection, and no `ResolveDocPageForMethod`/`docs` pointer; grep for `ResolveDocPageForMethod`/`docs` field/`path` alias in `Transport/` returns nothing; and zero of the 8 named tests (`PathAliasWiki`/`UnknownArgField`/`ErrorDocs`/`MethodPathConflict`/...) exist in `Tests/Infra/TestMcpRequestCore.cpp`. The commit `77856a0a` cited in #3 is not a valid object in this plugin checkout (`git cat-file -t 77856a0a` -> "fatal: Not a valid object name"), and the last commit touching `McpRequestCore.cpp` is `3dea7d61` (SSE transport), predating the fix. Not reopened: the fix is demonstrably absent (not present-yet-ineffective), so recurrence is expected until #3's commit actually lands on this tree; leaving status IN-REVIEW pending the real fix reaching this checkout.
- `#5-priority-bump-high` `IN-REVIEW` maintainer - Ruling: "this should be high priority." Severity Medium -> High. Context: the entry-point tool's docs-vs-schema mismatch cascades into blind failures for every fresh agent session, and #4 showed the claimed fix commit (`77856a0a`) never reached this tree - recovering/re-landing that fix (its 8 tests included) is the action item.
- `#6-fix-arrived-on-origin` `IN-REVIEW` developer - The "lost" fix was never lost: `77856a0a` ("call: alias path->method, reject unknown fields, doc refs on errors") landed on plugin origin/master (pushed from the other clone) and reached this tree via fetch/rebase while pushing `0f632687`. Recovery action item from #5 is resolved; the routing fix + its 8 `ToolsCall.*` tests are now in this checkout's source. Still pending for DONE: rebuild the plugin on this machine and live-smoke the path alias / unknown-field rejection / error `Docs:` pointers over MCP (the current editor binaries predate the fix).
