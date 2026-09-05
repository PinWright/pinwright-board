---
id: B-rendering-save-false-success
title: "Rendering project-setting verbs ignore the config writer's failure result and report a destination as if persistence succeeded"
status: IN-REVIEW
severity: High
category: bug
tags: [rendering, project-settings, config, persistence, false-success]
encounters: 1
lastSeen: 2026-09-03T23:27:21+03:00
---

# Rendering settings report persistence without observing the config write

## What happens

The shared `ApplyUpdates` helper obtains the default config filename, calls
`URendererSettings::TryUpdateDefaultConfigFile`, discards its boolean result, and
returns the filename in `savedTo` (`RenderingProjectSettingsHandler.cpp:124-135`).
UE 5.8 documents that return value as false when the file could not be updated
(`C:\UE_5.8\Engine\Source\Runtime\CoreUObject\Public\UObject\Object.h:1299-1306`).

`rendering.set_project_settings` and its convenience setters use this helper.
`rendering.set_dynamic_gi_method` goes further and publishes `configFile` from the
expected filename regardless of the discarded result
(`RenderingProjectSettingsHandler.cpp:283-326`). A read-only or unwritable
`DefaultEngine.ini` therefore yields a green mutation receipt that will vanish on
restart.

## Why it matters

Callers use these verbs specifically for project-level persistence. A path string
is not evidence that the config write succeeded. Severity is High for a successful
mutation that silently fails to persist.

## What should happen

Capture the writer's boolean result, return an error or an explicit `saved:false`
state with diagnostics, and expose `savedTo` only after success. Read the target
section back from disk, or reuse the complete save-report vocabulary used by other
PinWright persistence paths.

## Workaround

After every persistent rendering-settings call, inspect `DefaultEngine.ini` on disk
for the intended section/value before restarting or building.

## Related

- `E-rendering-write-verb-persist-key-drift` — response-key inconsistency; it does
  not cover the ignored config-write result.
- `F-rendering-project-settings` — feature ticket.

## Fix

The defect was TRUE: `ApplyUpdates` discarded `TryUpdateDefaultConfigFile`'s boolean result and populated a destination before persistence was known. The shared write path now combines the writer result with an on-disk file-presence probe, reports `saveRequested` / `saved` / `saveState` / `saveDetail`, exposes `savedTo` only after a measured success, and returns typed `SAVE_FAILED` error data when the write is refused; the dynamic-GI wrapper only publishes `configFile` after that same success.

Files changed: `Source/PinWright/Private/Handlers/Render/RenderingProjectSettingsHandler.cpp`, `Source/PinWright/Private/Tests/EditorOps/TestRenderingProjectSettingsSaveContract.cpp`, and `docs/wiki-src/rendering.md`. Test id: `PinWright.rendering.set_project_settings.ReportsConfigSaveFailure` (handler-level read-only-config refusal with the original attribute and CDO value restored). Deliberately not changed: the live CVar mutation order, property rejection behavior, or the engine config writer; no build, editor, MCP, or automation run was performed under the worker brief.

## History
- `#1-source-scan-config-result` `OPEN` reporter — Source-only scan confirmed the UE boolean save result is discarded and the expected path is returned by all helper consumers. No file permissions were changed and no build, test, editor, MCP call, or plugin edit was performed.
- `#2-measure-config-write` `IN-REVIEW` developer — Captured the config writer result plus file presence, made failed persistence a typed `SAVE_FAILED` with measured response data, withheld destination fields until success, documented the contract, and added a handler-level read-only-config regression test. Source/static checks only; compile and automation remain for the checkpoint agent.
