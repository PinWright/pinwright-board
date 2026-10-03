---
id: B-dump-root-scaffold-advises-asset-dump-force
title: "The seeded dump-root CLAUDE.md/AGENTS.md tells agents to refresh one asset with `asset.dump` `force=true`, which asset.dump rejects with UNKNOWN_PARAMS"
status: OPEN
severity: Low
category: bug
tags: [asset-dump, scaffold, agent-guidance, docs, unknown-params]
encounters: 1
lastSeen: 2026-10-02T00:00:00Z
rice: [2, 1, 1, 1]
priority: 17
---

# Seeded dump-root guidance names a parameter asset.dump does not accept

`AssetDumpWriter::EnsureDumpRootScaffold` (`Source/PinWright/Private/Utils/AssetDumpWriter.cpp`,
the `AgentOrientationMd` text) seeds `CLAUDE.md` and `AGENTS.md` into every dump root with:

```
- **Refresh:** `asset.dump` with `force=true` for one asset; `asset.dump_folder` for a subtree.
```

`asset.dump` declares only `assetPath`, `outRoot`, `diff` and `includeWidgetScreenshot`
(`Handlers/Asset/AssetDumpHandler.cpp`, its `REGISTER_RPC_HANDLER`), so an agent following the file
gets `UNKNOWN_PARAMS`. The plugin's own `CLAUDE.md` ("Aspect Version Bumping") documents exactly
this: `force` exists on `asset.dump_folder` only, and `asset.dump` never consults the cache, so a
bare `asset.dump` is already the one-asset refresh.

This is the file the dumper hands to every agent that opens a mirror, so the wrong advice is read
first. Seeding is create-if-missing, so existing roots keep the old text until the file is deleted.

**Fix:** change the line to "`asset.dump` (always re-dumps) for one asset; `asset.dump_folder`
(`force=true` to ignore the cache) for a subtree". Optional: a test asserting the seeded text names
no parameter `asset.dump` does not declare.

## History
- `#1-filed` `OPEN` developer — Found while editing the same scaffold text for `B-asset-dump-writes-crlf` / `B-asset-dump-no-source-freshness-stamp`. The seeded `CLAUDE.md`/`AGENTS.md` "Refresh" line says `asset.dump` with `force=true`; `asset.dump`'s parameter list has no `force`, and the plugin `CLAUDE.md` states it is rejected with `UNKNOWN_PARAMS`. Not fixed in that change to keep it scoped. Searched the board for `force=true` / scaffold / `AgentOrientation` tickets: none. Severity Low: an agent gets a clear refusal and retries without the flag, no wrong data.
