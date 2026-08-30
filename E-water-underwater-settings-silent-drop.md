---
id: E-water-underwater-settings-silent-drop
title: "water.set_water_body_underwater_post_process silently drops unmatched settings keys (ok:true, applied:[])"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [water, set_water_body_underwater_post_process, settings, post-process, silent-noop]
---

# Underwater post-process settings write drops unmatched keys with a clean success

`water.set_water_body_underwater_post_process` accepts a `settings` object
documented only as an "FUnderwaterPostProcessSettings field map" and applies it
"field-by-field via property reflection." The reported signal is an `applied`
array listing which keys took. When a requested key does **not** match a direct
field of `FUnderwaterPostProcessSettings`, the handler silently drops it and
still returns `ok:true` / `is_error:false` with the key absent from `applied` —
no error, no warning, no dropped-fields list.

The trap is that the underwater color/fog/tint knobs a caller actually wants
(`SceneColorTint`, `bOverride_SceneColorTint`, fog/exposure/bloom fields, etc.)
are **valid FPostProcessSettings field names** but they do **not** live directly
on `FUnderwaterPostProcessSettings`. The struct (engine
`WaterBodyComponent.h`) has only `bEnabled`, `Priority`, `BlendRadius`,
`BlendWeight`, and a nested `PostProcessSettings` (FPostProcessSettings) member.
So a caller who passes those real field names flat — the obvious reading of
"FUnderwaterPostProcessSettings field map," and exactly the framing of the task
("bump the underwater fog/tint settings... via the settings field map") — gets
every key silently dropped and a 200 OK with `applied:[]`. The only way to make
the tint stick is to know to nest the keys under a `PostProcessSettings` object,
which the wiki never states.

This is the same misleading-success defect class already filed on this board for
`gas.set_effect_tags` / `gas.set_ability_tags`
(`E-gas-set-effect-tags-drops-unregistered`) and
`ai.configure_slot_behavior` (`B-configure-slot-behavior-ignores-behavior-and-tags`):
a write RPC that drops part of its input must not report unqualified success.

Root cause — `Source/PinWright/Private/Handlers/Water/WaterHandler.cpp`
in the `settings` loop (around lines 413-422):

```cpp
for (const auto& Pair : (*SettingsPtr)->Values)
{
    FProperty* InnerProp = FindPropertyCI(SettingsStruct, ...Pair.Key);
    if (!InnerProp) continue;            // unmatched key evaporates, never reported
    FString ApplyErr;
    if (ApplyJsonValueToProperty(SettingsContainer, InnerProp, Pair.Value, ApplyErr))
        Applied.Add(...);                // only successes recorded; ApplyErr discarded
}
```

Two silent-drop paths: an unmatched key (`continue`) and an apply failure
(`ApplyErr` collected but thrown away). Both leave the key out of `applied` with
no other signal.

**Fix (match the established convention):** Adopt the validate-before-mutate
pattern already landed for the identical class in
`ai.configure_slot_behavior` / the GAS tag-write family — collect every settings
key that does not resolve to a direct field (or whose apply fails, surfacing
`ApplyErr`) into a `droppedSettings` list, and if any are dropped reject the
whole call with `INVALID_PARAMS` carrying that list and a message steering the
caller at the nested `PostProcessSettings` object for FPostProcessSettings knobs.
Cheaper alternative if a hard reject is unwanted: always echo a
`droppedSettings`/`skipped` array alongside `applied` so the drop is visible. On
the all-valid path behavior is unchanged. The wiki overlay for
`water.set_water_body_underwater_post_process` should also state the direct
fields (`bEnabled`/`Priority`/`BlendRadius`/`BlendWeight`) vs. the nested
`PostProcessSettings` object for color/tint/fog.

## Evidence (this task — focus `water.spawn_water_body`)

Replayed against `mcp__editor-automation__call` on a freshly spawned
`WaterBodyRiver`:

- Flat (obvious) form — valid FPostProcessSettings field names passed directly:
  `water.set_water_body_underwater_post_process {actor:"ReplayRiver", settings:{SceneColorTint:{r:0.9,g:0.2,b:0.1,a:1}, bOverride_SceneColorTint:true, FogDensity:0.05}}`
  → `{"applied":[]}` — `ok:true, is_error:false`. All three keys silently dropped.

- Nested (working) form — the keys hidden under `PostProcessSettings`:
  `... settings:{bEnabled:true, BlendWeight:0.9, PostProcessSettings:{bOverride_SceneColorTint:true, SceneColorTint:{r:0.1,g:0.35,b:0.55,a:1}}}`
  → `{"applied":["settings.bEnabled","settings.BlendWeight","settings.PostProcessSettings"]}`;
  `actor.describe` readback confirms the nested `SceneColorTint`
  (R 0.1 / G 0.35 / B 0.55) and `bOverride_SceneColorTint:true` persisted on the
  component's `UnderwaterPostProcessSettings.PostProcessSettings`.

