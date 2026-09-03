---
id: F-editor-start-map-parameter
title: "editor_start cannot launch into a map, and there is no restart-into-map primitive — the only reliable route to open a specific level was outside PinWright entirely"
status: IN-REVIEW
severity: High
category: feature
tags: [proxy, mcp-proxy, editor-launch, startup-map, editor_start, editor_restart, level-load, map-swap]
encounters: 1
lastSeen: 2026-08-14T13:13:00+05:00
---

# `editor_start` cannot launch into a map; no restart-into-map primitive

## What was missing

For a plugin whose purpose is driving the Unreal Editor, there was no reliable way
to open a specific map through its own API:

- `level.load` / `editor.open_level` swap the live world, which on a large map
  reliably crashed the editor (`B-open-asset-world-map-load-crash`).
- `editor_start` had no map parameter, and could not have grown one by accident:
  it always emits `-AutoDeclinePackageRecovery` at argv index 2, so a map routed
  through `extra_args` lands *after* the switches, where the engine never looks.

The engine reads the startup map as the **first token of the remaining command
line** and abandons the load outright if that token begins with `-`
(`Engine/Source/Editor/UnrealEd/Private/UnrealEdMisc.cpp:396-399`):

```cpp
FString ParsedMapName;
if ( FParse::Token(ParsedCmdLine, ParsedMapName, false) &&
     // If it's not a parameter
     ParsedMapName.StartsWith( TEXT("-") ) == false )
{
    FString InitialMapLongPackageName = FindMapFileFromPartialName(ParsedMapName);
```

`FindMapFileFromPartialName` resolves through `FPackageName::SearchForPackageOnDisk`
(`UnrealEdMisc.cpp:681`), so a long package name, a bare short name, or a `.umap`
path all work — but only from that one position. So the working route was to launch
`UnrealEditor.exe <project>.uproject /Game/Maps/X` by hand, entirely outside
PinWright.

Failure is also **silent**: a mispositioned or `-`-prefixed token is skipped and the
editor boots the default startup map with nothing logged.

## What shipped

`Content/Python/mcp_proxy.py`:

- `build_editor_command(..., map_name=None)` places the map token at argv index 2 —
  immediately after the `.uproject`, before `_ALWAYS_FLAGS`. Keyword-defaulted, so
  the existing 4-positional call sites and the exact-list assertions in
  `BuildEditorCommandTest` are unchanged.
- `normalize_start_map(value)` — pure validator/normalizer. Trims quotes/whitespace,
  strips a `.umap` suffix, and rejects empty, whitespace-bearing and `-`-prefixed
  tokens with `INVALID_MAP` before anything is spawned, because those are exactly
  the values the engine would ignore silently.
- `editor_start` gains `map`. It forces a direct spawn (the OS `open` verb carries no
  command line) and defaults `unattended_script` to `true` for that spawn — see
  `B-editor-start-blocks-on-zenserver-modal` for why. Keyed on the new parameter, so
  every existing call shape is byte-identical.
- New proxy-local `editor_restart` tool: `editor.quit` → wait for the endpoint to go
  down (`_EDITOR_STOP_TIMEOUT`, 120 s) → normal start path, optionally with `map`.
  It is one verb rather than a documented recipe because the seam is not callable
  from outside: `_editor_process_guard` returns `EDITOR_ALREADY_RUNNING` for the whole
  shutdown window, so a hand-sequenced quit + start races it. It never forces
  anything — `EDITOR_QUIT_REFUSED` relays a dirty-editor refusal verbatim,
  `EDITOR_STOP_TIMEOUT` reports a stuck shutdown with no kill and no respawn, and a
  modal-blocked / unresponsive / still-starting editor is reported rather than
  bypassed (`editor.quit` runs on the game thread those states own).

Tests (`Content/Python/tests/test_mcp_proxy_editor_start.py`): `NormalizeStartMapTest`
(8), `ProxyEditorRestartTest` (11), plus 7 added to `BuildEditorCommandTest` /
`ProxyEditorStartTest`. The handler-level ones drive the real `_editor_start` with
only `subprocess.Popen` mocked and assert on `popen.call_args.args[0]`, i.e. the argv
the production builder produced. Suite: 140 tests green under
`uv run python -m unittest discover -s tests -p "test_*.py"`.

