---
id: B-blueprint-get-omits-parent-class
title: "blueprint.get's description and wiki promise the parent class, but the response has no parent-class field"
status: DONE
severity: Medium
category: bug
tags: [blueprint, blueprint-get, readback, parent-class, docs-mismatch]
encounters: 1
lastSeen: 2026-09-23T18:30:00Z
---

# blueprint.get never returns the parent class it advertises

`blueprint.get {path: "/App/App/UI/LobbyAndMenu/Popups/W_AppSchoolNameLogin"}` returns only `blueprintPath` / `resolvedPath` / `assetPath` / `variables` / `functions` / `events` / `defaults` / `metadata`. The registered description (`BlueprintInfoHandler.cpp:55`, "Return summary metadata for a Blueprint: parent class, variables, ...") and the generated page (`Saved/PinWright/wiki/blueprint.get.md`, description and Notes "summary of a Blueprint class: parent class, ...") both promise a parent class.

**Source (8748c637):** the response is `BuildBlueprintSnapshot` (`BlueprintHandlerUtils.cpp:1349-1370`), which never sets a `parentClass` field; the handler then merges only `defaults` / `metadata` / `functions` / `events` from the registry (`BlueprintInfoHandler.cpp:89-144`) before `SendSuccess` at `:146`. `blueprint.inspect` does emit it (`BlueprintInspectHandler.cpp:64`, `BP->ParentClass->GetName()`), so the field exists on the heavier verb only.

**Workaround:** `blueprint.inspect` (reads the whole structural dump), or Python `EditorAssetLibrary.find_asset_data(path).get_tag_value("ParentClass")`.

**Fix:** add `parentClass` (full class path, e.g. `BP->ParentClass->GetPathName()`) to `BuildBlueprintSnapshot`, or drop "parent class" from the description and wiki.

**Related:** `E-blueprint-get-omits-components-readback-guidance` (IN-REVIEW), the same advertised-but-never-emitted shape for `components`; `E-blueprint-get-defaults-always-empty` (IN-REVIEW) for `defaults`.

## History
- `#1-parent-class-missing` `OPEN` reporter - UE 5.8, host `X:\src\unreal\unreal-fpv-new`, plugin `8748c637`, school-computer login work on W_AppSchoolNameLogin. Parent class had to be read through Python asset-registry tags. Source-verified before filing.
- `#2-parent-class-emitted` `IN-REVIEW` developer - Still reproducible at plugin `10212ee4`: `BuildBlueprintSnapshot` set no parent field. `BuildBlueprintSnapshot` (`Handlers/Blueprint/BlueprintHandlerUtils.cpp`) now emits `parentClass` = `Blueprint->ParentClass->GetPathName()` (JSON `null` when the parent is missing), so `blueprint.get` and every other verb returning the snapshot carry it. Registered description and `docs/wiki-src/blueprint.md` (`### blueprint.get`) name the field and its format; CHANGELOG entry. Test `PinWright.blueprint.get.ParentClassReadback` (`Tests/Blueprint/TestBlueprintGetInputEvents.cpp`) drives the dispatcher on an `APawn`-parented BP and asserts `parentClass == /Script/Engine.Pawn`; fails with the field removed.
- `#3-review-fixes` `IN-REVIEW` developer - Review NIT: `blueprint.get` returns `parentClass` as a full path while `blueprint.inspect` returns the short name under the same key; documented in `docs/wiki-src/blueprint.md` (`### blueprint.get`) rather than changing either verb's existing output.
- `#4-verified-linux` `DONE` tester — Passed non-skipped in run3/full: `PinWright.blueprint.get.ParentClassReadback` (dispatcher call on an APawn-parented BP returns `parentClass == /Script/Engine.Pawn`). Acceptance (add `parentClass` to `BuildBlueprintSnapshot` so the response matches the description and wiki): the field is emitted and asserted by that test. Doc half verified at PinWright 7230b41d: the registered `blueprint.get` description in `BlueprintInfoHandler.cpp` names `parentClass (full class path, null when the parent is missing)`, and `docs/wiki-src/blueprint.md` documents it and the short-name difference from `blueprint.inspect`. Coverage limit: the null-parent branch is not exercised by a test.
