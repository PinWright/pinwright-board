---
id: B-blueprint-get-omits-parent-class
title: "blueprint.get's description and wiki promise the parent class, but the response has no parent-class field"
status: OPEN
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
