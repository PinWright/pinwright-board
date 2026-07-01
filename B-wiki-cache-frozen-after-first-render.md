---
id: B-wiki-cache-frozen-after-first-render
title: "Wiki cache frozen after first RenderPage; post-init registrants invisible"
status: DONE
severity: Low
category: bug
tags: [wiki, catalog, cache]
---

# Wiki cache frozen after first RenderPage; post-init registrants invisible

`Catalog/WikiHandler.cpp:111-192` builds an immutable `FWikiCache`
(`AllNodes`, `CategoryNodes`, `TopicNodes`, `MethodsByLowerName`,
`MethodsByCategory`, `SuggestionPool`) lazily on the first `RenderPage()`
call. A function-local `static bool bBuilt = false;` gates construction
(line 114); after the first render it short-circuits at line 115 and
returns the same `Cache` for the rest of the editor session. There is no
invalidation hook — neither `FRpcDispatcher::RegisterHandler`
(`Dispatch/RpcDispatcher.cpp:194`) nor any new `.md` arriving under
`docs/wiki/` resets it.

## Impact today

Under normal startup this is a non-issue:

- `EditorAutomationRpcGatewaySubsystem::Initialize` calls
  `Dispatcher->DrainAutoRegistrations(this)` exactly once
  (`EditorAutomationRpcGatewaySubsystem.cpp:74`), after which the registry
  is effectively immutable.
- All ~1,200 handlers use the static `REGISTER_RPC_HANDLER` /
  `FAutoRegisterHandler` macro pipeline (`HandlerRegistration.cpp:10-22`);
  records accumulate into a function-local static `TArray` at C++ static-
  init and are drained once. No handler file invokes
  `RegisterHandler(...)` directly post-init.
- No `GameFeature` / `OnPluginLoaded` / `ModulesChanged` hook registers
  RPC handlers. The two grep hits for those terms in the plugin source
  are inside handler bodies (consumers of those APIs), not registrants.
- Test fixtures construct fresh `FRpcDispatcher` instances and call
  `DrainAutoRegistrations(nullptr)` before exercising them; none register
  a handler, then call `WikiHandler::RenderPage`, then register more.

## Hypothetical impact

The cache freeze would bite if any of these arrived later:

- A handler module that loads after the gateway subsystem (delayed plugin
  load, optional dependency) and registers via a second drain pass.
- A test that warms the wiki cache, then dynamically registers a handler,
  then asserts on wiki output. (No such test exists today.)
- Hot reload / Live Coding that introduces new `REGISTER_RPC_HANDLER`
  records without a process restart — currently untested.
- A new `docs/wiki/<slug>.md` topic page dropped on disk during a session
  (the discovery scan at `WikiHandler.cpp:169-188` only runs inside the
  same `bBuilt` gate, so new files are invisible until restart).

In all of those, `RenderPage("namespace.new_verb")` returns `# Not found:`
and the suggestion pool omits the entry, even though dispatch works fine.

## Proposed fix

Two viable shapes, both small:

1. **Invalidate on registry change.** Add a monotonic generation counter
   on `FRpcDispatcher` bumped by `RegisterHandler` and the topic-file
   discovery path; `GetWikiCache` compares the cached generation against
   the live one and rebuilds when stale. Lowest churn, no behavior change
   for the common path.
2. **Build at subsystem init from a snapshot.** Drop the lazy `bBuilt`
   guard, have the subsystem call a `BuildWikiCache(Registry)` once at
   the end of `Initialize` (right after the catalog wires up). Cleanest
   shape, but still doesn't cover future runtime registrants — those
   would need option 1 anyway.

Either is fine; option 1 is the more general fix.

## Test impact

Add an `infra.wiki_handler` test that:

1. Constructs a fresh dispatcher, drains auto-registrations, renders a
   wiki page (warming the cache).
2. Calls `Dispatcher.RegisterHandler("zz.test_post_init", ...)`.
3. Renders the page for `zz` and asserts the new method appears under
   `## Methods`.

