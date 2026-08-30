---
id: B-unknown-params-error-suggests-deleted-question-mark-suffix
title: "UNKNOWN_PARAMS error suggests deleted `?`-suffix discovery form"
status: DONE
severity: Low
category: bug
tags: [dispatcher, error-messages, stale-doc, discovery]
---

# UNKNOWN_PARAMS error suggests deleted `?`-suffix discovery form

`FRpcDispatcher` still tells callers to use the `?`-suffix discovery form that
`E-unify-discovery-omit-params` deleted. When a handler is invoked with an
unknown parameter, the `UNKNOWN_PARAMS` error message reads:

> `Unknown parameter(s) for '<method>': [...]. Valid parameters: [...]. Call "<method>?" for full help.`

Source: `Source/PinWright/Private/Dispatch/RpcDispatcher.cpp:190-196`.

There is no `?`-suffix routing anymore — the transport / dispatcher no longer
strip a trailing `?`, and `Catalog/ToolCatalog::HandleDiscoveryRequest` is gone
(removed in `#1-rework` of `E-unify-discovery-omit-params`). Following the
error's advice produces `[UNKNOWN_ACTION] Unknown action: <method>?` instead of
a schema. The correct discovery form on the current MCP surface is to pass
`method: "<path>"` with `args` omitted; that returns the wiki page that lists
parameters.

This session was misled by the message: an agent saw `?`-suffix in the error
text, invoked `mcp__editor-automation__call` with `path="widget.remove_widget?"`
and `args={}`, and received `[UNKNOWN_ACTION] Unknown action:
widget.remove_widget?`. Calling with `path="widget.remove_widget"` and no
`args` returned the wiki page as expected.

**Fix:** update the format string at `RpcDispatcher.cpp:112` to point at the
current omit-`args` form, e.g. `Call this method again with no args to fetch
its wiki page.` Audit other handler error messages for the same stale phrasing.

## History
- `#1-report` `OPEN` reporter — Filed: dispatcher's UNKNOWN_PARAMS error still
  recommends the deleted `?`-suffix discovery form. Repro: invoke any handler
  with an unknown param; observe the message at `RpcDispatcher.cpp:112`.
  Related: `E-unify-discovery-omit-params` (DONE) deleted the routing this
  message still advertises.
- `#2-current-guidance-regression` `IN-REVIEW` developer — `RpcDispatcher.cpp` already emits the current omit-`args` wiki-discovery guidance for `UNKNOWN_PARAMS`; added `FDispatcherUnknownParamsDiscoveryGuidanceTest` in `Tests/Infra/TestDispatcher.cpp` to assert `UNKNOWN_PARAMS` mentions calling the method with no `args` field and contains no stale `?` suffix. Audited source/wiki strings for the old `Call "<method>?" for full help` phrasing; none remain outside historical board text.
- `#3-skip-mcp-tool-unavailable` `SKIP` tester — Could not exercise the fix: `mcp__editor-automation__call` is not available in this session (absent from the deferred-tool list; `ToolSearch select:mcp__editor-automation__call` and keyword searches all returned no match), so no live `UNKNOWN_PARAMS` error could be triggered to inspect the message string. The fix surface is a runtime error-message string, not file/doc state, and source review alone is never PASS — left `IN-REVIEW` for a re-verify once the MCP editor tool is reachable.
- `#4-verify-fix` `DONE` tester — Verified: invoked `actor.list` with `args={bogusUnknownParamXyz:1}`; live response was `[UNKNOWN_PARAMS] Unknown parameter(s) for 'actor.list': [bogusUnknownParamXyz]. Valid parameters: [filter, world]. Call 'actor.list' with no 'args' field to fetch its wiki page.` The message points at the omit-`args` discovery form, correctly interpolates the method name, and contains no `?`-suffix or stale `Call "<method>?" for full help` phrasing.
- `#5-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place and verified against plugin HEAD `ef8a1f1b`. **Not just a path — the message this ticket reports no longer exists.** `:112` is now the alias type-map builder. The `UNKNOWN_PARAMS` message is built at `:190-196` (declared-param variant `:193`, `SendError` `:196`) and both variants now read “Call '%s' with no 'args' field to fetch its wiki page.” — the `?`-suffix advice this DONE ticket flagged is gone. Sweep-wide record, including every case that could not be repointed: `E-module-rename-citation-sweep`.
