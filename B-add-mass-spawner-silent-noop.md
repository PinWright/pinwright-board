---
id: B-add-mass-spawner-silent-noop
title: "ai.add_mass_spawner is a phantom-success stub — returns spawnCount/configPath echo but only marks dirty + saves, adds no component/property"
status: IN-REVIEW
severity: Medium
category: bug
tags: [ai, mass-entity, mass-spawner, add_mass_spawner, stub, silent-noop, success-no-effect]
encounters: 1
lastSeen: 2026-07-02T00:07:12.0762832+03:00
---

# ai.add_mass_spawner returns success but never touches the blueprint

`ai.add_mass_spawner` in `Handlers/AI/AIHandler.cpp` (registered at line 2135)
is a success-returning no-op. After loading the target Blueprint it does
**nothing but mark the package dirty and save**, then builds a result that
echoes the request params (`componentName`, `spawnCount`, `configPath`) back
with a reassuring `message`. No component is added to the SCS, the
`configPath` (Mass Entity Config) is never wired to anything, and the
`spawnCount` is never applied to any property. A caller wiring a crowd
spawner has no RPC-visible signal that nothing happened.

## Guilty source (AIHandler.cpp lines 2156-2179)
```cpp
// Load the Blueprint
FString NormalizedPath, LoadError;
UBlueprint* Blueprint = LoadBlueprintAsset(BlueprintPath, NormalizedPath, LoadError);
if (!Blueprint)
{
    Ctx.SendError(TEXT("NOT_FOUND"), LoadError);
    return true;
}

// Note: MassSpawner is typically an Actor class, not a component.
// This implementation adds metadata indicating spawner configuration.
Blueprint->MarkPackageDirty();
McpSafeAssetSave(Blueprint);

TSharedPtr<FJsonObject> Result = MakeShareable(new FJsonObject());
Result->SetStringField(TEXT("componentName"), ComponentName);
Result->SetStringField(TEXT("blueprintPath"), NormalizedPath);
Result->SetNumberField(TEXT("spawnCount"), SpawnCount);
if (!ConfigPath.IsEmpty())
{
    Result->SetStringField(TEXT("configPath"), ConfigPath);
}
Result->SetStringField(TEXT("message"), TEXT("Mass Spawner configuration added. Note: For high-performance crowd spawning, use AMassSpawner actor directly."));
Ctx.SendSuccess(Result);
```
The comment concedes "This implementation adds metadata indicating spawner
configuration" — but it adds **no** metadata: `ConfigPath`, `SpawnCount`, and
`ComponentName` are read from args and echoed straight into the response,
never stored on the asset. The only side effect on disk is a dirty+save of an
otherwise-unchanged blueprint.

