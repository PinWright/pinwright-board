---
id: B-macos-port-blockers
title: "Latent macOS port blockers: AtomicFileWriter has no Mac branch (wiki generation and client config writes fail), and the stdio proxy looks for the launcher manifest and Install.ini in the wrong places on Mac"
status: WONTFIX
severity: Low
category: bug
tags: [macos, platform-port, latent, atomic-write, wiki, agent-config, mcp-proxy, engine-discovery, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-28T12:00:00Z
---

# Latent macOS port blockers

PinWright does not target Mac today: every module in `PinWright.uplugin` has
`"PlatformAllowList": ["Win64", "Linux"]` (`:24-60`), so nothing here fails for a current user. These
are the defects that a Mac port would hit at runtime rather than at compile time, recorded so the
port does not rediscover them one crash report at a time (plugin HEAD `71c91649`).

1. **`AtomicFileWriter` has no Mac implementation.**
   `Source/PinWright/Private/Utils/AtomicFileWriter.cpp` branches on `PLATFORM_WINDOWS` and
   `PLATFORM_LINUX` only; the fallback compiles and returns
   "Atomic file publication is supported only on Win64 and Linux." as an `IoError`
   (`:227-229` for staging, `:412-416` for publication). Its callers would therefore fail on Mac
   at runtime, not at build time: `Catalog/WikiDiskGenerator.cpp` (the on-disk wiki that the MCP
   server instructions tell agents to read), `Setup/AgentMcpConfigurator.cpp` (writing MCP client
   config), `Handlers/Debug/TraceExportCore.cpp` and `Handlers/Sequencer/SequencerFbxHandler.cpp`.
   The Linux branch is mostly POSIX (`fsync`, `rename`), but its no-replace publish uses the
   Linux-only `renameat2(..., RENAME_NOREPLACE)` (`:55`); the Mac equivalent is `renamex_np` with
   `RENAME_EXCL`.
2. **Launcher manifest path is wrong on Mac.** `Content/Python/mcp_proxy.py:517-526`
   (`_launcher_manifest_path`) uses `/Users/Shared/Epic/UnrealEngineLauncher/LauncherInstalled.dat`.
   The engine reads `FPlatformProcess::ApplicationSettingsDir()/UnrealEngineLauncher/LauncherInstalled.dat`
   (`Engine/Source/Developer/DesktopPlatform/Private/DesktopPlatformBase.cpp:1572-1574`), and on Apple
   platforms `ApplicationSettingsDir()` is the user's `Library/Application Support/Epic/`
   (`Engine/Source/Runtime/Core/Private/Apple/ApplePlatformProcess.cpp:52-63`). The proxy would not
   resolve launcher-installed engines from the manifest on Mac.
3. **Mac source-build registry is never read.** `_install_ini_path` (`mcp_proxy.py:547-552`)
   returns a path only on Linux. On Mac the engine registers source builds in
   `ApplicationSettingsDir()/UnrealEngine/Install.ini`
   (`Engine/Source/Developer/DesktopPlatform/Private/Mac/DesktopPlatformMac.cpp:514`, `:544`), the
   same file layout as Linux under a different root, so a GUID engine association cannot resolve.

Engines: all (the defects are platform-, not version-specific). Affects Mac only.

**Fix:** add a `PLATFORM_MAC` branch to `AtomicFileWriter.cpp` (share the POSIX parts with the Linux
branch, swap `renameat2` for `renamex_np`), with its test. In `mcp_proxy.py`,
use `~/Library/Application Support/Epic/UnrealEngineLauncher/LauncherInstalled.dat` and
`~/Library/Application Support/Epic/UnrealEngine/Install.ini` on `darwin`. Do this as part of adding
`Mac` to `PlatformAllowList`, not before.

## History
- `#1-mac-port-blockers` `OPEN` reporter - Found in the 2026-09-28 competitive gap analysis and re-verified at plugin HEAD `71c91649` against UE 5.8 engine source: `AtomicFileWriter.cpp` fallback at `:227-229` and `:412-416`; proxy manifest path at `mcp_proxy.py:523-525`; Linux-only `Install.ini` at `:547-552`; engine Mac paths at `DesktopPlatformBase.cpp:1574`, `ApplePlatformProcess.cpp:52-63`, `DesktopPlatformMac.cpp:514`. One ticket for the port because the defects share a trigger. Severity Low: the impact would be a hard blocker on Mac, but no supported platform reaches it (`PlatformAllowList` excludes Mac), so reach is nil until a port starts; the README has no latent class, so tagged `latent` and `platform-port`.
- `#2-stale-sweep-yagni` `WONTFIX` developer — Still accurate at plugin HEAD `10212ee4`: `AtomicFileWriter.cpp` has only Windows and Linux branches, and `mcp_proxy.py` `_launcher_manifest_path` / `_install_ini_path` (`:760-796`) use the paths the ticket names. But every module's `PlatformAllowList` is `["Win64", "Linux"]` (`PinWright.uplugin:24-30`), no Mac port is planned, and the ticket's own Fix says to do this only when Mac is added. Closing as speculative; this history entry stays searchable as the porting checklist if a Mac port starts.
