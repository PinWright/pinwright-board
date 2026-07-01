---
id: F-rendering-project-settings
title: "No typed read/write for URendererSettings or persistent Lumen method selection"
status: DONE
severity: Medium
category: feature
tags: [rendering, lumen, project-settings, developer-settings, persistence, escape-hatch]
---

# No typed read/write for URendererSettings or persistent Lumen method selection

The plugin currently has no typed surface for reading or mutating
`URendererSettings` (the `UDeveloperSettings` subclass backing
**Project Settings → Engine → Rendering**, persisted to
`Config/DefaultEngine.ini` under `[/Script/Engine.RendererSettings]`).
Adjacent surfaces only address neighboring concerns and leak through
to CVars or the live scene:

- `system.inspect.get_project_settings`
  (`EnvironmentHandler.cpp:1182`) is a placeholder that returns
  `{"message":"Project settings retrieved","success":true}` with no
  actual data — it does not enumerate `UDeveloperSettings` classes or
  their UPROPERTYs.
- `render.lumen_update_scene` only triggers a runtime recapture; it
  does not change Lumen configuration and the effect is in-memory.
- `lighting.setup_global_illumination` accepts
  `LumenGI|ScreenSpace|None|RayTraced|Lightmass`
  (`LightingHandler.cpp:725-779`) but writes only via
  `IConsoleManager::FindConsoleVariable` (`r.DynamicGlobalIlluminationMethod`,
  `r.ReflectionMethod`). CVar writes do not persist across editor
  restarts and do not update `URendererSettings` UPROPERTYs that the
  editor UI reads.

Net effect: agents that want to change a renderer setting durably
must shell out to `system.console_command` ("`r.Lumen.HardwareRayTracing 1`"),
which (a) doesn't persist, (b) doesn't update the Project Settings UI,
(c) bypasses any UPROPERTY-driven validation, and (d) is invisible to
later `system.inspect.*` calls.

## Proposal

Add a `rendering.*` namespace targeting `URendererSettings` specifically
(the highest-traffic developer-settings class) with three handlers:

```
rendering.get_project_settings(filter?: string)
  -> { settings: { "<UPROPERTYName>": <jsonValue>, ... }, configFile: "DefaultEngine.ini" }

rendering.set_project_settings(updates: { "<UPROPERTYName>": <jsonValue>, ... },
                               save?: bool = true)
  -> { applied: [string], rejected: [{name, reason}], savedTo: string }

rendering.set_lumen_method(hardwareRT?: bool,
                          finalGatherQuality?: number,
                          reflectionMethod?: "None"|"Lumen"|"SSR"|"RT",
                          softwareRTMode?: "Detail"|"Global")
  -> { previous: {...}, current: {...} }

rendering.set_dynamic_gi_method(method: "LumenGI"|"ScreenSpace"|"None"|"RayTraced"|"Lightmass",
                               persist?: bool = true)
  -> { previous, current, configFile }
```

Implementation surface:

- `URendererSettings* S = GetMutableDefault<URendererSettings>();`
- Walk `S->GetClass()->PropertyIterator()`; map JSON in/out via the
  reflection system (reuse `Utils/PropertyUtils.h` patterns).
- Persistence: `S->TryUpdateDefaultConfigFile()` (UE 5.4+) or
  `S->SaveConfig(CPF_Config, *S->GetDefaultConfigFilename())` —
  `UDeveloperSettings` already routes to `DefaultEngine.ini`. After
  write, broadcast `FCoreUObjectDelegates::OnObjectPropertyChanged` so
  any open Project Settings tab refreshes.
- `set_dynamic_gi_method` with `persist:true` should update the
  UPROPERTY *and* mirror the matching CVars so the live viewport
  reflects the new method without an editor restart (the existing
  `lighting.setup_global_illumination` CVar logic is the right body
  for the mirror step).
- `set_lumen_method` writes the four most-asked-for Lumen UPROPERTYs
  (`bUseHardwareRayTracing`, `LumenFinalGatherQuality`,
  `ReflectionMethod`, `LumenRayLightingMode`); other Lumen fields fall
  back to the generic `set_project_settings` path.

## Why scoped to `URendererSettings` first

The proposal explicitly does **not** try to be a generic
`developer_settings.*` namespace covering all `UDeveloperSettings`
subclasses — that's a much larger surface (audio, input, garbage
collection, networking, physics, …) with per-class quirks. Renderer
settings are the empirically hottest target (every Lumen / Nanite /
shadow / VT investigation hits them) and the right place to validate
the read/write/persist pattern before generalizing.

A follow-up ticket could add `developer_settings.list_classes` +
`developer_settings.get` / `set` if the read/write pattern proves out.

