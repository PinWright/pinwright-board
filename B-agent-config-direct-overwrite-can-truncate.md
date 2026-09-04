---
id: B-agent-config-direct-overwrite-can-truncate
title: "Agent setup writes merged MCP configuration directly over existing files, so a write failure can destroy unrelated configuration"
status: IN-REVIEW
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

## Fix

The root cause was writing rebuilt shared configuration directly to the destination: Unreal opens
that file destructively before the write is known to have succeeded. The shared atomic writer now
creates a collision-safe GUID sibling with Win32 `CREATE_NEW`, fully writes and flushes it, closes
it, verifies its exact bytes in fixed-size chunks, and first publishes with a same-directory,
non-replacing `MoveFileExW`. `FailIfExists` returns on a destination collision; `ReplaceExisting`
uses `MoveFileExW` with `MOVEFILE_REPLACE_EXISTING | MOVEFILE_WRITE_THROUGH` only after that
collision. It re-stats the destination immediately before replacement; if the destination vanished,
it retries the non-replacing publication and reports `bReplaced=false` when that creation succeeds.
Only a successful existing-destination replacement reports `bReplaced=true`. Neither path deletes
the destination before publishing. Both JSON and Codex TOML setup paths use the shared
UTF-8-without-BOM entry point while preserving their existing messages.

Changed files:

- `Source/PinWright/Public/Utils/AtomicFileWriter.h`
- `Source/PinWright/Private/Utils/AtomicFileWriter.cpp`
- `Source/PinWright/Private/Utils/AtomicFileWriterInternal.h`
- `Source/PinWright/Private/Setup/AgentMcpConfigurator.cpp`
- `Source/PinWright/Private/Tests/Utility/TestAtomicFileWriter.cpp`
- `X:\src\unreal\.pinwright-board\B-agent-config-direct-overwrite-can-truncate.md`

Regression test: `PinWright.utils.atomic_file_writer.FailurePreservesOriginal`
(`FAtomicFileWriterFailurePreservesOriginalTest`). It covers partial-stage and publish failures,
original-byte preservation, sibling uniqueness and cleanup, close-before-publish, normal final
attributes, UTF-8 without a BOM including empty content, and raw-binary success. The publish-failure
case locks the staged source without delete sharing and invokes the real production publisher.

Deliberately not changed: wiki sources, mesh export, `AssetDumpWriter`, `DemoGate`, `ErrorCodes`,
`Build.cs`, compiled skills, and the configurator's existing output encoding and user-facing
messages.

## History
- `#1-direct-config-overwrite` `OPEN` reporter — Traced all JSON-agent and Codex config writes to
  their final paths and compared them with the existing DemoGate atomic-store implementation. No
  setup action or file write was performed.
- `#2-atomic-config-write` `IN-REVIEW` developer — Added the shared verified atomic writer, routed
  both agent configuration formats through it, and added failure-preservation and raw-byte
  regression coverage. Build and automation execution were deliberately deferred by task scope.
