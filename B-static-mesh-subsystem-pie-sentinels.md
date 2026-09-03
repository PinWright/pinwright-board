---
id: B-static-mesh-subsystem-pie-sentinels
title: "Through python.execute during another stream's PIE, StaticMeshEditorSubsystem.get_lod_count returns -1 and get_num_uv_channels returns 0 for every mesh — including /Engine/BasicShapes/Cube — with success:true and an empty error log"
status: OPEN
severity: High
category: bug
tags: [python-execute, static-mesh, StaticMeshEditorSubsystem, pie, sentinel, silent-wrong-data, false-success, multi-agent, editor-scripting-guard]
encounters: 1
lastSeen: 2026-09-03T00:00:00Z
---

# Sentinel values dressed up as measurements while PIE is held by another stream

## What was called

Via `python.execute`, in the shared editor, while **another stream held PIE**:

```python
sub = unreal.get_editor_subsystem(unreal.StaticMeshEditorSubsystem)
mesh = unreal.load_asset("/Game/FPS/Weapons/Meshes/SM_WPN_AR")
sub.get_lod_count(mesh)            # -> -1
sub.get_num_uv_channels(mesh, 0)   # -> 0
```

## What happened

`-1` and `0` came back for **every** mesh tried, including a stock engine primitive that cannot
plausibly have zero UV channels:

| asset | `get_lod_count` | `get_num_uv_channels(mesh, 0)` |
|---|---|---|
| `/Game/FPS/Weapons/Meshes/SM_WPN_AR` | `-1` | `0` |
| `/Game/FPS/Weapons/Meshes/SM_WPN_Pistol` | `-1` | `0` |
| `/Engine/BasicShapes/Cube` | `-1` | `0` |

The RPC answered **`success: true`** with an **empty error log**. Nothing in the response, and
nothing in the log, said the values were refusals rather than measurements.

The play state at the time was confirmed independently:

```python
unreal.UnrealEditorSubsystem().get_game_world()   # -> UEDPIE_0_T_UI
```

`get_lod_build_settings` on the same meshes, in the same window, returned **correct data** — so
the mesh handles were valid and the failure is per-binding, not per-asset.

## Why this is High

A caller who does not happen to cross-check against a known-good asset reports "SM_WPN_AR has
0 UV channels" as fact. `0` and `-1` are not obviously-invalid in context: a mesh legitimately
can have one LOD, and a reviewer chasing a lightmap or texel-density question is *expecting* a
small number. This is the rubric's High band verbatim — silent wrong data on a normal path,
where the caller trusts a result that is a lie and builds on it. The reach is the multi-agent
shared editor, where any stream can enter PIE at any moment without the reading stream knowing.

Cross-checking is also not cheap: proving these numbers wrong required loading a *third*,
unrelated engine asset and noticing that it returned the same impossible answer.

## Root cause — GUESS, not source-read

**No source was read for this ticket.** The inference is that these bindings sit behind
`EditorScriptingHelpers::CheckIfInEditorAndPIE`, which fails while PIE is running, and that the
bindings then return their sentinel (`-1` / `0`) instead of raising. `get_lod_build_settings`
returning correct data in the same window fits: it has no such guard. Whoever picks this up
should confirm against the engine's `StaticMeshEditorSubsystem` binding bodies before choosing a
fix — the guard, if it is the guard, is engine-side, which constrains what the plugin can do.

## What was expected

An **error, not a sentinel**, when the editor-scripting guard rejects the call. Concretely, one
of:

- If the guard is reachable from the plugin side: have `python.execute` surface the rejection —
  the call answered `success: false` with a message naming PIE, matching the existing PIE-aware
  error family below.
- If the sentinel is unavoidable because the refusal happens inside an engine binding the plugin
  does not wrap: document the trap on the `python.execute` page and, better, expose the LOD/UV
  counts through a PinWright verb that *can* report the refusal honestly — see the sibling
  `F-static-mesh-uv-channel-readout`, which asks for the UV readout this call was standing in for.

**Workaround used:** cross-check every value against `/Engine/BasicShapes/Cube` (or another
asset whose answer is known) in the same call, and treat a matching impossible answer as
"PIE is running, discard the batch". Confirm play state with
`unreal.UnrealEditorSubsystem().get_game_world()` and re-run once the world name loses its
`UEDPIE_` prefix.

## Related — the PIE false-answer family

Same shape (a verb answers confidently and wrongly while PIE is held by another agent),
different surface; filed separately because the fixes do not overlap:

- `B-asset-exists-duplicate-false-negative-in-pie` (OPEN, High) — `asset.exists` answers
  `exists:false` for **everything** during PIE and `asset.duplicate` blames the path.
  Closest sibling: same "false answer, no mention of PIE" contract failure.
- `B-compile-bpir-edit-lost-during-pie` (OPEN, High) — an edit made during PIE returns
  `compiled:true` and is then silently absent.
- `B-asset-save-pie-failure-reports-pendingflush` (OPEN), `B-model-compile-pie-blocked-save-unattributed` (OPEN)
  — PIE-blocked saves reported as something else.

The count of these argues for a cross-cutting answer (a PIE probe every read path consults, or
a standard "this result may be a PIE refusal" annotation) rather than four independent patches.
Not proposing that here; noting the pattern for whoever works the band.

## History
- `#1-filed` `OPEN` WEAPONS-critic — Measured during a WEAPONS critic review, in the shared editor, while another stream held PIE (`unreal.UnrealEditorSubsystem().get_game_world()` returned `UEDPIE_0_T_UI` at the time). Via `python.execute`: `StaticMeshEditorSubsystem.get_lod_count(mesh)` returned **-1** and `get_num_uv_channels(mesh, 0)` returned **0** for `/Game/FPS/Weapons/Meshes/SM_WPN_AR`, `/Game/FPS/Weapons/Meshes/SM_WPN_Pistol` **and for `/Engine/BasicShapes/Cube`** — an engine primitive that cannot have zero UV channels — with `success: true` and an empty error log. `get_lod_build_settings` on the same meshes in the same window returned correct data, so the handles were valid and the failure is per-binding. Root cause is an explicit GUESS, no source read: `EditorScriptingHelpers::CheckIfInEditorAndPIE` fails during PIE and the bindings return sentinel values rather than erroring, with `get_lod_build_settings` unaffected because it carries no such guard. Impact is the rubric's High band verbatim — a caller who does not cross-check reports "0 UV channels" as fact, and neither `0` nor `-1` looks impossible for a mesh in isolation. Ask: an error, not a sentinel, when the editor-scripting guard rejects the call; if the refusal is unreachable from the plugin side, document the trap on `python.execute` and expose the counts through a verb that can report the refusal (see `F-static-mesh-uv-channel-readout`). Workaround: cross-check every batch against a known-good asset in the same call and discard the batch when the known-good answer is also impossible. Same family as `B-asset-exists-duplicate-false-negative-in-pie` and `B-compile-bpir-edit-lost-during-pie`, different surface.
