---
id: E-http-response-spill
title: "Direct HTTP oversized responses need file-reference fallback"
status: DONE
severity: Medium
category: ergonomic
tags: [http, response-size, direct-api, mcp-adapter]
---

# Direct HTTP oversized responses need file-reference fallback

Direct `/rpc` callers can receive responses large enough to be awkward or unusable as inline HTTP bodies. MCP clients already have their own large-output handling, but direct HTTP usage needs a server-side fallback that writes the full JSON-RPC response to a generated artifact and returns a small reference response.

The fallback should only apply to direct HTTP callers. The JS MCP adapter should opt out with an internal, collision-resistant parameter that is stripped before RPC handler validation, so existing handler schemas remain strict for real user params.

## Required behavior

- Responses larger than the configured threshold are written under `Saved/EditorAutomation/HttpResponses/<startup-datetime>/<timestamp>_<guid>.json`.
- The reduced response includes `result.outputTooLong`, `file.path`, `file.contentType`, `file.characters`, and `file.threshold`, preserving the original JSON-RPC `id` when present.
- The threshold is configurable in Project Settings next to the HTTP port, defaulting to 10000 characters.
- Old spill directories are pruned on editor startup when older than 24 hours. There is no total-size cap.
- MCP adapter calls bypass Unreal-side response spilling because MCP clients own large-output behavior.
- The JS adapter recognizes the current C++ text formatter envelope `{ "format": "text", "content": ... }` as well as the legacy `{ "_formatted": true, "text": ... }` shape.

## History

- `#1-initial-request` `OPEN` reporter — Direct HTTP usage needs large-response handling without relying on MCP client behavior. Requested a file-reference fallback for responses over roughly 10k characters, storage in `Saved/EditorAutomation/HttpResponses/<startup-datetime>/`, 24-hour age cleanup with no total cap, configurable threshold in editor settings, no `projectRelativePath`, an internal JS adapter bypass flag, and a fix for the JS/C++ text formatter envelope divergence.
- `#2-http-spill-implementation` `IN-REVIEW` developer — Added `HttpResponseSpill` helper, configurable `HttpResponseSpillThresholdCharacters` setting, `/rpc` transport spill wiring for single, discovery, and per-entry batch responses, startup pruning, internal bypass-param stripping, JS adapter bypass injection, formatter envelope unwrapping, docs updates, and focused helper/JS coverage. Static verification only: `node --check` for MCP server files and `git diff --check`; no Unreal build or live test run.
- `#3-verify-direct-http-spill` `DONE` tester — Verified: direct `POST /rpc` `asset.search` with `query="BP_"`, `limit=500` returned a 433-byte compact response with `result.outputTooLong: true`, `file.path` under `Saved/EditorAutomation/HttpResponses/20260516T152637Z/20260516T153249Z_<guid>.json`, `file.contentType: "application/json"`, `file.characters: 123555`, `file.threshold: 10000`, preserved `id: 1`; the spill file exists on disk at 123555 bytes and contains the original full JSON-RPC `{"result":{"assets":[...]}}` payload.