The attempt agent only succeeded because it pre-read engine `WaterBodyComponent.h`
+ `Scene.h` to discover the nested-`PostProcessSettings` shape (verbatim friction:
"the underwater 'fog/tint' settings field names are not documented — the wiki only
says 'FUnderwaterPostProcessSettings field map'... I read the engine
WaterBodyComponent.h and Scene.h to learn the nested PostProcessSettings.SceneColorTint
+ bOverride_SceneColorTint shape"). A caller who reads "field map" literally and
passes the flat names gets a clean `applied:[]` no-op with nothing pointing at the
nesting.

## History
- `#4-fix-validate-before-mutate` `IN-REVIEW` developer — Clean retry of the `#2` intended work (which was never landed). Replaced the silent-drop `settings` loop in `Source/EditorAutomationRpcGateway/Private/Handlers/Water/WaterHandler.cpp` (`water.set_water_body_underwater_post_process`, was lines 413-422) with the established validate-before-mutate convention (matches `ai.configure_slot_behavior` `droppedTags` / the GAS tag-write family `MakeDroppedTagsErrData`). Pass 1 pre-scans every `settings` key against the direct fields of `FUnderwaterPostProcessSettings`; any key that does not resolve is collected into a `droppedSettings` array of `{key, error}` objects (the `error` steers the caller at the nested `PostProcessSettings` object for FPostProcessSettings color/tint/fog knobs) and the whole call is rejected with `INVALID_PARAMS` before mutating anything. Pass 2 applies the now-all-resolved keys and, on an apply failure, surfaces the previously-discarded `ApplyErr` as a `droppedSettings` `{key,error}` entry + `INVALID_PARAMS` instead of dropping it. The all-valid path is unchanged (`applied:[...]` success). Wiki overlay `docs/wiki-src/water.md` updated: the step-5 summary now names the four direct fields vs. the nested `PostProcessSettings` object and the reject-on-unmatched behavior, and a new `### water.set_water_body_underwater_post_process` H3 documents the direct-vs-nested contract with a flat-fails / nested-works JSON example. Regression test added in `Source/EditorAutomationRpcGateway/Private/Tests/World/TestWaterHandlers.cpp` — `FWaterUnderwaterSettingsSilentDropTest` (`...RejectsUnmatchedSettingsKeys`, MCP_HAS_WATER-gated) spawns a lake, calls the handler with flat `bOverride_SceneColorTint` + `FogDensity`, and asserts `bSuccess==false`, `ErrorCode==INVALID_PARAMS`, and that `droppedSettings` lists both keys; reverting the pre-scan to the old `if (!InnerProp) continue;` silent drop flips the response back to an empty-`applied` success and fails the test. Not yet compiled/run (later phase).
- `#3-attempt-failed` `OPEN` developer — fuzz3 fix-workflow run hit the Anthropic session limit mid-pipeline (after Implement, before Simplify/Test). The orchestrator's own rollback step also failed on the same limit, so the cleanup never ran. The `#2` changes were **NEVER compiled, tested, or pushed to origin** — the uncommitted diff was discarded (`git reset --hard` on the plugin clone; backed up to `X:\src\unreal\E-water-underwater-attempt-20260622.patch`) and the ticket returned to `OPEN` for a clean retry. Treat the `#2` entry below as *intended* work that is not in the repo.
- `#2-fix-validate-before-mutate` `IN-REVIEW` developer — Replaced the silent-drop `settings` loop in `Source/EditorAutomationRpcGateway/Private/Handlers/Water/WaterHandler.cpp` (`water.set_water_body_underwater_post_process`) with the established validate-before-mutate convention (matches `ai.configure_slot_behavior` `droppedTags` / the GAS tag-write family). Pass 1 pre-scans every `settings` key against the `FUnderwaterPostProcessSettings` direct fields and, if any do not resolve, rejects the whole call with `INVALID_PARAMS` + a `droppedSettings` array (message steers the caller at the nested `PostProcessSettings` object for the FPostProcessSettings color/tint/fog knobs) before mutating anything. Pass 2 applies the now-all-resolved keys and, on an apply failure, surfaces the previously-discarded `ApplyErr` as a `droppedSettings` `{key,error}` entry + `INVALID_PARAMS` instead of dropping it. The all-valid path is unchanged (still `applied:[...]` success). Wiki overlay `docs/wiki-src/water.md` updated: line-19 summary now points at the per-method page, and a new `### water.set_water_body_underwater_post_process` H3 documents the four direct fields vs. the nested `PostProcessSettings` object (with a flat-fails/nested-works example). Regression test added in `Source/EditorAutomationRpcGateway/Private/Tests/World/TestWaterHandlers.cpp` — `FWaterUnderwaterSettingsSilentDropTest` (`...RejectsUnmatchedSettingsKeys`) spawns a lake, calls the handler with flat `bOverride_SceneColorTint`/`FogDensity`, and asserts `bSuccess==false`, `ErrorCode==INVALID_PARAMS`, and `droppedSettings` lists both keys; reverting to the `continue` silent-drop flips the response back to an empty-`applied` success and fails the test. Not yet compiled/run (later phase).
- `#1-initial-repro` `OPEN` reporter — Process audit of the river-valley water task (focus `water.spawn_water_body`). `water.set_water_body_underwater_post_process` returns `ok:true` with `applied:[]` when given valid FPostProcessSettings field names (`SceneColorTint`, `bOverride_SceneColorTint`, `FogDensity`) flat in `settings`, silently dropping all of them — those fields live on the nested `PostProcessSettings` member of `FUnderwaterPostProcessSettings`, not directly on it, and the wiki only says "FUnderwaterPostProcessSettings field map." Confirmed by replay: flat form → `applied:[]`; nested `PostProcessSettings:{...}` form → applied and persisted (verified via actor.describe). Root cause WaterHandler.cpp:416 `if (!InnerProp) continue;` (unmatched key evaporates) + line 418 (apply failure discards `ApplyErr`). Same misleading-success class as `E-gas-set-effect-tags-drops-unregistered` / `B-configure-slot-behavior-ignores-behavior-and-tags`; proposes the same validate-before-mutate `INVALID_PARAMS`+`droppedSettings` remedy (or at minimum echo a `droppedSettings` array) plus a wiki note on the nested-`PostProcessSettings` shape.
- `#5-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place; every rewritten path was confirmed to exist at plugin HEAD `ef8a1f1b`. 4 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
