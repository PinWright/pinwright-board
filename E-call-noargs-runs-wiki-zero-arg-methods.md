---
id: E-call-noargs-runs-wiki-zero-arg-methods
title: "Zero-arg methods need explicit args:{} to execute; omitting args silently returns the wiki page"
status: WONTFIX
severity: Low
category: ergonomic
tags: [transport, discovery, ergonomic, wiki]
---

# Zero-arg methods need explicit args:{} to execute; omitting args silently returns the wiki page

The `call` tool routes purely by argument *presence*, not content:
`McpTransport.cpp:435` sets `bHasArgsField = Arguments->HasField("args")`, and
`:470` (`bHasMethod && !bHasArgsField`) returns the wiki page instead of
dispatching. For a method with **no required params** (e.g.
`recorder.list_sessions`, which declares only `RPC_PARAM_DEF("limit", ...)` at
`RecorderHandler.cpp:56-62`), the natural invocation — method name, no `args` —
hits the discovery arm and returns the doc-page reference. The caller must know
to pass `args: {}` to execute. This is the documented omit-`args` contract
(`E-unify-discovery-omit-params`, DONE) and is unambiguous for methods *with*
required params, but for zero-required-param methods it forces a recurring
double round-trip: intend-to-run, get a doc page, re-issue with `args: {}`.

Session evidence: `call({method:"recorder.list_sessions"})` returned
`{"page":".../wiki-generated/recorder.list_sessions.md", ...}` (matching the
reference shape at `McpTransport.cpp:477-481`); re-issuing with `args:{}`
executed and returned the session list.

**Workaround:** always pass `args: {}` when you intend to execute a zero-arg method.
**Fix:** when the named method has zero required params and is called with no `args`, either execute it directly, or return a one-line hint ("this method takes no required args; pass args:{} to run") instead of the full doc-page reference. The registry already knows each method's required-param set, so the transport can branch on it in the `bHasMethod && !bHasArgsField` arm.

## History
- `#1-initial-repro` `OPEN` reporter — Filed: `call` routes by args-presence (`McpTransport.cpp:435,470`), so a zero-required-param method (`recorder.list_sessions`) invoked without `args` returns its wiki page instead of executing; caller must re-issue with `args:{}`. Verified routing in transport source and that no required params are declared. No existing board entry covers the zero-arg ergonomic gap (`E-unify-discovery-omit-params` is the design rationale for the behavior, not a fix for it).
- `#2-wontfix-doc-note` `WONTFIX` developer — Behavior is by design: the args-presence switch is the discovery/execute boundary and won't change. Mitigated by documenting the requirement on the root wiki index instead — added a line to `WikiHandler::RenderRoot` (`Catalog/WikiHandler.cpp`): "A method that takes no arguments still needs an explicit empty `args: {}` to run - omitting `args` returns the wiki page instead of executing." Surfaces at `call()` after the next editor-startup wiki regen.
- `#3-duplicate-noted` `WONTFIX` developer — `E-no-param-read-needs-empty-args-undocumented` (its `foliage.get_instances` example) closed as a duplicate of this ticket: same args-presence routing, same zero-required-param reader class, same already-shipped root-index mitigation; its only addition (extend the note onto namespace/method overlay pages) it puts out of scope.
- `#3-dup-evidence` `WONTFIX` developer — `E-no-param-read-needs-empty-args-undocumented` (foliage example: `foliage.get_instances`, zero required params at `FoliageHandler.cpp:340-343`) re-filed this same omit-args=doc-fetch trap under a new slug; resolved WONTFIX as a duplicate of this ticket. The root-index mitigation here (`WikiHandler.cpp:369`) already covers it; its only delta was relocating the note onto the per-namespace overlay, which it scoped out to the wiki-authoring process.
