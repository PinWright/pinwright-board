---
id: F-physics-project-settings
title: "No typed read/write for UPhysicsSettings, so registering the project's SurfaceType names needs a hand-edited DefaultEngine.ini plus a property.set that never persists — and until they are registered the whole EPhysicalSurface enum is unreachable from Python"
status: DONE
severity: Medium
category: feature
tags: [physics, physical-material, surface-type, project-settings, developer-settings, persistence, python-enum, escape-hatch]
encounters: 1
costly: 1
lastSeen: 2026-09-02T22:25:00+03:00
---

# `UPhysicsSettings` has no typed surface, and its `PhysicalSurfaces` table gates a whole engine enum

`UPhysicsSettings` is the `UDeveloperSettings` subclass behind **Project Settings -> Engine ->
Physics**, persisted to `Config/DefaultEngine.ini` under `[/Script/Engine.PhysicsSettings]`. Its
`PhysicalSurfaces` array is not an ordinary setting: it is what gives `EPhysicalSurface`'s
`SurfaceType1..62` their names, and until an entry exists for an index, that index is `Unused` /
hidden everywhere — the Blueprint dropdown, and the generated Python enum wrapper.

This is the same shape as `F-rendering-project-settings` (DONE, which produced the `rendering.*`
namespace for `URendererSettings`), one class over.

## What was needed and what it cost

The FPS build's WEAPONS stream owns the project's surface-type table: seven names, referenced by
five other agent streams, needed in the first twenty minutes. There is no verb for it.

**1. Reading the current state was possible, and immediately disclosed a trap.**

    call("property.get", { objectPath: "/Script/Engine.Default__PhysicsSettings",
                           propertyName: "PhysicalSurfaces" })
    -> { "value": [] }

Empty — on a project whose `Config/DefaultEngine.ini` carries a fully populated ten-entry
`[PhysicalMaterial.SurfaceTypes]` section (`SurfaceType1=Wood` ... `SurfaceType10=Button`). That
section is UE4-era config that UE 5.8 does not read, so every one of those names was dead and
nothing in the editor said so. A typed `physics.get_project_settings` returning the live array is
the cheapest possible way to catch that class of stale-config error; `property.get` found it only
because the caller happened to distrust the ini.

**2. Writing it live worked, through the generic escape hatch.**

    call("property.set", { objectPath: "/Script/Engine.Default__PhysicsSettings",
                           propertyName: "PhysicalSurfaces",
                           value: [ {"Type":"SurfaceType1","Name":"Concrete"}, ... ],
                           markDirty: false })
    -> { "applied": true, "markedDirty": false, "existsOnDisk": false }

The write took effect for the session. Note `existsOnDisk: false` and `markedDirty: false` — correct
for a `/Script/` CDO and not a defect, but it means **the response cannot distinguish "applied and
durable" from "applied and lost on restart"**, and for a settings object durability is the whole
point. Nothing in `property.set`'s response or its wiki page says "this will not reach the ini".

**3. Persisting it had to be done by editing the file by hand.** `UObject::SaveConfig()` is not a
`UFUNCTION`, so `python.execute` cannot reach it either; the ini was patched textually with a
stdlib script outside the editor:

    +PhysicalSurfaces=(Type=SurfaceType1,Name="Concrete")
    ... through SurfaceType7

That is a text edit to a config file with no validation, no schema check, and no read-back — for a
table that five downstream streams key their content off.

## The knock-on: the Python enum wrapper is generated once, at startup

Because `PhysicalSurfaces` was empty when the editor booted, `unreal.PhysicalSurface` exposes
exactly one entry:

    [n for n in dir(unreal.PhysicalSurface) if not n.startswith('__')]
    -> ['SURFACE_TYPE_DEFAULT', '_wrapper_meta_data', 'cast', 'get_display_name', 'name',
        'static_enum', 'value']

