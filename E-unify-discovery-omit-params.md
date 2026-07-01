---
id: E-unify-discovery-omit-params
title: "Unify discovery on omit-params; drop wiki.get, ?-suffix, and id field"
status: DONE
severity: Medium
category: ergonomic
tags: [protocol, breaking-change, wiki, transport]
---

> Superseded — `/rpc` removed wholesale during MCP migration; the omit-`params` HTTP convention survives in spirit as the omit-`args` shape inside the single `call` MCP tool.

# Unify discovery on omit-params; drop wiki.get, ?-suffix, and id field

The `/rpc` surface carried three overlapping discovery channels and a JSON-RPC-vestige `id` field even though the protocol was never actually JSON-RPC:

1. **`wiki.get` RPC** — markdown wiki for a path, registered handler.
2. **`?`-suffix on method name** — JSON schema for namespace/method, served by `FToolCatalog::HandleDiscoveryRequest` via the `OnDiscoveryReceived` transport delegate.
3. **Bridge synthesis** — `mcp-server/src/index.js` translated "MCP `args` undefined" into a separate `wiki.get` request, making the MCP surface *look* like one tool while the C++ side had no notion of discovery.

The `id` field was parsed (`JsonRpc.h:48`), stored on `FRequest`, threaded through every dispatch lambda, and echoed in every response — pure JSON-RPC vestige, never used.

Reference for the cleaner pattern: Unity-MCP commit `24143823` (`C:\Unity\Unity-MCP\`, `RpcDispatcher.cs:66-90`, `JsonRpc.cs:24`) — single signal: `Params is null` → wiki, `Params == {}` → execute, no id field, positional batch correlation.

**Fix:** Collapse all three channels onto omit-`params` discovery. Body without `params` → `WikiHandler::RenderPage(method)`; body with `params` (even `{}`) → dispatcher. Strip the `id` field end-to-end. Delete `wiki.get`, `OnDiscoveryReceived`, `FToolCatalog::HandleDiscoveryRequest`. Bridge becomes a thin pass-through.

**Breaking change:** Any external client that posts requests with `id` and reads `response.id` needs updating. Any client that uses `wiki.get` or `?`-suffix discovery breaks. Per the plugin's "no legacy" scope, this is acceptable.

## History
- `#1-rework` `IN-REVIEW` developer — Implemented the unification end-to-end:
  - `JsonRpc.h`: dropped `Id` from `FRequest`; added `bHasParams` (set from `Obj->HasField("params")`); `BuildSuccess(Result)` and `BuildError(Code, Message, Data?)` no longer take an id; relaxed the empty-method rejection so `{"method":""}` renders the root wiki.
  - `Transport/RpcTransport.cpp` / `.h`: deleted `OnDiscoveryReceived` delegate, the `?`-suffix branches in single and batch paths, all `Req.Id`/`EntryId`/`ReqId` captures, the `hasDiscoveryHandler` health field; added the single omit-`params` → wiki render branch in both paths via a `BuildWikiResponse` lambda calling `WikiHandler::RenderPage`. Updated the server-active log to "HTTP gateway" instead of "JSON-RPC server". Deleted the now-unused `ParseJsonBodyFromHttpResponse` helper.
  - `Catalog/WikiHandler.cpp` / `.h`: lifted the renderer body out of the deleted `wiki.get` handler into `WikiHandler::RenderPage(path, outMarkdown)`. Removed the `REGISTER_RPC_HANDLER("wiki.get", ...)` registration.
  - `Catalog/ToolCatalog.cpp` / `.h`: deleted `HandleDiscoveryRequest`, `CachedNamespaceCounts`, `CachedSortedNamespaces`. Kept `Initialize` and `WriteMarkdownReference` (the rpc-method-reference auto-doc generator is independent).
  - `EditorAutomationRpcGatewaySubsystem.cpp`: removed the `OnDiscoveryReceived` binding lambda.
  - `Utils/HttpResponseSpill.h` / `.cpp`: dropped the dead `Id` parameter from `BuildSpillReferenceResponse` and the unused `GetResponseId` helper.
  - `mcp-server/src/index.js`: collapsed the `args === undefined` branch; `rpcCall` omits the `params` field when undefined and never emits an `id`; updated `callToolDescription` for the two-mode surface.
  - Tests: deleted `Tests/Infra/TestToolCatalog.cpp` (8 tests for the removed discovery handler); rewrote `Tests/Infra/TestWikiHandler.cpp` to drive `WikiHandler::RenderPage` directly; rewrote `Tests/Infra/TestHttpApi.cpp` (~1020 → ~500 lines, kept the two material-authoring tests verbatim); rewrote `mcp-server/test/test-mcp.js` and `test-error-handling.js` for the new shape. Left `mcp-server/test/test-mcp-http-cli.js` alone — self-skips when the external CLI script is absent.
  - Docs: updated `CLAUDE.md`, `docs/arch.md`, `docs/SCHEMA.md`, `docs/wiki/README.md`, `docs/wiki/wiki.md`, `docs/wiki/sequencer.md`, and `.universal-agent/skills/using-editor-automation/SKILL.md` to describe the new wire shape and remove `wiki.get` / `?`-suffix references.
  - Board files and `rpc-method-reference.generated.md` intentionally untouched — historical record and auto-regenerated, respectively.

  Not done in this pass: no compile, no commit, no live verification. Awaits user-driven build + sweep through curl/MCP to confirm the new wire format works against the live editor.
