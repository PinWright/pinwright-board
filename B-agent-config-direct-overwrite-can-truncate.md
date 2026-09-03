---
id: B-agent-config-direct-overwrite-can-truncate
title: "Agent setup writes merged MCP configuration directly over existing files, so a write failure can destroy unrelated configuration"
status: OPEN
severity: High
category: bug
tags: [setup, config, atomic-write, data-loss, mcp]
encounters: 1
lastSeen: 2026-09-03T23:17:31+03:00
---

# Agent MCP config replacement is not atomic

## What happens

The setup flow reads and merges project agent configuration for `.mcp.json`,
`.gemini/settings.json`, and `.vscode/mcp.json`
(`Setup/AgentMcpConfigurator.cpp:170-190,283-343`), then calls
`FFileHelper::SaveStringToFile` directly on the final path at `:345-350`. The Codex TOML flow
similarly rebuilds only the PinWright table while preserving the rest in memory, then directly
overwrites `.codex/config.toml` at `:466-528`.

UE's `SaveStringToFile` opens a writer on the supplied final filename before it serializes and
only reports I/O/close failure afterwards
(`C:/UE_5.8/Engine/Source/Runtime/Core/Private/Misc/FileHelper.cpp:775-783`). A disk-full,
interrupted, or close failure can therefore leave the existing file truncated or partial. The
function returns “Failed to write”, but unrelated servers and user settings from the old file are
already lost.

## Why it matters

The Install button edits shared project configuration, not a disposable PinWright-only artifact.
A failed setup can break every other configured MCP server or Codex setting. Severity is High for
durable config clobbering.

## What should happen

Write and close a unique same-directory temporary file, validate it, then replace the final file
atomically; delete the temp on failure and leave the old bytes untouched. The existing fix shape is
`Demo/DemoGate.cpp:272-287`, which stages with `SaveArrayToFile` and commits with
`IFileManager::Move(... Replace=true)`. Use a GUID/collision-safe temp name and add an injected
write/move-failure test that proves the original file remains byte-identical.

**Workaround:** Commit or back up agent config files before using Setup > Install.

## Related

- Catalog: `unsafe-output-replacement-or-collision`
- `F-gateway-bearer-token-auth` — touched these generated entries but does not provide atomic replacement for the host config files.

## History
- `#1-direct-config-overwrite` `OPEN` reporter — Traced all JSON-agent and Codex config writes to
  their final paths and compared them with the existing DemoGate atomic-store implementation. No
  setup action or file write was performed.