## Why it matters
- A realistic Mass-crowd authoring task ("give BP_VillagerSpawner a Mass
  Spawner wired to MEC_Villager, count 250") calls `ai.add_mass_spawner`,
  gets `{"spawnCount":250,"configPath":".../MEC_Villager","message":"Mass
  Spawner configuration added"}`, and concludes the spawner is wired. It is
  not. The blueprint's SCS still contains only `DefaultSceneRoot`; dropped
  into a level it spawns nothing.
- The default `componentName` "MassSpawner" is itself misleading: engine Mass
  has no `UMassSpawnerComponent` — the spawner is the `AMassSpawner` **actor**
  (Count + EntityTypes[].EntityConfig). So even the promised "component" can
  never exist as a component; `blueprint.scs.get(componentClass=MassSpawner)`
  rejects it with "must derive from UActorComponent".
- Same defect class as the DONE ticket `B-material-stub-handlers-silent-success`
  and the IN-REVIEW `B-input-trigger-modifier-stub-silent-success`.

## Fix options (any of)
1. **Implement it honestly.** Since a Mass spawner is an actor, not a
   component, either (a) reparent/require the blueprint to derive from
   `AMassSpawner` and set `EntityTypes[0].EntityConfig` = the config +
   `Count` = spawnCount on the CDO, or (b) if keeping the actor-Blueprint
   shape, add an actual component and populate it. Either way the config +
   count must land on the asset and survive a cold load.
2. **Make it fail loud** (minimum-safe, matches the accepted resolution of
   the sibling stub tickets): replace `SendSuccess` with
   `SendError("NOT_IMPLEMENTED", "add_mass_spawner is a stub; reparent the
   blueprint to AMassSpawner and set EntityTypes/Count directly")`, and fix
   the summary + wiki accordingly — so callers stop trusting the echo.

## Repro (replayed live at HEAD)
1. `ai.create_mass_entity_config({name:"MEC_ReplayVillager", path:"/Game/ReplayMass"})`
   → `{"configPath":"/Game/ReplayMass/MEC_ReplayVillager","traitCount":0,"message":"Mass Entity Config created"}`.
2. `blueprint.create({name:"BP_ReplaySpawner", parentClass:"Actor", savePath:"/Game/ReplayMass"})`
   → created, `existsAfter:true`.
3. `ai.add_mass_spawner({blueprintPath:"/Game/ReplayMass/BP_ReplaySpawner", configPath:"/Game/ReplayMass/MEC_ReplayVillager", spawnCount:250})`
   → returns `{"componentName":"MassSpawner","blueprintPath":"/Game/ReplayMass/BP_ReplaySpawner","spawnCount":250,"configPath":"/Game/ReplayMass/MEC_ReplayVillager","message":"Mass Spawner configuration added. ..."}` — affirmative success.
4. `blueprint.scs.get({blueprintPath:"/Game/ReplayMass/BP_ReplaySpawner"})`
   → `{"components":[{"name":"DefaultSceneRoot","class":"SceneComponent",...}],"count":1,...}` — SCS unchanged, no spawner component, config/count persisted nowhere.

severity rationale: impact=silent-false-success (caller trusts an affirmative echo of a no-op) × reach=rare (experimental Mass Entity crowd-authoring path, not every-session) -> Medium

## History
- `#1-initial-repro` `OPEN` reporter — Seed-mode Mass-crowd authoring task (seed `ai.create_mass_entity_config`); the seed itself worked, the culprit is the neighbor `ai.add_mass_spawner`. Replayed live at HEAD: created MEC_ReplayVillager + BP_ReplaySpawner, called `ai.add_mass_spawner(configPath, spawnCount=250)` → affirmative success echoing spawnCount/configPath, but `blueprint.scs.get` afterward showed only `DefaultSceneRoot` (count:1) — no component added, config/count persisted nowhere. Confirmed against source: AIHandler.cpp lines 2165-2168 do only `MarkPackageDirty()` + `McpSafeAssetSave()` after load, then echo the args. Same defect class as DONE `B-material-stub-handlers-silent-success` and IN-REVIEW `B-input-trigger-modifier-stub-silent-success`. No prior board file mentions `add_mass_spawner` (ripgrep clean). Minimal honest fix: `SendSuccess` → `SendError("NOT_IMPLEMENTED", ...)`.
- `#2-stub-returns-not-implemented` `IN-REVIEW` developer — Applied Option 2 (the accepted DONE `B-material-stub-handlers-silent-success` / IN-REVIEW `B-input-trigger-modifier-stub-silent-success` pattern). In `Handlers/AI/AIHandler.cpp` the `#if MCP_MASS_AI_HEADERS_AVAILABLE` branch of `ai.add_mass_spawner` no longer loads+dirties+saves the blueprint and echoes the params into a `SendSuccess`; it now returns `SendError("NOT_IMPLEMENTED", …)` whose message steers callers to the real route (reparent to `AMassSpawner` /Script/MassSpawner.MassSpawner, then set `Count` + `EntityTypes[0].EntityConfig` on the CDO via blueprint.set_default/property.set). The now-dead param reads / Load / MarkPackageDirty / result-build were removed; the `#else` UNSUPPORTED_VERSION branch is unchanged. Updated the `REGISTER_RPC_HANDLER` summary from "Configure a Mass Spawner on a blueprint" to a NOT_IMPLEMENTED notice. Left `docs/wiki-src/ai.md` untouched — the wiki is owned by the deliberately-partitioned companion `E-mass-spawner-wiki-no-in-mcp-recipe` (its `TestAiMassSpawnerAuthoringDocs.cpp` pins those overlay strings; the summary change carries none of E's overlay-exclusive tokens, so it does not weaken E's revert detection). Regression test `PinWright.ai.add_mass_spawner.ReturnsNotImplemented` added to `Private/Tests/Gameplay/TestAIHandlers.cpp` invokes the real handler via `InvokeHandlerWithCapture` and asserts `bSuccess == false` and `ErrorCode == "NOT_IMPLEMENTED"` — it fails if the silent-success stub were restored. Not compiled/run here (later phase verifies).