Counterfactual verified by temporarily moving the map append below `_ALWAYS_FLAGS`:
`test_map_lands_immediately_after_the_uproject_on_the_real_launch` fails with
`'-AutoDeclinePackageRecovery' != '/Game/Maps/Dota2_Blockout'`.

Docs: `Docs/wiki-src/unattended.md` gains a `## Starting into a specific map`
section; `Docs/arch.md` gains an `### editor_restart` section and a map paragraph
under `### editor_start`; `README.md` updated from three proxy tools to four.

## Not verified

No runtime verification — the editor was owned by another agent for the duration.
Needs a live pass: `editor_start {map}` from cold, `editor_restart {map}` against a
running editor, and the dirty-editor refusal.

## See also

- `B-open-asset-world-map-load-crash` — the in-place swap crash that made launching
  into a map the only reliable route; fixed separately in the same change.
- `B-editor-start-blocks-on-zenserver-modal` — why `map` defaults modal suppression on.
- `F-proxy-editor-start-rpc` — the ticket that introduced `editor_start` itself.

## History
- `#1-no-map-parameter` `OPEN` reporter — Filed after a session in which the only way to open `/Game/Maps/Dota2_Blockout` was a hand-rolled `UnrealEditor.exe <project> <map>` command line outside PinWright: `level.load` crashed the editor, and `editor_start` could not place a map token in the one position the engine reads it from because it always prepends its own switches. Requested a map parameter plus a restart-into-map primitive, on the grounds that a fresh process — not an in-place world swap — is the honest primitive for "open this map".
- `#2-map-and-restart-shipped` `IN-REVIEW` developer — Added `map` to `editor_start` (placed at argv index 2 by `build_editor_command`, validated by the new pure `normalize_start_map`, forcing a direct spawn and defaulting `unattended_script` on), and a new proxy-local `editor_restart` tool that quits via `editor.quit`, waits out the shutdown window `editor_start`'s own guard would otherwise reject, then starts fresh. 26 new Python unit tests; full suite 140 green; the argv-position test confirmed to fail when the map append is moved below the switches. Docs updated in `Docs/wiki-src/unattended.md`, `Docs/arch.md`, `README.md`. Not gate-verified live — the editor was owned by another agent, so `editor_start {map}`, `editor_restart {map}` and the dirty-editor refusal still need a runtime pass.
- `#3-argv-position-verified-live` `IN-REVIEW` tester — PARTIAL runtime verification, and it covers the load-bearing half. Shipped as commit `2586498b` on `master` (pushed). Launched the editor from a shell in exactly the argv shape `build_editor_command` emits — `UnrealEditor.exe X:\src\unreal\EAContentExamples58\EAContentExamples58.uproject /Game/Maps/ExampleProjectWelcome -AutoDeclinePackageRecovery`, map token at index 2, before every switch — on UE 5.8. The editor booted straight into that map: `level.get_info` immediately after readiness returned `{"levelPath":"/Game/Maps/ExampleProjectWelcome",...}`. So the engine's "first token of the remaining command line" read (`UnrealEdMisc.cpp:396-399`) behaves as the fix assumes, and the position is right. Python suite re-run green at 140/140 (98 -> 126 in `test_mcp_proxy_editor_start.py`, 14 unchanged in `test_mcp_proxy_sse.py`, nothing removed). STILL UNVERIFIED, and deliberately so — these were not exercised and no claim is made about them: the `editor_start {map}` tool path itself (the launch above was a shell launch, not a proxy call, so `normalize_start_map`, the forced direct spawn, and the `unattended_script` keyed default are covered only by unit tests); `editor_start {map:"-bogus"}` -> `INVALID_MAP` with no spawn; every `editor_restart` path including the dirty-editor `EDITOR_QUIT_REFUSED` refusal and `{save:true}`; and the ZenServer long-wait modal (unreproducible on demand — Zen was healthy for this pass, and it was Zen being DOWN during an earlier automation run, 138 x `Failed to connect to localhost port 8558`, that starved `FAutomationControllerManager::Tick` by 53.9 s and aborted that run early, which is incidental evidence that the hazard this ticket cites is real).