Registering the names live (step 2 above) does **not** regenerate that wrapper, so for the rest of
the session Python can neither construct nor read a surface type:

    unreal.PhysicalSurface.cast(1)
      -> TypeError: PhysicalSurface: Cannot cast type 'int' to 'PhysicalSurface'

    pm.get_editor_property('surface_type')       # on a PhysicalMaterial whose value is SurfaceType3
      -> TypeError: PhysicalMaterial: Failed to convert property 'SurfaceType' (ByteProperty)
           TypeError: PythonizeEnumEntry: Cannot pythonize '1' (int64) as 'PhysicalSurface'

The read failure is the dangerous one: it raises **mid-script**, after earlier statements in the
same script have already run. In this session the throwing expression sat inside the log line of a
seven-iteration save-and-verify loop, so the loop died on iteration one having saved one asset of
seven, and the response reported `success: false` with a traceback rather than "six assets not
saved". A caller who did not re-read the loop would conclude nothing had been written.

Wrapper generation is engine behaviour (`PyGenUtil`), not a PinWright defect. What *is* in scope is
that PinWright's own `property.get` / `property.set` handle the same value fine through reflection
(`"SurfaceType3"` as a string round-trips), so the plugin already holds the working route and
nothing points a caller at it. `python.md`'s "Calls that crash / freeze the editor" sections are the
right place, and they do not mention that hidden-enum reads throw.

## The ask

    physics.get_project_settings(filter?: string)
      -> { settings: { "<UPROPERTYName>": <jsonValue>, ... }, configFile: "DefaultEngine.ini" }

    physics.set_project_settings(updates: { ... }, save?: bool = true)
      -> { applied: [string], rejected: [{name, reason}], savedTo: string }

    physics.set_surface_types(surfaces: [{ index: int, name: string }], save?: bool = true)
      -> { registered: [{index, name}], previous: [...], savedTo: string, enumRefreshed: bool }

`set_surface_types` earns its own verb rather than riding on the generic setter because it is the
one physics setting with cross-asset consequences: it should reject a duplicate name, reject an
index outside 1..62, refuse to silently drop an index that existing `UPhysicalMaterial` assets
already reference, and say in `enumRefreshed` whether the display-name metadata was rebuilt.
`save: true` must write through `SaveConfig()` (native, reachable from C++) so the caller is not
hand-editing an ini, and the response's `savedTo` is the durability claim `property.set` cannot make.

A companion line on `python.md` — "an `EPhysicalSurface` (or any hidden-entry enum) property raises
`PythonizeEnumEntry` on read; use `property.get` and compare the string name" — costs nothing and
removes the mid-script failure above.

## Dedup

Board-wide search for `physics` project settings, `PhysicalSurfaces`, `SurfaceType`, `SaveConfig`
and `DeveloperSettings`. `F-rendering-project-settings` (DONE, Medium) is the same ticket for
`URendererSettings` and is the precedent this one copies, including the "adjacent surfaces only
touch neighbouring concerns" argument — it is not a duplicate because it shipped `rendering.*`
specifically and explicitly scoped itself to the renderer class. `F-generic-asset-create-verb`
(OPEN, Medium) carries the sibling half of this session's problem (nothing creates the
`UPhysicalMaterial` assets these surface types are assigned to) and has been appended to with that
evidence as `#2`; it stays separate because a generic `asset.create` would not register a single
surface type. `E-rendering-write-verb-persist-key-drift` is about a shipped `rendering.*` verb
writing the wrong ini key, which is downstream of having a verb at all. No ticket covers
`UPhysicsSettings`.

## Severity

**Medium.** Impact class is the rubric's *"soft blocker: doable, but only via a documented
workaround, a source dive, or many extra calls"* — three separate mechanisms (a hand-edited ini for
durability, a `property.set` on a `/Script/` CDO for the live session, and a string-valued
`property.set` per asset because the Python enum is unreachable) stand in for one verb, and all
three worked. It is not High: nothing here returned a false success, and the `property.set` route is
honest about `existsOnDisk: false` even if it does not explain it.