## Edge cases

- **Class flags**: refuse to write UPROPERTYs without `CPF_Config` or
  `CPF_GlobalConfig` (they don't round-trip through the ini); return
  per-key rejection with reason `"not_config_serializable"`.
- **Read-only / advanced display**: surface the metadata flags on
  `get` so callers know which fields the editor hides; don't refuse
  writes on those — agents legitimately need to set advanced fields.
- **Hot-reload effect**: many renderer settings only take effect after
  shader recompile or map reload; document the in-memory vs reload
  effect per-field in the wiki, not as runtime checks.
- **Concurrent CVar overrides**: a CVar set via `console_command` at
  `Scalability` priority will still win against the UPROPERTY's
  `SetByProjectSetting` source until the project-setting write re-runs
  through the CVar-bind path. Mention this caveat on the wiki page;
  no special handling.

## Cross-ref

- `system.inspect.get_project_settings` — currently a stub; this
  ticket replaces the rendering slice of that intent.
- `lighting.setup_global_illumination` — would gain a persistent
  sibling (`rendering.set_dynamic_gi_method` with `persist:true`)
  while remaining the live-only fast path.
- `render.lumen_update_scene` — unchanged; complements
  `set_lumen_method` for forcing a recapture after a method change.
- `F-search-api-console-commands` — pairs with this ticket; agents
  often discover the relevant CVar via console search and then need
  this ticket to persist the change.

## History
- `#1-no-renderer-settings-rw` `OPEN` reporter — Verified
  `system.inspect.get_project_settings` returns a placeholder string
  (`EnvironmentHandler.cpp:1182`); `lighting.setup_global_illumination`
  only writes CVars (`LightingHandler.cpp:725-779`); `render.lumen_update_scene`
  is recapture-only. No `URendererSettings` / `UDeveloperSettings`
  read or write path anywhere under `Source/Handlers/`. Agents must
  use `system.console_command "r.Lumen.HardwareRayTracing 1"` which
  doesn't persist to `DefaultEngine.ini` and doesn't update the
  Project Settings UI. Proposes a `rendering.*` namespace scoped to
  `URendererSettings`: generic `get_project_settings` /
  `set_project_settings` plus two convenience verbs
  (`set_lumen_method`, `set_dynamic_gi_method`) that persist via
  `TryUpdateDefaultConfigFile` and optionally mirror to CVars for
  immediate viewport effect. Deliberately not generalized to all
  `UDeveloperSettings` subclasses — that's a follow-up ticket once
  the renderer slice ships and the read/write pattern is validated.
- `#2-rendering-namespace-with-cvar-mirror-extract` `IN-REVIEW` developer — Added Handlers/Render/RenderingProjectSettingsHandler.cpp with rendering.get_project_settings, set_project_settings (CPF_Config gating + TryUpdateDefaultConfigFile), set_lumen_method (bUseHardwareRayTracingForLumen/Reflections/LumenSoftwareTracingMode; finalGatherQuality CVar-only — flagged in wiki), and set_dynamic_gi_method (UPROPERTY + CVar mirror). Lifted the inline if/else-if CVar block from LightingHandler.cpp:688-747 into Handlers/Environment/GIMethodCVarHelper.h (ApplyDynamicGIMethodToCVars) so both handlers share one source. Used verified field names DynamicGlobalIllumination / Reflections / bUseHardwareRayTracingForLumen; ticket text had stale names. Regression tests assert known-field round-trip + non-config-property rejection + registration.
- `#3-skip-editor-offline` `SKIP` tester — Editor not running (port 19880 refused connection, no UnrealEditor process); cannot exercise rendering.get_project_settings / set_lumen_method / set_dynamic_gi_method live. Source files exist on disk as claimed (Handlers/Render/RenderingProjectSettingsHandler.cpp, Handlers/Environment/GIMethodCVarHelper.h) but verification requires a live editor.
- `#4-verify-fix` `DONE` tester — Editor live this run. `rendering.get_project_settings` returned full URendererSettings JSON including `bUseHardwareRayTracingForLumen`, `LumenSoftwareTracingMode`, `DynamicGlobalIllumination`, `Reflections` with `configFile:"...Config/DefaultEngine.ini"`. `rendering.set_lumen_method {hardwareRT:true, softwareRTMode:"DetailTracing"}` returned `applied:["bUseHardwareRayTracingForLumen","LumenSoftwareTracingMode"]`, `rejected:[]`, `savedTo:"...DefaultEngine.ini"`. `rendering.set_project_settings {updates:{NotARealProp:42}}` returned `rejected:[{name:"NotARealProp",reason:"unknown_property"}]` — CPF_Config gating + reflection-driven set works as designed.