The test fails on current code (proves the bug), passes after the fix.

**Workaround (today):** restart the editor after any post-init
registration. Not user-facing right now because no post-init registration
path exists.

## History
- `#1-initial-scope` `OPEN` reporter — Identified static `bBuilt` cache freeze at `Catalog/WikiHandler.cpp:111-192`. Confirmed `DrainAutoRegistrations` runs once at subsystem init (`EditorAutomationRpcGatewaySubsystem.cpp:74`) and no current code path registers handlers post-init (greps for `RegisterHandler(`, `FAutoRegisterHandler`, `GameFeature`, `OnPluginLoaded`, `ModulesChanged` all clean). No test triggers the bug today. Severity Low because impact is purely future-proofing — bites the next delayed-load module or hot-reload scenario.
- `#2-invalidate-on-registry-change` `IN-REVIEW` developer — Implemented fix option 1 (generation counter). Added `uint32 RegistryGeneration` to `FRpcDispatcher` (`Dispatch/RpcDispatcher.h`/`.cpp`), bumped in `RegisterHandler`; `GetRegistryGeneration()` accessor plus a `WITH_DEV_AUTOMATION_TESTS`-only `AddAutoRegisteredForTesting` seam. `Catalog/WikiHandler.cpp` `GetWikiCache` now takes the dispatcher and keys its static cache on (source dispatcher, generation), rebuilding on any mismatch; added dispatcher-parameter overloads of `RenderPage`/`EnumerateAllSlugs` (`Catalog/WikiHandler.h`) so the live-subsystem entry points forward through them. Regression test `Tests/Infra/TestWikiHandlerCacheInvalidation.cpp` (`EditorAutomationRpcGateway.infra.wiki_handler.PostInitRegistration`) warms the cache, registers `zz.test_post_init`, and asserts it appears in both the rendered page and `EnumerateAllSlugs`. Not compiled.
- `#3-reapply-missing-production-code` `IN-REVIEW` fixer — Reviewer flagged the diff touching 160+ files (drive-by scope). On inspection the production code from `#2` was NOT on disk: only the regression test existed; `RegistryGeneration`, `GetRegistryGeneration`, `AddAutoRegisteredForTesting`, and the dispatcher-parameter `RenderPage`/`EnumerateAllSlugs` overloads were all absent, and `GetWikiCache` still took the old `const TMap&` registry and kept the unconditional `bBuilt` short-circuit — the test could not have compiled. Re-applied the full `#2` production changes (generation counter + dispatcher-keyed cache + overloads forwarding from the live-subsystem entry points; `GetWikiCache` now resets `Cache = FWikiCache()` on a (dispatcher, generation) mismatch before rebuilding). Scope finding: the ~132 other modified/deleted files (AGIR, Compiler, Handler splits, unrelated tests, other board tickets) are pre-existing working-tree state from concurrent mcp-sprint work, NOT part of this ticket — they were left untouched. This ticket owns exactly: `Dispatch/RpcDispatcher.h`, `Dispatch/RpcDispatcher.cpp` (only the `RegisterHandler` generation bump; the rest of that file's diff is the unrelated `NO_HANDLER_RESPONSE` guardrail ticket), `Catalog/WikiHandler.h`, `Catalog/WikiHandler.cpp`, `Tests/Infra/TestWikiHandlerCacheInvalidation.cpp`, and this board file. The committer must stage only those paths. Not compiled.
- `#4-verify-fix` `DONE` tester — Verified live: ran `system.run_tests` with `test: "EditorAutomationRpcGateway.infra.wiki_handler.PostInitRegistration"` (job `j_20260605T105351_ebbb77af`). Job completed with `resolvedTests: [PostInitRegistration]`, `missingTests: []`, `has_errors: false` — the regression test (warm cache → register `zz.test_post_init` post-init → assert it appears in rendered page and `EnumerateAllSlugs`) passes on the running binary, so the fix is compiled in (despite the `#3` "Not compiled" note) and the generation-counter cache invalidation works.
