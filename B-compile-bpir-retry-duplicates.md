---
id: B-compile-bpir-retry-duplicates
title: "`compile_bpir` RPC timeout-retry duplicates entry nodes"
status: DONE
severity: High
category: bug
tags: []
---

# `compile_bpir` RPC timeout-retry duplicates entry nodes

When `compile_bpir` RPC times out mid-write (common under parallel test load), the server-side compile often completes and commits entry + body nodes to the graph, but the RPC returns `UE RPC request timed out after Xms`. Retrying the same BPIR string — even with `mode: "replace"` — creates **duplicate** entry nodes and body subgraphs, because replace mode's pre-clean step doesn't find the previous partial write if it was stored under a different internal ID or wasn't visible to the replace-scope at that moment.

**Repro (observed this session):**
1. Call `compile_bpir` on `W_ReplaySaveHandler` with `entry custom_event HandleStateChanged(...) {...} entry override Construct() {...}` — times out at ~14.8s.
2. Retry identical call — times out again at ~14.8s.
3. `blueprint_decompile_bpir` shows **2 duplicate `Construct` overrides**, both with `bind_dispatcher` bodies; plus multiple orphan `AddDelegate` / `PrintString` nodes from the failed compiles.
4. `blueprint_graph_find_nodes` returns 5 `AddDelegate` nodes, 5 `PrintString` nodes, 3 `Event Construct` nodes — each retry added another copy.

Related to but distinct from `B-replace-orphan-body` (DONE, about body-node orphans after successful replace) and `B-replace-sig-mismatch` (DONE, about signature-mismatch rejection). This is specifically about **RPC retry semantics** — the server side has no deduplication when the same BPIR submits a second time after a timeout.

**Cleanup cost this session:** ~12 manual `blueprint_graph_delete_node` calls on `W_ReplaySaveHandler` plus ~27 on `W_SingleRaceResultsFrame` / `W_RaceOnlineResultsFrame` to repair duplicate entries and orphans.

**Workaround:** Before every retry: `blueprint_decompile_bpir`, find and manually delete any already-committed entry nodes matching the intended ones.

**Proposal:** Two complementary fixes:
1. **Server-side idempotency**: `compile_bpir` should detect pre-existing entry nodes with identical signature + body hash and short-circuit as `status: "UpToDate"` without re-adding.
2. **Client-visible compile ID**: Return a `compileId` on success; allow retry with that ID to no-op if the prior call already committed.

A simpler floor fix: `mode: replace` should unconditionally clean any entry matching the target signature even if it's present from a prior partial write (not only entries authored by the current compile call).

## History
- `#1-initial-repro` `OPEN` reporter — Hit multiple times during post-restart probing this session. Timeouts are frequent under parallel test load (user confirmed). Each timeout-retry cycle doubled the entries in the target BP. Required lengthy manual node-by-node cleanup to get back to a clean state.
- `#2-upsert-mode-fix` `IN-REVIEW` developer — Root cause: RPC timeout is response-only; handler commits work regardless. Default mode "append" did not pre-clean existing entries, so retries accumulated duplicates. Fix: Phase 0 signature-based cleanup in `BpirCompiler.cpp` (lines ~649-761) now runs unconditionally; default mode is effectively upsert. `mode: "replace"` remains as an alias with identical semantics. Added duplicate-entry warning scan at end of compile to surface remnants of prior partial writes. Test added: `FCompilerIntegrationDefaultModeIdempotentTest` asserts that two consecutive default-mode compiles of identical BPIR produce a single entry node. Docs: `bpir-language-reference.md` mode table updated. Behavior change: callers previously relying on default mode to ERROR on duplicate would no longer see the error — no such callers identified in codebase.
- `#3-verified-idempotent` `DONE` tester — Verified on `W_McpVerifyTemp`. Compiled BPIR `entry override Construct() { call PrintString(InString: "Hello"); }` twice in default append mode. After second compile, `blueprint_graph_find_nodes` returned 1 Event Construct match (the unrelated PreConstruct disabled-stub is always present) and 1 PrintString match. Phase 0 upsert cleaned the first pair before re-emitting. No duplicate accumulation observed.
