---
id: B-agent-config-backslash-path-match-fails-on-linux
title: "Agent MCP config detection treats a backslash-spelled interpreter path as a different path on Linux, so two agent_mcp_config tests fail on every Linux run"
status: DONE
severity: Medium
category: bug
tags: [setup, agent-config, mcp, linux, paths, test-failure]
encounters: 1
lastSeen: 2026-10-02T21:14:28-05:00
---

# Backslashed path spelling does not match on Linux in AgentMcpConfigurator::ServerEntryMatches

Commit 34ec8e30 ("Agent config detection compares the whole entry; Cursor via file") added the
following tests. They fail in every full Linux suite run (seen on PinWright 8de8a5a2, rebased on
origin/master 34ec8e30; the files are unchanged since 34ec8e30):

- `PinWright.infra.agent_mcp_config.PathSpellingOnlyDifferenceIsConfigured`: `Expected 'backslashed command and args match' to be true.`
- `PinWright.infra.agent_mcp_config.InstallThenDetectRoundTrips`: `Expected '<file>: backslashed interpreter is still Configured' to be true.` This fails for `.mcp.json`, `.cursor/mcp.json`, `.gemini/settings.json` and `.vscode/mcp.json`.

## Root cause

`Source/PinWright/Private/Setup/AgentMcpConfigurator.cpp:640` and `:658` compare `command` and
`args` with `FPaths::IsSamePath`. That function runs `ConvertRelativePathToFull`, which tests
`FPaths::IsRelative` before normalizing. `IsRelative` treats a path as rooted only when it starts
with `/`, `\\`, or `<drive>:` (`Runtime/Core/Private/Misc/Paths.cpp:1265-1279`). On Linux, the
test's `Reslash(P.Python, "\\")` turns `/sdb-disk/...` into `\sdb-disk\...`. A single leading
backslash counts as relative, so `BaseDir()` is prepended before the slashes are normalized, and
the strings never match. On Windows the path starts with `C:` and the comparison holds.

## Fix direction (applied: the first option)

Pick one:
- Normalize `\` to `/` on both sides before `IsSamePath` in `ServerEntryMatches`. This is the
  product-side fix if backslashed config entries should match on POSIX.
- Gate the backslash assertions under `PLATFORM_WINDOWS`, like the existing case-only block.

## History
- `#1-linux-backslash-relative` `OPEN` reporter (build/test runner, run3) — Full offscreen suite on
  the committed, rebased tree: 5788 started, 5786 ok, 2 fail (these two). Root cause traced to
  `FPaths::IsRelative` with a single leading backslash. No source changed.
- `#2-normalize-before-issamepath` `IN-REVIEW` developer — PinWright commit 7230b41d (on 8de8a5a2):
  `ServerEntryMatches` in `Source/PinWright/Private/Setup/AgentMcpConfigurator.cpp` now runs both
  sides through `FPaths::NormalizeFilename` (`\` -> `/`, plus `~` expansion on Unix) before
  `FPaths::IsSamePath`, for `command` and every `args` item. Windows case-insensitivity is unchanged
  because `IsSamePath` still uses `Stricmp` there. The tests are unchanged and not platform-gated. Scoped
  offscreen run `PinWright.infra.agent_mcp_config` (stdio-proxy transport, so the backslash
  assertions executed): 5 started, 5 ok, 0 fail, 0 skipped, `check_suite_log` COMPLETED_CLEAN.
  `check_test_ids.py` and `check_test_skips.py` are clean.
- `#3-verified-linux` `DONE` tester — Fix commit 7230b41d. Before the fix, `PinWright.infra.agent_mcp_config.InstallThenDetectRoundTrips` and `.PathSpellingOnlyDifferenceIsConfigured` failed in run3/full, as reported. After 7230b41d, the scoped run `agentcfg` (filter `PinWright.infra.agent_mcp_config`) passed 5/5 non-skipped: `InstallThenDetectRoundTrips`, `PathSpellingOnlyDifferenceIsConfigured`, `AnyFieldDifferenceIsOutdated`, `CodexBodyComparison` and `IdenticalEntryIsConfigured`. The backslash assertions executed, and the tests are not platform-gated. Acceptance: `ServerEntryMatches` normalizes `\` to `/` on both sides before `FPaths::IsSamePath` for `command` and every `args` item, so a backslash-spelled interpreter matches on Linux. Coverage limit: Windows was not rerun; case-insensitive matching there is unchanged by construction (`IsSamePath` still uses `Stricmp`).