**Reach modifier declined.** Surface types are configured once per project rather than once per
session, which argues down; but every impact-effect, footstep, decal and damage-falloff system in
any shooter-shaped project keys off this table, and getting it wrong is discovered late and fixed
expensively, which argues back up. Medium stands.

## History
- `#1-filed` `OPEN` reporter — `UPhysicsSettings` has no typed verb anywhere in the plugin, so registering the project's seven `EPhysicalSurface` names (the FPS build's first cross-stream deliverable, blocking five other agents) took three separate escape hatches. `call("property.get", { objectPath: "/Script/Engine.Default__PhysicsSettings", propertyName: "PhysicalSurfaces" })` returned `{"value": []}` on a project whose `Config/DefaultEngine.ini` carries a populated ten-entry `[PhysicalMaterial.SurfaceTypes]` section — that section is UE4-era config UE 5.8 does not read, so all ten names were dead and nothing surfaced it; a typed getter would have caught it for free. `call("property.set", ...)` with the seven `{Type, Name}` structs applied live (`applied: true`) but answers `markedDirty: false` / `existsOnDisk: false`, which is correct for a `/Script/` CDO yet means the response **cannot distinguish applied-and-durable from applied-and-lost-on-restart** — the whole point of a settings write — and neither the response nor `property.set.md` says the write does not reach the ini. `UObject::SaveConfig()` is not a `UFUNCTION`, so `python.execute` cannot persist it either; durability came from patching `[/Script/Engine.PhysicsSettings]` textually with a stdlib script outside the editor (seven `+PhysicalSurfaces=(Type=SurfaceTypeN,Name="...")` lines), unvalidated and unread-back. Knock-on, engine-side but undocumented here: the Python enum wrapper is generated at startup from the then-empty table, so `unreal.PhysicalSurface` exposes only `SURFACE_TYPE_DEFAULT` and registering the names live does not regenerate it — `unreal.PhysicalSurface.cast(1)` raises `TypeError: Cannot cast type 'int' to 'PhysicalSurface'`, and reading a `UPhysicalMaterial`'s `surface_type` raises `TypeError: PythonizeEnumEntry: Cannot pythonize '1' (int64) as 'PhysicalSurface'`. That read throws **mid-script**: it sat inside the log line of a seven-iteration save-and-verify loop, which died on iteration one having saved one asset of seven and reported a traceback rather than "six not saved". PinWright's own `property.get`/`property.set` round-trip the same value fine as the string `"SurfaceType3"`, so the plugin holds the working route and nothing points a caller at it. Asked for: `physics.get_project_settings` / `physics.set_project_settings` mirroring the shipped `rendering.*` pair, plus a dedicated `physics.set_surface_types(surfaces, save)` that rejects duplicate names and out-of-range indices, refuses to silently drop an index existing `UPhysicalMaterial` assets reference, persists through native `SaveConfig()` and reports `savedTo` + `enumRefreshed`; and one line on `python.md` warning that a hidden-entry enum property raises on read. Workarounds used, all three, all successful — hence Medium, not High. Dedup: `F-rendering-project-settings` (DONE) is the same ticket one class over and is the precedent copied here; `F-generic-asset-create-verb` (OPEN) holds the sibling half (nothing creates the `UPhysicalMaterial` assets) and was appended to as `#2`; no ticket covers `UPhysicsSettings`.
- `#2-physics-settings-verbs` `IN-REVIEW` developer — No generic project-settings verb exists in the plugin (only the class-specific `rendering.*`), so this ships the three asked-for `physics.*` verbs in a new `Source/PinWright/Private/Handlers/Physics/PhysicsProjectSettingsHandler.cpp` (helpers in named namespace `PwPhysicsSettings`; the rendering handler was not touched — its anonymous-namespace helpers stay a duplicate shape, worth folding into a shared `UObject*` helper if a third settings class lands). `physics.get_project_settings` returns every config UPROPERTY plus `surfaceTypes[{index,name}]` (the live table), `configFile`, `configSection`. `physics.set_project_settings` applies with `PostEditChangeProperty` (where UPhysicsSettings actually takes effect) and rejects `PhysicalSurfaces` with reason `use_physics.set_surface_types`. `physics.set_surface_types(surfaces[{index,name}], replace=false, save=true)` validates index 1..62 / integer / non-empty name / no repeated index (`INVALID_ARGUMENT`), unique names case-insensitively in the resulting table (`DUPLICATE_NAME`), refuses removing an index any `UPhysicalMaterial` asset uses (new `SURFACE_TYPE_IN_USE`, `inUse[{index, physicalMaterials[]}]`, nothing changed), stores the table sorted, runs `LoadSurfaceType` and re-hides removed indices, and reports measured `enumRefreshed`, `registered`, `previous`, `removed`. `surfaces` adopts the nested-key gate (closed `{index,name}`). Persistence (all writers): `TryUpdateDefaultConfigFile` (the Project Settings path, whole-section rewrite), then each written property is cleared and re-imported with `LoadConfig` from the reloaded GConfig; `savedTo` only when all match (`reloadVerified[]` / `reloadMismatch[]`), else `SAVE_FAILED`; `configFile` + `configSection` always reported. Python enum wrapper regeneration is engine-side and not attempted; documented instead. Files: new handler above; new `Source/PinWright/Private/Tests/Gameplay/TestPhysicsProjectSettings.cpp`; `Handlers/ErrorCodes.h` (+`ERR_SURFACE_TYPE_IN_USE`); `Tests/Infra/TestNestedParamKeyGate.cpp` (adoption list +`physics.set_surface_types:surfaces`); `docs/wiki-src/physics.md` (prelude, `## Project settings and surface types`, three `###` method sections); `docs/wiki-src/python.md` (`## Enum values Python cannot represent`); `docs/wiki-src/property.md` (`/Script/` settings CDO writes are not persisted, points at the typed writers); `docs/error-code-catalog.md` (row); `CHANGELOG.md`. Tests (filter `PinWright.physics.get_project_settings+PinWright.physics.set_project_settings+PinWright.physics.set_surface_types`): `PinWright.physics.get_project_settings.ReturnsLiveSurfaceTable`, `PinWright.physics.set_project_settings.AppliesInMemoryAndRefusesSurfaceTable`, `PinWright.physics.set_surface_types.RegistersAndRefreshesEnum`, `...RejectsInvalidEntries`, `...RefusesDroppingIndexInUse` (fixture `/Game/PinWrightTests/PhysicsSettings/PM_PwSurfaceInUse` + unused-index control), `...PersistsAndReloadsFromConfig` (byte-snapshot of DefaultEngine.ini + GConfig reload restore; asserts the name on disk independently), `...ReportsConfigSaveFailure` (read-only ini). Every test restores the CDO table and EPhysicalSurface metadata; the two config-writing tests restore DefaultEngine.ini byte-for-byte. Compile-checked with fastcheck; not yet built or run.
- `#3-review-fixes` `IN-REVIEW` developer — Review follow-ups. `UDeveloperSettings::PostEditChangeProperty` does not chain to UObject's `OnObjectPropertyChanged` broadcast (`DeveloperSettings.cpp:89-92`), so both writers now broadcast it explicitly after `PostEditChangeProperty`; this makes the wiki's "open Project Settings tabs refresh" claim true. Removed the re-hide loop after `LoadSurfaceType`, which did nothing: that function already hides every entry that has no Hidden metadata before it un-hides the table's entries. `saveDetail` now says "Nothing was applied, so nothing was written." when `save:true` rejected every update. Test ini guard: the snapshot records `bSnapshotOk`, and the destructor never writes when the read failed (the two config tests assert it before touching the file). When the bytes on disk still match, it only restores the read-only flag. Otherwise it writes the bytes back, then `GConfig->Flush(false, GEngineIni)`, then the forced reload, in the same order as `UpdateSingleSectionOfConfigFile`. `RefusesDroppingIndexInUse` now picks the two highest indices no physical material asset uses rather than a fixed 61/62. fastcheck OK on both files; check_test_ids and check_test_skips report clean.
- `#4-run1-upsert-keeps-entries` `IN-REVIEW` developer — run1 failed `RegistersAndRefreshesEnum` ("upsert kept the existing entries" expected 18, was 9). Real bug, not a test assumption: the upsert rebuilt the table as a `TMap<int32, FName>` keyed by index, so a host ini that repeats an index (PDS `Config/DefaultEngine.ini` has 16 `+PhysicalSurfaces` lines over indices 1..7) was collapsed to 7 entries, and with `save:true` that would have rewritten the host ini dropping 9 lines. `RefusesDroppingIndexInUse` only passed because its setup upsert had already collapsed the table. Fix in `PhysicsProjectSettingsHandler.cpp`: upsert keeps every entry of an unlisted index verbatim; a listed index becomes exactly the one requested entry (at its first old entry's position, else appended in index order); `DUPLICATE_NAME` now checks only the requested names against every other index's effective (last-wins, as `LoadSurfaceType`) name, so a duplicate the host table already carries is not the caller's to fix; table no longer re-sorted. Test: `RegistersAndRefreshesEnum` seeds a repeated-index pair (index 60) on every host and asserts both entries survive with the last-wins name; expected count is `pre - countAt(A) - countAt(B) + 2`. Helpers `CurrentEntries` (replace payloads) and `NameAt` now use the effective last-wins table. Wiki `physics.md` upsert/duplicate/`previous` wording updated. fastcheck OK on both files; check_test_ids / check_test_skips clean.
- `#5-review-fixes` `IN-REVIEW` developer — Review of #4. `RegistersAndRefreshesEnum` now also seeds the listed index A twice, interleaved with the unlisted repeated index (`{61,PwOldA1},{60,PwDupFirst},{61,PwOldA2},{60,PwDupLast}`), and asserts A collapses to one entry at its first old position and that entries of unlisted indices keep their sequence (catches keep-and-append and re-sort mutations). Host-project wording removed from the handler and test comments. `physics.md`: a repeated-index `previous` is restored from source control. fastcheck OK; check_test_ids / check_test_skips clean.
- `#6-verified-linux` `DONE` tester — Fix commit 6baf62f6. All seven tests passed non-skipped in run3/full: `PinWright.physics.get_project_settings.ReturnsLiveSurfaceTable`, `PinWright.physics.set_project_settings.AppliesInMemoryAndRefusesSurfaceTable`, and `PinWright.physics.set_surface_types.{RegistersAndRefreshesEnum, RejectsInvalidEntries, RefusesDroppingIndexInUse, PersistsAndReloadsFromConfig, ReportsConfigSaveFailure}`. The NestedParamKeyGate suite also passed (adoption of `physics.set_surface_types:surfaces`). Acceptance: the get/set pair mirrors rendering.*; set_surface_types rejects out-of-range indices and duplicate names, refuses dropping an index a UPhysicalMaterial uses (`SURFACE_TYPE_IN_USE`), and reports `enumRefreshed`; `savedTo` is set only after a config write and reload read-back, else `SAVE_FAILED`; the repeated-index upsert bug found in run1 is pinned. Docs verified at 7230b41d: `python.md` `## Enum values Python cannot represent` and the `property.md` /Script/ settings-CDO non-persistence note. Limit: Python enum wrapper regeneration is engine-side, documented and not attempted.
